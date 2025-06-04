# 1. pox/pox/forwarding/simple_controller.py

```python
from pox.core import core
import pox.openflow.libopenflow_01 as of
from pox.lib.util import dpidToStr

log = core.getLogger()

class SimpleController(object):
    def __init__(self):
        core.openflow.addListeners(self)
    
    def _handle_ConnectionUp(self, event):
        log.info("Switch %s has connected.", dpidToStr(event.dpid))
        
        # Install a default flow entry that sends packets to the controller
        msg = of.ofp_flow_mod()
        msg.priority = 0  # Lowest priority
        msg.actions.append(of.ofp_action_output(port=of.OFPP_CONTROLLER))
        event.connection.send(msg)
        
        log.info("Default flow entry installed on switch %s", dpidToStr(event.dpid))
    
    def _handle_PacketIn(self, event):
        packet = event.parsed
        log.info("Packet received from switch %s on port %s", 
                 dpidToStr(event.dpid), event.port)
        
        # Flood the packet out all ports except the input port
        msg = of.ofp_packet_out()
        msg.data = event.ofp
        msg.actions.append(of.ofp_action_output(port=of.OFPP_FLOOD))
        msg.in_port = event.port
        event.connection.send(msg)
        
        log.info("Packet flooded from switch %s", dpidToStr(event.dpid))

def launch():
    core.registerNew(SimpleController)
```

# 2. simple_topology.py

```python
from mininet.net import Mininet
from mininet.node import RemoteController
from mininet.cli import CLI
from mininet.log import setLogLevel

def simpleTopo():
    net = Mininet(controller=RemoteController)
    
    # Add controller
    c0 = net.addController('c0', controller=RemoteController, ip='127.0.0.1', port=6633)
    
    # Add switches
    s1 = net.addSwitch('s1')
    s2 = net.addSwitch('s2')
    
    # Add hosts
    h1 = net.addHost('h1')
    h2 = net.addHost('h2')
    h3 = net.addHost('h3')
    h4 = net.addHost('h4')
    
    # Add links
    net.addLink(h1, s1)
    net.addLink(h2, s1)
    net.addLink(h3, s2)
    net.addLink(h4, s2)
    net.addLink(s1, s2)
    
    net.start()
    CLI(net)
    net.stop()

if __name__ == '__main__':
    setLogLevel('info')
    simpleTopo()
```
