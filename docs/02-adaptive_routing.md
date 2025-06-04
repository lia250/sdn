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
        self.flow_stats = {}
        self.link_delays = {}
        self.topology = {}
        self.hosts = {}
        self.mac_to_port = {}
        self.graph = nx.Graph()
        self.link_discovery_started = False

    def _handle_ConnectionUp(self, event):
        log.info("Switch %s connected", dpidToStr(event.dpid))
        self.topology[event.dpid] = {}
        self.mac_to_port[event.dpid] = {}
        self.graph.add_node(event.dpid)
        
        if not self.link_discovery_started:
            self._start_link_discovery()
            self.link_discovery_started = True

        self._request_stats(event.connection)

    def _start_link_discovery(self):
        log.info("Starting LLDP-based link discovery")
        for connection in core.openflow._connections.values():
            self._send_lldp_packet(connection)
        core.callDelayed(5, self._start_link_discovery)

    def _send_lldp_packet(self, connection):
        """Send raw LLDP packet without using POX's LLDP class"""
        # Build LLDP payload
        lldp_payload = (
            b'\x02\x07' + connection.dpid.to_bytes(8, 'big')[:7] +  # Chassis ID
            b'\x04\x07' + connection.dpid.to_bytes(8, 'big')[:7] +  # Port ID
            b'\x06\x02\x00\x78' +  # TTL (120 seconds)
            b'\x00\x00'  # End
        )
        
        # Build Ethernet frame
        eth_frame = (
            b'\x01\x80\xc2\x00\x00\x0e' +  # Destination MAC (LLDP)
            EthAddr("%012x" % connection.dpid).toRaw() +  # Source MAC
            b'\x88\xcc' +  # EtherType (LLDP)
            lldp_payload
        )
        
        # Send packet
        msg = of.ofp_packet_out()
        msg.data = eth_frame
        msg.actions.append(of.ofp_action_output(port=of.OFPP_ALL))
        connection.send(msg)

    def _handle_LLDP(self, event):
        """Handle raw LLDP packet by parsing it manually"""
        packet = event.ofp
        try:
            # Skip Ethernet header (14 bytes) and check for LLDP ethertype
            if len(packet.data) < 16 or packet.data[12:14] != b'\x88\xcc':
                return
                
            # Parse LLDP TLVs
            data = packet.data[14:]
            pos = 0
            
            # Parse Chassis ID (first TLV)
            if len(data) < pos+2 or data[pos] != 0x02:
                return
            chassis_len = data[pos+1]
            if len(data) < pos+2+chassis_len:
                return
            dst_dpid = int.from_bytes(data[pos+2:pos+2+chassis_len], 'big')
            pos += 2 + chassis_len
            
            # Get source info
            src_dpid = event.connection.dpid
            src_port = event.port
            
            # Update topology
            if src_dpid not in self.topology:
                self.topology[src_dpid] = {}
            self.topology[src_dpid][dst_dpid] = src_port
            self.graph.add_edge(src_dpid, dst_dpid, weight=1)
            
            log.debug("Discovered link: %s (port %d) -> %s",
                     dpidToStr(src_dpid), src_port, dpidToStr(dst_dpid))
        except Exception as e:
            log.warning("LLDP parsing error: %s", str(e))

    def _handle_PacketIn(self, event):
        packet = event.parsed
        
        # Check for raw LLDP packet (by ethertype)
        if packet.type == 0x88cc:
            self._handle_LLDP(event)
            return
            
        if not packet.parsed:
            log.warning("Ignoring incomplete packet")
            return

        dpid = event.dpid
        in_port = event.port

        # Host learning
        if packet.src not in self.mac_to_port[dpid]:
            self.mac_to_port[dpid][packet.src] = in_port
            if packet.type == packet.IP_TYPE:
                ip_packet = packet.payload
                self.hosts[ip_packet.srcip] = (dpid, packet.src)
                log.info("Learned host %s at switch %s", ip_packet.srcip, dpidToStr(dpid))

        # ARP handling
        if packet.type == packet.ARP_TYPE:
            self._handle_ARP(event)
            return

        # IP handling
        if packet.type == packet.IP_TYPE:
            ip_packet = packet.payload
            src_ip = ip_packet.srcip
            dst_ip = ip_packet.dstip

            if dst_ip not in self.hosts:
                self._flood_packet(event)
                return

            dst_dpid, dst_mac = self.hosts[dst_ip]
            if dpid == dst_dpid:
                out_port = self.mac_to_port[dpid].get(dst_mac)
                if out_port:
                    self._send_packet(dpid, out_port, packet)
                else:
                    self._flood_packet(event)
            else:
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

    # [Rest of the methods remain unchanged from previous version]
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
        try:
            return nx.shortest_path(self.graph, source=src_dpid, target=dst_dpid, 
                                  weight='weight')
        except nx.NetworkXNoPath:
            return None

    def _install_path_flows(self, path, src_mac, dst_mac, src_ip, dst_ip):
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

    def _request_stats(self, connection):
        r = of.ofp_stats_request(body=of.ofp_flow_stats_request())
        connection.send(r)
        core.callDelayed(5, self._request_stats, connection)

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
./pox.py log.level --DEBUG  forwarding.adaptive_routing
```

# 4. run mininet 

```
sudo mn --custom adaptive_routing_topology.py --controller=remote,ip=127.0.0.1 --topo=adaptive
```
