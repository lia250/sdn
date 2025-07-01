# <p dir="rtl" align="justify">بخش 1: کلاس اصلی</p>

```python
class AdaptiveRouting (object):
    def __init__(self):
        core.openflow.addListeners(self)
        
        import pox.openflow.discovery
        pox.openflow.discovery.launch()
        core.openflow_discovery.addListeners(self)

        self.topology    = {}   # {dpid: {nbr_dpid: out_port}}
        self.mac_to_port = {}   # {dpid: {mac: port}}
        self.hosts       = {}   # {IPAddr: (dpid, mac)}
        self.graph       = nx.Graph()
```

# <p dir="rtl" align="justify">بخش 2: مدیریت اتصال سوئیچ‌ها</p>

```python
    def _handle_ConnectionUp(self, ev):
        dpid = ev.dpid
        log.info("🔌 Switch %s connected", dpidToStr(dpid))
        self.topology[dpid]    = {}
        self.mac_to_port[dpid] = {}
```

# <p dir="rtl" align="justify">بخش 3: مدیریت لینک‌های شبکه</p>

```python
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
```

# <p dir="rtl" align="justify">بخش 4: پردازش بسته‌های دریافتی</p>

```python
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
```

<p dir="rtl" align="justify">توضیحات کد:</p>

```python
def _handle_PacketIn(self, ev):
```

<p dir="rtl" align="justify">
  <ul dir="rtl">
    <li>تابعی که هنگام دریافت بسته از سوئیچ فراخوانی می‌شود</li>
	<li>ev شامل اطلاعات رویداد دریافتی است</li>
  </ul>
</p>

```python
    pkt, dpid, in_p = ev.parsed, ev.dpid, ev.port
```

<p dir="rtl" align="justify">
	<ul dir="rtl">
	  <li>تجزیه بسته دریافتی به سه بخش:
		<ul dir="rtl">
		  <li>pkt: بسته پارس شده</li>
		  <li>dpid: شناسه سوئیچ فرستنده</li>
		  <li>in_p: پورت ورودی که بسته از آن دریافت شده</li>
		</ul>
	  </li>
	</ul>
</p>

```python
    # learn MAC
    self.mac_to_port.setdefault(dpid, {}).setdefault(pkt.src, in_p)
```

<p dir="rtl" align="justify">
	<ul dir="rtl">
	  <li>یادگیری مکان MAC آدرس مبدأ:
		<ul dir="rtl">
		  <li>.ایجاد ساختار داده برای سوئیچ اگر وجود نداشته باشد</li>
		  <li>ثبت پورت ورودی برای MAC آدرس مبدأ</li>
		</ul>
	  </li>
	</ul>
</p>

```python
    # learn host on IP
    if pkt.type == pkt.IP_TYPE:
        self.hosts[pkt.payload.srcip] = (dpid, pkt.src)
```

<p dir="rtl" align="justify">
	<ul dir="rtl">
	  <li>اگر بسته از نوع IP باشد:
		<ul dir="rtl">
		  <li>ثبت مکان میزبان (IP به سوئیچ و MAC نگاشت می‌شود)</li>
		</ul>
	  </li>
	</ul>
</p>

```python
    # ARP processing
    if pkt.type == pkt.ARP_TYPE:
        self._handle_ARP(ev)
        return
```

<p dir="rtl" align="justify">
	<ul dir="rtl">
	  <li>اگر بسته از نوع ARP باشد:
		<ul dir="rtl">
		  <li>پردازش را به تابع مخصوص ARP واگذار می‌کند</li>
		  <li>از تابع خارج می‌شود</li>
		</ul>
	  </li>
	</ul>
</p>

```python
    # IPv4 routing
    if pkt.type == pkt.IP_TYPE:
        ip = pkt.payload
        dst_ip = ip.dstip
```

<p dir="rtl" align="justify">
	<ul dir="rtl">
	  <li>اگر بسته از نوع IP باشد (غیر از ARP):
		<ul dir="rtl">
		  <li>اطلاعات لایه IP را استخراج می‌کند</li>
		  <li>آدرس IP مقصد را می‌خواند</li>
		</ul>
	  </li>
	</ul>
</p>

```python
        if dst_ip not in self.hosts:
            self._flood(ev); return
```

