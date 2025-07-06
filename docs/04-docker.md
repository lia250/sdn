# 1. Install Docker on Ubuntu:


1. update the system:
```bash
sudo apt update && sudo apt upgrade -y
```

2. Install the necessary packages:
```bash
sudo apt install -y apt-transport-https ca-certificates curl software-properties-common
```

3. Add the official Docker GPG key:
```bash
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
```

4. Adding Docker repository to apt sources:
```bash
echo "deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

5. Installing the Docker engine:
```bash
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io
```

6. Check the installation status:
```bash
sudo systemctl status docker
```


7. Add user to docker group (to run commands without sudo)
```bash
sudo usermod -aG docker $USER
newgrp docker
```

# 2. Install Docker Compose 

1. Download the latest version of Docker Compose
```bash
sudo curl -L "https://github.com/docker/compose/releases/download/v2.23.0/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
```

2. apply of executive licenses
```bash
sudo chmod +x /usr/local/bin/docker-compose
```

3. Create a symbolic link
```bash
sudo ln -s /usr/local/bin/docker-compose /usr/bin/docker-compose
```

4. Check the installed version
```bash
docker-compose --version
```

# 3. Installation test

1. Run the test container 
```bash
docker run hello-world
```

2. View running containers
```bash
docker ps
```

# 4. Docker service management

1. Start the service
```
sudo systemctl start docker
``` 
```
sudo systemctl status docker
```

2. Enable autorun at boot
```
sudo systemctl enable docker
```

3. Stop service
```
sudo systemctl stop docker
``` 



# 5. Place the following files in the sdn-adaptive-routing directory:

1. sdn-adaptive-routing:
```bash
mkdir sdn-adaptive-routing
cd sdn-adaptive-routing
touch Dockerfile.pox docker-compose.yml adaptive_routing.py adaptive_routing_topology.py
```

2. adaptive_routing.py (pox):
```python
from pox.core import core
import pox.openflow.libopenflow_01 as of
from pox.lib.util import dpidToStr
import networkx as nx

log = core.getLogger()


class AdaptiveRouting (object):
    def __init__(self):
        core.openflow.addListeners(self)

        # We enable Discovery within the code so that no additional argument is needed.
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

3. adaptive_routing_topology.py (mininet):
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

# 6. Dockerfile for Pox
```dockerfile
# Dockerfile.pox
FROM python:3.9-slim

RUN apt-get update && \
    apt-get install -y git && \
    rm -rf /var/lib/apt/lists/*

WORKDIR /pox
RUN git clone https://github.com/noxrepo/pox.git .

COPY adaptive_routing.py /pox/pox/forwarding/adaptive_routing.py

CMD ["./pox.py", "log.level", "--DEBUG", "forwarding.adaptive_routing"]
```

# 7. Dockerfile Dockerfile Mininet
```dockerfile
# Dockerfile.mininet
FROM ubuntu:20.04

ENV DEBIAN_FRONTEND=noninteractive

RUN apt-get update && \
    apt-get install -y \
    git \
    mininet \
    python3 \
    python3-pip \
    xterm \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /app
COPY adaptive_routing_topology.py .

CMD ["mn", "--custom", "adaptive_routing_topology.py", "--controller=remote,ip=pox,port=6633", "--topo=adaptive"]
```

# 8. docker-compose.yml
```yaml
version: '3'

services:
  pox:
    build:
      context: .
      dockerfile: Dockerfile.pox
    container_name: pox
    networks:
      - sdn-net

  mininet:
    build:
      context: .
      dockerfile: Dockerfile.mininet
    container_name: mininet
    privileged: true
    networks:
      - sdn-net
    depends_on:
      - pox
    stdin_open: true
    tty: true

networks:
  sdn-net:
    driver: bridge
```

# 9. Building and running containers:
```
docker-compose build
docker-compose up
```

# 10. To access the Mininet environment:
```
docker exec -it mininet bash
```

# 11. Network test:

```
mininet> pingall
mininet> h1 ping h2
```

# 12. For better debugging, you can view POX logs with the following command:
```
docker logs -f pox
```
