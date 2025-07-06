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
```bash
mkdir sdn-adaptive-routing
cd sdn-adaptive-routing
touch Dockerfile.pox docker-compose.yml adaptive_routing.py adaptive_routing_topology.py
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