<p dir="rtl" align="justify">
	<ul dir="rtl">
	  <li>اگر مقصد در لیست میزبان‌های شناخته شده نباشد:
		<ul dir="rtl">
		  <li>بسته را flood می‌کند</li>
		  <li>از تابع خارج می‌شود</li>
		</ul>
	  </li>
	</ul>
</p>

```python
        dst_dpid, dst_mac = self.hosts[dst_ip]
```

<p dir="rtl" align="justify">
	<ul dir="rtl">
	  <li>اطلاعات مقصد را از ساختار داده hosts می‌خواند:
		<ul dir="rtl">
		  <li>شناسه سوئیچ مقصد</li>
		  <li>MAC آدرس مقصد</li>
		</ul>
	  </li>
	</ul>
</p>

```python
        if dpid == dst_dpid:                       # same switch
            out = self.mac_to_port[dpid].get(dst_mac)
            self._unicast(dpid, out, pkt) if out else self._flood(ev)
```

<p dir="rtl" align="justify">
	<ul dir="rtl">
	  <li>اگر مبدأ و مقصد روی یک سوئیچ باشند:
		<ul dir="rtl">
		  <li>پورت خروجی به مقصد را پیدا می‌کند</li>
		  <li>اگر پورت وجود داشت بسته را unicast می‌کند</li>
		  <li>در غیر این صورت flood می‌کند</li>
		</ul>
	  </li>
	</ul>
</p>

```python
        else:
            path = self._shortest(dpid, dst_dpid)
```

<p dir="rtl" align="justify">
	<ul dir="rtl">
	  <li>اگر مبدأ و مقصد روی سوئیچ‌های مختلف باشند:
		<ul dir="rtl">
		  <li>کوتاه‌ترین مسیر بین سوئیچ‌ها را محاسبه می‌کند</li>
		  <li></li>
		  <li></li>
		</ul>
	  </li>
	</ul>
</p>

```python
            if path:
                log.debug("🛣 %s → %s : %s",
                          dpidToStr(dpid), dpidToStr(dst_dpid),
                          " → ".join(dpidToStr(sw) for sw in path))
                self._install_path(path, pkt.src, dst_mac,
                                   ip.srcip, dst_ip)
```

<p dir="rtl" align="justify">
	<ul dir="rtl">
	  <li>اگر مسیری یافت شد:
		<ul dir="rtl">
		  <li>مسیر را در لاگ ثبت می‌کند</li>
		  <li>قوانین جریان را در مسیر یافت شده نصب می‌کند</li>
		</ul>
	  </li>
	</ul>
</p>

```
            else:
                self._flood(ev)
```

<p dir="rtl" align="justify">
	<ul dir="rtl">
	  <li>اگر مسیری یافت نشد:
		<ul dir="rtl">
		  <li>بسته را flood می‌کند</li>
		</ul>
	  </li>
	</ul>
</p>

```python
    else:
        self._flood(ev)
```

<p dir="rtl" align="justify">
	<ul dir="rtl">
	  <li>اگر بسته از نوع IP نباشد (مثلاً IPv6 یا سایر پروتکل‌ها):
		<ul dir="rtl">
		  <li>بسته را flood می‌کند</li>
		</ul>
	  </li>
	</ul>
</p>

# <p dir="rtl" align="justify">بخش 5: پردازش ARP</p>

```python
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
```

# <p dir="rtl" align="justify">بخش 6: ابزارهای مسیریابی</p>

```python
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
```

# <p dir="rtl" align="justify">بخش 7: ابزارهای ارسال بسته</p>

```python
    def _flood(self, ev):
        ev.connection.send(of.ofp_packet_out(
            data=ev.ofp, in_port=ev.port,
            actions=[of.ofp_action_output(port=of.OFPP_FLOOD)]))

    def _unicast(self, dpid, port, pkt):
        if port is None: return self._flood(pkt)   # fallback
        core.openflow.sendToDPID(dpid, of.ofp_packet_out(
            data=pkt.pack(), actions=[of.ofp_action_output(port=port)]))
```

# <p dir="rtl" align="justify">بخش 8: راه‌اندازی کنترلر</p>

```python
def launch():
    core.registerNew(AdaptiveRouting)
```
