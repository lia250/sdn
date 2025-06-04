# 1. pox

pox/pox/forwrading/adaptive_routing.py

```python
from pox.core import core
import pox.openflow.libopenflow_01 as of
from pox.lib.util import dpidToStr
from pox.lib.addresses import IPAddr, EthAddr
import time
import networkx as nx

log = core.getLogger()

class AdaptiveRouting(object):
    def __init__(self):
        core.openflow.addListeners(self)
        self.flow_stats = {}       # {dpid: {flow_key: (byte_count, timestamp)}}
        self.link_delays = {}      # {(src_dpid, dst_dpid): delay}
        self.topology = {}         # {dpid: {neighbor_dpid: port}}
        self.hosts = {}            # {host_ip: (attached_switch_dpid, host_mac)}
        self.mac_to_port = {}      # {dpid: {mac: port}}
        self.graph = nx.Graph()    # NetworkX graph for path calculation
        self.link_discovery_started = False

    def _handle_ConnectionUp(self, event):
        log.info("Switch %s connected", dpidToStr(event.dpid))
        self.topology[event.dpid] = {}
        self.mac_to_port[event.dpid] = {}
        self.graph.add_node(event.dpid)
        
        # Initialize link discovery
        if not self.link_discovery_started:
            self._start_link_discovery()
            self.link_discovery_started = True

        # Request flow stats periodically
        self._request_stats(event.connection)

    def _start_link_discovery(self):
        """Periodically send LLDP packets for topology discovery"""
        log.info("Starting LLDP-based link discovery")
        for connection in core.openflow._connections.values():
            self._send_lldp_packet(connection)
        core.callDelayed(5, self._start_link_discovery)  # Repeat every 5 seconds

    def _send_lldp_packet(self, connection):
        """Send properly formatted LLDP packet for link discovery"""
        # Ethernet header
        eth_dst = b'\x01\x80\xc2\x00\x00\x0e'  # LLDP multicast MAC
        eth_src = EthAddr("%012x" % connection.dpid).toRaw()  # Switch DPID as MAC
        eth_type = b'\x88\xcc'  # LLDP ethertype
        
        # LLDP payload
        # 1. Chassis ID TLV (mandatory)
        chassis_id = b'\x02'  # TLV type 2 (Chassis ID)
        chassis_length = b'\x07'  # Length 7 (for 8-byte DPID)
        chassis_value = connection.dpid.to_bytes(8, 'big')  # DPID as value
        chassis_tlv = chassis_id + chassis_length + chassis_value
        
        # 2. Port ID TLV (mandatory)
        port_id = b'\x04'  # TLV type 4 (Port ID)
        port_length = b'\x07'  # Length 7 (for 8-byte DPID)
        port_value = connection.dpid.to_bytes(8, 'big')  # DPID as value
        port_tlv = port_id + port_length + port_value
        
        # 3. TTL TLV (mandatory)
        ttl_id = b'\x06'  # TLV type 6 (TTL)
        ttl_length = b'\x02'  # Length 2
        ttl_value = b'\x00\x78'  # TTL = 120 seconds
        ttl_tlv = ttl_id + ttl_length + ttl_value
        
        # 4. End TLV (mandatory)
        end_tlv = b'\x00\x00'  # End of LLDPDU
        
        # Combine all parts
        lldp_payload = chassis_tlv + port_tlv + ttl_tlv + end_tlv
        
        # Create packet_out
        msg = of.ofp_packet_out()
        msg.data = eth_dst + eth_src + eth_type + lldp_payload
        msg.actions.append(of.ofp_action_output(port=of.OFPP_ALL))
        connection.send(msg)

    def _handle_LLDP(self, event):
        """Handle incoming LLDP packets for topology discovery"""
        packet = event.parsed
        src_dpid = event.connection.dpid
        src_port = event.port
        
        try:
            # Parse LLDP payload (basic parsing)
            data = packet.payload
            if len(data) < 8:
                return
                
            # Extract neighbor DPID from Chassis ID TLV
            # This assumes the Chassis ID is the DPID (as we sent it)
            dst_dpid = int.from_bytes(data[7:15], 'big')
            
            # Update topology
            if src_dpid not in self.topology:
                self.topology[src_dpid] = {}
            self.topology[src_dpid][dst_dpid] = src_port
            
            # Update network graph with default weight
            self.graph.add_edge(src_dpid, dst_dpid, weight=1)
            log.debug("Discovered link: %s (port %d) -> %s", 
                     dpidToStr(src_dpid), src_port, dpidToStr(dst_dpid))
        except Exception as e:
            log.warning("Error processing LLDP packet: %s", e)

    def _request_stats(self, connection):
        """Periodically request flow statistics from switches"""
        r = of.ofp_stats_request(body=of.ofp_flow_stats_request())
        connection.send(r)
        core.callDelayed(5, self._request_stats, connection)

    def _handle_FlowStatsReceived(self, event):
        """Handle received flow statistics"""
        dpid = event.connection.dpid
        self.flow_stats.setdefault(dpid, {})
        
        for stat in event.stats:
            key = (stat.match.nw_src, stat.match.nw_dst)
            self.flow_stats[dpid][key] = (stat.byte_count, time.time())

    def _handle_PacketIn(self, event):
        packet = event.parsed
        if not packet.parsed:
            log.warning("Ignoring incomplete packet")
            return

        dpid = event.dpid
        in_port = event.port

        # Handle LLDP packets first
        if packet.type == packet.LLDP_TYPE:
            self._handle_LLDP(event)
            return

        # Learn host location
        if packet.src not in self.mac_to_port[dpid]:
            self.mac_to_port[dpid][packet.src] = in_port
            if packet.type == packet.IP_TYPE:
                ip_packet = packet.payload
                self.hosts[ip_packet.srcip] = (dpid, packet.src)
                log.info("Learned host %s at switch %s", ip_packet.srcip, dpidToStr(dpid))

        # Handle ARP packets
        if packet.type == packet.ARP_TYPE:
            self._handle_ARP(event)
            return

        # Handle IP packets
        if packet.type == packet.IP_TYPE:
            ip_packet = packet.payload
            src_ip = ip_packet.srcip
            dst_ip = ip_packet.dstip

            # Flood if destination unknown
            if dst_ip not in self.hosts:
                self._flood_packet(event)
                return

            dst_dpid, dst_mac = self.hosts[dst_ip]
            if dpid == dst_dpid:
                # Destination is on same switch
                out_port = self.mac_to_port[dpid].get(dst_mac)
                if out_port:
                    self._send_packet(dpid, out_port, packet)
                else:
                    self._flood_packet(event)
            else:
                # Find path to destination
                path = self._calculate_best_path(dpid, dst_dpid)
                if path:
                    log.debug("Installing path from %s to %s: %s", 
                             dpidToStr(dpid), dpidToStr(dst_dpid), 
                             " -> ".join(dpidToStr(p) for p in path))
                    self._install_path_flows(path, packet.src, dst_mac, src_ip, dst_ip)
                else:
                    log.warning("No path found from %s to %s", 
                               dpidToStr(dpid), dpidToStr(dst_dpid))
                    self._flood_packet(event)
        else:
            self._flood_packet(event)

    def _handle_ARP(self, event):
        packet = event.parsed
        arp = packet.payload
        dpid = event.dpid

        if arp.opcode == arp.REQUEST:
            log.debug("ARP request %s -> %s", arp.protosrc, arp.protodst)
            
            if arp.protodst in self.hosts:
                dst_dpid, dst_mac = self.hosts[arp.protodst]
                r = arp.make_reply(hwsrc=dst_mac,
                                  hwdst=packet.src,
                                  protosrc=arp.protodst,
                                  protodst=arp.protosrc)
                self._send_packet(dpid, event.port, r)
                log.debug("Sent ARP reply %s -> %s", arp.protodst, arp.protosrc)
            else:
                self._flood_packet(event)

    def _calculate_best_path(self, src_dpid, dst_dpid):
        """Calculate best path based on current link delays"""
        try:
            return nx.shortest_path(self.graph, source=src_dpid, target=dst_dpid, 
                                  weight='weight')
        except nx.NetworkXNoPath:
            return None

    def _install_path_flows(self, path, src_mac, dst_mac, src_ip, dst_ip):
        """Install flow entries along the calculated path"""
        for i in range(len(path)):
            current = path[i]
            
            if i < len(path)-1:
                next_hop = path[i+1]
                out_port = self.topology[current][next_hop]
            else:
                out_port = self.mac_to_port[current].get(dst_mac)
                if not out_port:
                    continue

            # Install forward flow
            fm = of.ofp_flow_mod()
            fm.match = of.ofp_match(dl_type=0x800, nw_src=src_ip, nw_dst=dst_ip)
            fm.actions.append(of.ofp_action_dl_addr.set_dst(dst_mac))
            fm.actions.append(of.ofp_action_output(port=out_port))
            core.openflow.sendToDPID(current, fm)

            # Install reverse flow
            fm = of.ofp_flow_mod()
            fm.match = of.ofp_match(dl_type=0x800, nw_src=dst_ip, nw_dst=src_ip)
            fm.actions.append(of.ofp_action_dl_addr.set_dst(src_mac))
            if i > 0:
                prev_hop = path[i-1]
                fm.actions.append(of.ofp_action_output(port=self.topology[current][prev_hop]))
            else:
                fm.actions.append(of.ofp_action_output(port=self.mac_to_port[current][src_mac]))
            core.openflow.sendToDPID(current, fm)

    def _flood_packet(self, event):
        """Flood packet to all ports except input port"""
        msg = of.ofp_packet_out()
        msg.data = event.ofp
        msg.actions.append(of.ofp_action_output(port=of.OFPP_ALL))
        msg.in_port = event.port
        event.connection.send(msg)

    def _send_packet(self, dpid, port, packet):
        """Send packet out of specific port"""
        msg = of.ofp_packet_out()
        msg.data = packet.pack()
        msg.actions.append(of.ofp_action_output(port=port))
        core.openflow.sendToDPID(dpid, msg)

def launch():
    """Start the Adaptive Routing component"""
    core.registerNew(AdaptiveRouting)
```

