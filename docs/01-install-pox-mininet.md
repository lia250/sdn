1.

```bash
cd pox
python3 pox.py openflow.of_01 --port=6633 forwarding.l2_learning
```

# 2. Creating a Mininet Topology

In a new terminal window, create a simple Mininet topology connected to your POX controller:

```bash
sudo mn --controller=remote,ip=127.0.0.1,port=6633 --topo=single,3
```

# 3. Testing the Network

1. Test connectivity between all hosts:

```bash
pingall
```

2. Test connectivity between specific hosts (e.g., h1 and h2):

```bash
h1 ping h2
```

3. View the flow table entries on the switch:

```bash
dpctl dump-flows
```


