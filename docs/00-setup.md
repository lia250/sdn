# 1. install python 3.9 for pox

1. 

```
sudo apt update
sudo apt install software-properties-common
sudo add-apt-repository ppa:deadsnakes/ppa
sudo apt update
sudo apt install python3.9 python3.9-dev python3.9-venv
python3.9 --version
```

2. 

```
python3.9 -m venv pox-env
source pox-env/bin/activate
```

3.

```
sudo apt install mininet git
```

```
git clone https://github.com/noxrepo/pox.git
```

# 2. ssh

```
sudo nano /etc/ssh/sshd_config
```

```
ClientAliveInterval 60      
ClientAliveCountMax 3       
```

```
sudo systemctl restart sshd
```