# 2. mininet

adaptive_routing_topology.py

```python
from mininet.topo import Topo

class adaptive(Topo):
    def __init__(self):
        Topo.__init__(self)
        
        # Add switches
        s1 = self.addSwitch('s1')
        s2 = self.addSwitch('s2')
        s3 = self.addSwitch('s3')
        
        # Add hosts with IP addresses
        h1 = self.addHost('h1', ip='10.0.0.1/24', mac='00:00:00:00:00:01')
        h2 = self.addHost('h2', ip='10.0.0.2/24', mac='00:00:00:00:00:02')
        h3 = self.addHost('h3', ip='10.0.0.3/24', mac='00:00:00:00:00:03')
        
        # Add links with explicit ports
        self.addLink(h1, s1, port1=1, port2=1)
        self.addLink(h2, s2, port1=1, port2=1)
        self.addLink(h3, s3, port1=1, port2=1)
        
        # Switch-to-switch links
        self.addLink(s1, s2, port1=2, port2=2)
        self.addLink(s2, s3, port1=3, port2=2)
        self.addLink(s3, s1, port1=3, port2=3)

topos = {'adaptive': (lambda: adaptive())}
```

# 3. run controller pox

```
./pox.py log.level --DEBUG  forwarding.adaptive_routing
```

# 4. run mininet 

```
sudo mn --custom adaptive_routing_topology.py --controller=remote,ip=127.0.0.1 --topo=adaptive
```
