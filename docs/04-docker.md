

# 1. <p dir="rtl" align="justify">فایل‌های زیر را در دایرکتوری sdn-adaptive-routing قرار دهید:</p>
```bash
mkdir sdn-adaptive-routing
cd sdn-adaptive-routing
touch Dockerfile.pox docker-compose.yml adaptive_routing.py adaptive_routing_topology.py
```



# 2. Dockerfile for Pox
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

# 3. Dockerfile Dockerfile Mininet
```
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

# 4. docker-compose.yml
```
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

# 5. <p dir="rtl" align="justify">ساخت و اجرای کانتینرها:</p>
```
docker-compose build
docker-compose up
```

# 6. <p dir="rtl" align="justify">برای دسترسی به محیط Mininet:</p>
```
docker exec -it mininet bash
```

# 7. <p dir="rtl" align="justify">. تست شبکه</p>
```
mininet> pingall
mininet> h1 ping h2
```

# 8. <p dir="rtl" align="justify">برای دیباگ بهتر، می‌توانید لاگ‌های POX را با دستور زیر مشاهده کنید:</p>
```
docker logs -f pox
```
