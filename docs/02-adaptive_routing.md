# 1. pox

pox/pox/forwrading/adaptive_routing.py

```python
# -*- coding: utf-8 -*-
"""
AdaptiveRouting – POX SDN Controller (سازگار با POX 0.7.0 gar)
  * کشف توپولوژی:  openflow.discovery
  * کوتاه‌ترین مسیر:  NetworkX
"""
from pox.core import core
import pox.openflow.libopenflow_01 as of
from pox.lib.util import dpidToStr
import networkx as nx

log = core.getLogger()


class AdaptiveRouting (object):
    def __init__(self):
        core.openflow.addListeners(self)

        # Discovery را درون کد فعال می‌کنیم تا نیاز به آرگومان اضافی نباشد
        import pox.openflow.discovery
        pox.openflow.discovery.launch()
        core.openflow_discovery.addListeners(self)

        self.topology    = {}   # {dpid: {nbr_dpid: out_port}}
        self.mac_to_port = {}   # {dpid: {mac: port}}
        self.hosts       = {}   # {IPAddr: (dpid, mac)}
        self.graph       = nx.Graph()

    # ───────── Switch connect ─────────
    def _handle_ConnectionUp(self, ev):
        dpid = ev.dpid
        log.info("🔌 Switch %s connected", dpidToStr(dpid))
        self.topology[dpid]    = {}
        self.mac_to_port[dpid] = {}

    # ───────── Link events (LLDP) ─────────
    def _handle_LinkEvent(self, ev):
        l = ev.link                # (dpid1,port1) ↔ (dpid2,port2)
        s1,p1,s2,p2 = l.dpid1, l.port1, l.dpid2, l.port2

        if ev.added:
            self.topology.setdefault(s1, {})[s2] = p1
            self.topology.setdefault(s2, {})[s1] = p2
            self.graph.add_edge(s1, s2)
            log.info("➕ %s:%d ↔ %s:%d", dpidToStr(s1),p1, dpidToStr(s2),p2)

        if ev.removed:
            self.topology.get(s1, {}).pop(s2, None)
            self.topology.get(s2, {}).pop(s1, None)
            if self.graph.has_edge(s1, s2):
                self.graph.remove_edge(s1, s2)
            log.info("➖ %s:%d ↔ %s:%d", dpidToStr(s1),p1, dpidToStr(s2),p2)

    # ───────── Packet-In ─────────
    def _handle_PacketIn(self, ev):
        pkt, dpid, in_p = ev.parsed, ev.dpid, ev.port

        # learn MAC
        self.mac_to_port.setdefault(dpid, {}).setdefault(pkt.src, in_p)

        # learn host on IP
        if pkt.type == pkt.IP_TYPE:
            self.hosts[pkt.payload.srcip] = (dpid, pkt.src)

        # ARP processing
        if pkt.type == pkt.ARP_TYPE:
            self._handle_ARP(ev)
            return

        # IPv4 routing
        if pkt.type == pkt.IP_TYPE:
            ip = pkt.payload
            dst_ip = ip.dstip

            if dst_ip not in self.hosts:
                self._flood(ev); return

            dst_dpid, dst_mac = self.hosts[dst_ip]

            if dpid == dst_dpid:                       # same switch
                out = self.mac_to_port[dpid].get(dst_mac)
                self._unicast(dpid, out, pkt) if out else self._flood(ev)
            else:
                path = self._shortest(dpid, dst_dpid)
                if path:
                    log.debug("🛣 %s → %s : %s",
                              dpidToStr(dpid), dpidToStr(dst_dpid),
                              " → ".join(dpidToStr(sw) for sw in path))
                    self._install_path(path, pkt.src, dst_mac,
                                       ip.srcip, dst_ip)
                else:
                    self._flood(ev)
        else:
            self._flood(ev)

    # ───────── ARP helper ─────────
    def _handle_ARP(self, ev):
        pkt, arp, dpid = ev.parsed, ev.parsed.payload, ev.dpid

        if arp.opcode == arp.REQUEST:
            # مقصد را نمی‌شناسیم → Flood
            self._flood(ev)

        elif arp.opcode == arp.REPLY:
            # learn from reply
            self.hosts[arp.protosrc] = (dpid, pkt.src)
            # forward to requester if seen
            out = self.mac_to_port[dpid].get(arp.hwdst)
            self._unicast(dpid, out, pkt) if out else self._flood(ev)

    # ───────── Path utilities ─────────
    def _shortest(self, s, d):
        try:    return nx.shortest_path(self.graph, s, d)
        except: return None

    def _install_path(self, path, src_mac, dst_mac, src_ip, dst_ip):
        for i, sw in enumerate(path):
            if i < len(path)-1:   next_sw = path[i+1]; out = self.topology[sw][next_sw]
            else:                 out = self.mac_to_port[sw][dst_mac]

            # forward flow
            fm = of.ofp_flow_mod()
            fm.match = of.ofp_match(dl_type=0x0800, nw_src=src_ip, nw_dst=dst_ip)
            fm.actions.append(of.ofp_action_output(port=out))
            core.openflow.sendToDPID(sw, fm)

            # reverse flow
            fm_b = of.ofp_flow_mod()
            fm_b.match = of.ofp_match(dl_type=0x0800, nw_src=dst_ip, nw_dst=src_ip)
            out_b = self.topology[sw][path[i-1]] if i else self.mac_to_port[sw][src_mac]
            fm_b.actions.append(of.ofp_action_output(port=out_b))
            core.openflow.sendToDPID(sw, fm_b)

    # ───────── I/O helpers ─────────
    def _flood(self, ev):
        ev.connection.send(of.ofp_packet_out(
            data=ev.ofp, in_port=ev.port,
            actions=[of.ofp_action_output(port=of.OFPP_FLOOD)]))

    def _unicast(self, dpid, port, pkt):
        if port is None: return self._flood(pkt)   # fallback
        core.openflow.sendToDPID(dpid, of.ofp_packet_out(
            data=pkt.pack(), actions=[of.ofp_action_output(port=port)]))

# ───────── Launcher ─────────
def launch():
    core.registerNew(AdaptiveRouting)



```

# 2. run controller pox

```
./pox.py log.level --DEBUG  forwarding.adaptive_routing
```


# 3. mininet

adaptive_routing_topology.py

```python
from mininet.topo import Topo

class AdaptiveTopo(Topo):
    def __init__(self):
        super(AdaptiveTopo, self).__init__()
        s1,s2,s3 = (self.addSwitch(n) for n in ('s1','s2','s3'))
        h1 = self.addHost('h1', ip='10.0.0.1/24', mac='00:00:00:00:00:01')
        h2 = self.addHost('h2', ip='10.0.0.2/24', mac='00:00:00:00:00:02')
        h3 = self.addHost('h3', ip='10.0.0.3/24', mac='00:00:00:00:00:03')

        self.addLink(h1, s1, port1=1, port2=1)
        self.addLink(h2, s2, port1=1, port2=1)
        self.addLink(h3, s3, port1=1, port2=1)

        self.addLink(s1, s2, port1=2, port2=2)
        self.addLink(s2, s3, port1=3, port2=2)
        self.addLink(s3, s1, port1=3, port2=3)

topos = {'adaptive': (lambda: AdaptiveTopo())}

```



# 4. run mininet 

```
sudo mn --custom adaptive_routing_topology.py --controller=remote,ip=127.0.0.1 --topo=adaptive
```

