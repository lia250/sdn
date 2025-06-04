# 1. Install Required Packages

```bash
sudo apt update
sudo apt install mininet git python3
```

# 2. Set Up POX Controller

```bash
git clone https://github.com/noxrepo/pox.git
```

```bash
cd pox
python3 pox.py openflow.of_01 --port=6634 forwarding.l2_learning
```

# 3. Creating a Mininet Topology

In a new terminal window, create a simple Mininet topology connected to your POX controller:

```bash
sudo mn --controller=remote,ip=127.0.0.1,port=6634 --topo=single,3
```

# 4. Testing the Network

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


