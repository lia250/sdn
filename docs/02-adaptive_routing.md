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
        self.mac_to_port = {}      # {(dpid, mac): port}
        self.graph = nx.Graph()    # NetworkX graph for path calculation

    def _handle_ConnectionUp(self, event):
        log.info("Switch %s connected", dpidToStr(event.dpid))
        self.topology[event.dpid] = {}
        self.graph.add_node(event.dpid)

    def _handle_PacketIn(self, event):
        packet = event.parsed
        if not packet.parsed:
            log.warning("Ignoring incomplete packet")
            return

        dpid = event.dpid
        in_port = event.port

        # Learn host location
        if packet.src not in self.mac_to_port.get((dpid, packet.src), (None,))[0]:
            self.mac_to_port[(dpid, packet.src)] = in_port
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

            # Learn topology (simple link discovery)
            if packet.src.isBridge() and packet.dst.isBridge():
                if (dpid, event.connection.otherDPID) not in self.link_delays:
                    self.link_delays[(dpid, event.connection.otherDPID)] = 10  # default 10ms
                    self.topology[dpid][event.connection.otherDPID] = in_port
                    self.graph.add_edge(dpid, event.connection.otherDPID, 
                                      weight=self.link_delays[(dpid, event.connection.otherDPID)])
                    log.info("Discovered link %s <-> %s", 
                            dpidToStr(dpid), dpidToStr(event.connection.otherDPID))

            # Adaptive routing for known hosts
            if dst_ip in self.hosts:
                dst_dpid, dst_mac = self.hosts[dst_ip]
                path = self._calculate_best_path(dpid, dst_dpid)
                
                if path:
                    log.debug("Installing path from %s to %s: %s", 
                             dpidToStr(dpid), dpidToStr(dst_dpid), 
                             " -> ".join(dpidToStr(p) for p in path))
                    self._install_path_flows(path, src_ip, dst_ip, dst_mac)
                else:
                    log.warning("No path found from %s to %s", 
                               dpidToStr(dpid), dpidToStr(dst_dpid))
                    self._flood_packet(event)
            else:
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

    def _calculate_best_path(self, src_dpid, dst_dpid):
        try:
            return nx.shortest_path(self.graph, source=src_dpid, target=dst_dpid, 
                                  weight='weight')
        except nx.NetworkXNoPath:
            return None

    def _install_path_flows(self, path, src_ip, dst_ip, dst_mac):
        for i in range(len(path)-1):
            current = path[i]
            next_hop = path[i+1]
            out_port = self.topology[current].get(next_hop)

            if out_port is None:
                continue

            # Forward flow
            fm = of.ofp_flow_mod()
            fm.match = of.ofp_match(dl_type=0x800, 
                                  nw_src=src_ip, 
                                  nw_dst=dst_ip)
            fm.actions.append(of.ofp_action_dl_addr.set_dst(dst_mac))
            fm.actions.append(of.ofp_action_output(port=out_port))
            core.openflow.sendToDPID(current, fm)

            # Reverse flow
            fm = of.ofp_flow_mod()
            fm.match = of.ofp_match(dl_type=0x800, 
                                  nw_src=dst_ip, 
                                  nw_dst=src_ip)
            fm.actions.append(of.ofp_action_output(port=self.mac_to_port[(current, dst_mac)]))
            core.openflow.sendToDPID(next_hop, fm)

    def _flood_packet(self, event):
        msg = of.ofp_packet_out()
        msg.data = event.ofp
        msg.actions.append(of.ofp_action_output(port=of.OFPP_ALL))
        msg.in_port = event.port
        event.connection.send(msg)

    def _send_packet(self, dpid, port, packet):
        msg = of.ofp_packet_out()
        msg.data = packet.pack()
        msg.actions.append(of.ofp_action_output(port=port))
        core.openflow.sendToDPID(dpid, msg)

def launch():
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
python3 pox.py openflow.of_01 --port=6634 forwarding.adaptive_routing
```

# 4. run mininet 

```
sudo mn --custom adaptive_routing_topology.py --controller=remote,ip=127.0.0.1,port=6634 --topo=adaptive
```
