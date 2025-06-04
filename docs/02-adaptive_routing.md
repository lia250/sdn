```python
from pox.core import core
import pox.openflow.libopenflow_01 as of
from pox.lib.util import dpidToStr
from pox.lib.addresses import IPAddr
import time

log = core.getLogger()

class AdaptiveRouting(object):
    def __init__(self):
        core.openflow.addListeners(self)
        self.flow_stats = {}  # {dpid: {flow_key: (byte_count, timestamp)}}
        self.link_delays = {} # {(src_dpid, dst_dpid): delay}
        self.topology = {}    # {dpid: {neighbor_dpid: port}}

    def _handle_ConnectionUp(self, event):
        log.info("Switch %s has come up.", dpidToStr(event.dpid))

    def _handle_PacketIn(self, event):
        packet = event.parsed
        if not packet.parsed:
            log.warning("Ignoring incomplete packet")
            return

        dpid = event.dpid
        inport = event.port
        src = packet.src
        dst = packet.dst

        if packet.type == packet.IP_TYPE:
            ip_packet = packet.payload
            src_ip = ip_packet.srcip
            dst_ip = ip_packet.dstip

            # Learn topology (simplified)
            if packet.src not in self.topology.get(dpid, {}):
                if dpid not in self.topology:
                    self.topology[dpid] = {}
                self.topology[dpid][packet.src] = inport

            # Adaptive routing logic
            if dst_ip in self.hosts:
                best_path = self.calculate_best_path(dpid, dst_ip)
                if best_path:
                    self.install_path_flows(event, best_path, src_ip, dst_ip)
            else:
                # Flood for ARP or other non-IP packets
                self.flood_packet(event)

    def calculate_best_path(self, src_dpid, dst_ip):
        """
        Calculate the best path based on current delay metrics
        """
        dst_dpid = self.hosts[dst_ip]

        # Dijkstra's algorithm for shortest path based on delay
        paths = self.dijkstra(src_dpid)
        if dst_dpid in paths:
            return paths[dpid]
        return None

    def dijkstra(self, start_dpid):
        """
        Dijkstra's algorithm to find shortest paths based on delay
        """
        distances = {node: float('inf') for node in self.topology}
        distances[start_dpid] = 0
        previous = {node: None for node in self.topology}
        unvisited = set(self.topology.keys())

        while unvisited:
            current = min(unvisited, key=lambda node: distances[node])
            unvisited.remove(current)

            for neighbor, port in self.topology[current].items():
                if neighbor not in self.topology:
                    continue
                link_delay = self.link_delays.get((current, neighbor), 10)  # default 10ms
                alternative = distances[current] + link_delay
                if alternative < distances[neighbor]:
                    distances[neighbor] = alternative
                    previous[neighbor] = current

        paths = {}
        for node in self.topology:
            if node == start_dpid:
                continue
            path = []
            current = node
            while current is not None:
                path.insert(0, current)
                current = previous[current]
            if len(path) > 1:
                paths[node] = path

        return paths

    def install_path_flows(self, event, path, src_ip, dst_ip):
        """
        Install flows along the calculated path
        """
        for i in range(len(path)-1):
            current = path[i]
            next_hop = path[i+1]

            # Find the port to the next hop
            out_port = self.topology[current].get(next_hop, None)
            if out_port is None:
                continue

            # Create flow mod
            msg = of.ofp_flow_mod()
            msg.match = of.ofp_match(dl_type=0x800, nw_src=src_ip, nw_dst=dst_ip)
            msg.actions.append(of.ofp_action_output(port=out_port))

            # Send to switch
            core.openflow.sendToDPID(current, msg)

        # Also install reverse path
        for i in range(len(path)-1, 0, -1):
            current = path[i]
            prev_hop = path[i-1]

            out_port = self.topology[current].get(prev_hop, None)
            if out_port is None:
                continue

            msg = of.ofp_flow_mod()
            msg.match = of.ofp_match(dl_type=0x800, nw_src=dst_ip, nw_dst=src_ip)
            msg.actions.append(of.ofp_action_output(port=out_port))

            core.openflow.sendToDPID(current, msg)

    def flood_packet(self, event):
        """
        Flood the packet to all ports except the input port
        """
        msg = of.ofp_packet_out()
        msg.data = event.ofp
        msg.actions.append(of.ofp_action_output(port=of.OFPP_ALL))
        msg.in_port = event.port
        event.connection.send(msg)

    def _handle_FlowStatsReceived(self, event):
        """
        Handle flow statistics to estimate link delays
        """
        for stat in event.stats:
            key = (stat.match.nw_src, stat.match.nw_dst, stat.match.tp_src, stat.match.tp_dst)
            current_time = time.time()

            if key in self.flow_stats.get(event.dpid, {}):
                prev_bytes, prev_time = self.flow_stats[event.dpid][key]
                byte_diff = stat.byte_count - prev_bytes
                time_diff = current_time - prev_time

                if time_diff > 0:
                    # Simple delay estimation (can be enhanced)
                    estimated_delay = byte_diff / (time_diff * 1000)  # Simplified metric
                    # Update link delay (this is a simplified approach)
                    # In a real implementation, you would map flows to links
                    pass

            self.flow_stats.setdefault(event.dpid, {})[key] = (stat.byte_count, current_time)

def launch():
    core.registerNew(AdaptiveRouting)
```
