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

<p dir="rtl" align="justify">توضیحات کد:</p>

```python
class AdaptiveRouting (object):
    def __init__(self):
        core.openflow.addListeners(self)
```

<p dir="rtl" align="justify">
  <ul dir="rtl">
    <li>تعریف کلاس اصلی کنترلر با نام AdaptiveRouting</li>
	<li>متد سازنده (constructor) کلاس که هنگام ایجاد نمونه از کلاس اجرا می‌شود.</li>
	<li>ثبت کلاس فعلی به عنوان listener برای رویدادهای OpenFlow، این باعث می‌شود متدهایی مانند handle_PacketIn_ برای رویدادهای مربوطه فراخوانی شوند.</li>
  </ul>
</p>

```python
        import pox.openflow.discovery
        pox.openflow.discovery.launch()
```

<p dir="rtl" align="justify">
  <ul dir="rtl">
    <li>ایمپورت و راه‌اندازی ماژول کشف توپولوژی POX</li>
	<li>این ماژول با استفاده از پروتکل LLDP لینک‌های بین سوئیچ‌ها را کشف می‌کند</li>
  </ul>
</p>

```python
        core.openflow_discovery.addListeners(self)
```

<p dir="rtl" align="justify">
  <ul dir="rtl">
    <li>ثبت کلاس فعلی به عنوان listener برای رویدادهای کشف توپولوژی</li>
	<li>این باعث می‌شود متدهایی مانند handle_LinkEvent_ برای رویدادهای مربوطه فراخوانی شوند</li>
  </ul>
</p>

```python
        self.topology    = {}   # {dpid: {nbr_dpid: out_port}}
```

<p dir="rtl" align="justify">
	<ul dir="rtl">
	  <li>ایجاد دیکشنری برای نگهداری توپولوژی شبکه:
		<ul dir="rtl">
		  <li>کلید: شناسه سوئیچ (dpid)</li>
		  <li>مقدار: دیکشنری از همسایه‌ها و پورت‌های خروجی به آنها</li>
		</ul>
	  </li>
	</ul>
</p>

```python
        self.mac_to_port = {}   # {dpid: {mac: port}}
```

<p dir="rtl" align="justify">
	<ul dir="rtl">
	  <li>ایجاد دیکشنری برای نگاشت MAC آدرس‌ها به پورت‌ها:
		<ul dir="rtl">
		  <li>کلید سطح اول: شناسه سوئیچ</li>
		  <li>کلید سطح دوم: MAC آدرس</li>
		  <li>مقدار: شماره پورت</li>
		</ul>
	  </li>
	</ul>
</p>

```python
        self.hosts       = {}   # {IPAddr: (dpid, mac)}
```

<p dir="rtl" align="justify">
	<ul dir="rtl">
	  <li>ایجاد دیکشنری برای نگاشت آدرس IP به میزبان‌ها:
		<ul dir="rtl">
		  <li>کلید: آدرس IP</li>
		  <li>مقدار: tuple شامل (شناسه سوئیچ، MAC آدرس)</li>
		</ul>
	  </li>
	</ul>
</p>

```python
        self.graph       = nx.Graph()
```

<p dir="rtl" align="justify">
  <ul dir="rtl">
    <li>ایجاد یک گراف خالی با استفاده از کتابخانه NetworkX</li>
	<li>این گراف برای محاسبات مسیریابی و یافتن کوتاه‌ترین مسیر استفاده می‌شود</li>
  </ul>
</p>


# <p dir="rtl" align="justify">بخش 2: مدیریت اتصال سوئیچ‌ها</p>

```python
    def _handle_ConnectionUp(self, ev):
        dpid = ev.dpid
        log.info("🔌 Switch %s connected", dpidToStr(dpid))
        self.topology[dpid]    = {}
        self.mac_to_port[dpid] = {}
```

<p dir="rtl" align="justify">توضیحات کد:</p>

```python
    def _handle_ConnectionUp(self, ev):
```

<p dir="rtl" align="justify">
	<ul dir="rtl">
	  <li>تابع مدیریت اتصال سوئیچ‌ها:
		<ul dir="rtl">
		  <li>یک متد callback که هنگام برقراری ارتباط جدید با یک سوئیچ OpenFlow فراخوانی می‌شود</li>
		  <li>ev پارامتر رویداد دریافتی حاوی اطلاعات اتصال</li>
		</ul>
	  </li>
	</ul>
</p>

```python
        dpid = ev.dpid
```

<p dir="rtl" align="justify">
	<ul dir="rtl">
	  <li>استخراج شناسه سوئیچ:
		<ul dir="rtl">
		  <li>ev.dpid حاوی شناسه منحصربفرد سوئیچ (DPID - Datapath ID) است</li>
		  <li>این مقدار در متغیر محلی dpid ذخیره می‌شود</li>
		</ul>
	  </li>
	</ul>
</p>

```python
        log.info("🔌 Switch %s connected", dpidToStr(dpid))
```

<p dir="rtl" align="justify">
	<ul dir="rtl">
	  <li>ثبت رویداد اتصال در لاگ:
		<ul dir="rtl">
		  <li>نمایش پیغام اطلاع‌رسانی در لاگ سیستم</li>
		  <li>استفاده از آیکون 🔌 برای نشان دادن اتصال</li>
		  <li>تبدیل DPID به فرمت قابل خواندن با تابع dpidToStr</li>
		</ul>
	  </li>
	</ul>
</p>

```python
        self.topology[dpid]    = {}
```

<p dir="rtl" align="justify">
	<ul dir="rtl">
	  <li>مقداردهی اولیه توپولوژی برای سوئیچ جدید:
		<ul dir="rtl">
		  <li>ایجاد یک entry جدید در دیکشنری topology با کلید DPID سوئیچ</li>
		  <li>مقداردهی اولیه به عنوان یک دیکشنری خالی برای ذخیره اطلاعات همسایگی</li>
		</ul>
	  </li>
	</ul>
</p>

```python
        self.mac_to_port[dpid] = {}
```

<p dir="rtl" align="justify">
	<ul dir="rtl">
	  <li>مقداردهی اولیه جدول MAC برای سوئیچ جدید:
		<ul dir="rtl">
		  <li>ایجاد یک entry جدید در دیکشنری mac_to_port با کلید DPID سوئیچ</li>
		  <li>مقداردهی اولیه به عنوان یک دیکشنری خالی برای ذخیره نگاشت MAC به پورت</li>
		</ul>
	  </li>
	</ul>
</p>

## <p dir="rtl" align="justify">نکات کلیدی عملکرد:</p>

<p dir="rtl" align="justify">
	<ul dir="rtl">
	  <li>1. این تابع به ازای هر سوئیچ جدیدی که به کنترلر متصل می‌شود دقیقاً یکبار فراخوانی می‌شود</li>
	  <li>2. دو ساختار داده اصلی برای سوئیچ جدید مقداردهی اولیه می‌شوند:
		<ul dir="rtl">
		  <li>topology[dpid]: برای ذخیره اطلاعات لینک‌های همسایگی</li>
		  <li>mac_to_port[dpid]: برای یادگیری و نگاشت آدرس‌های MAC</li>
		</ul>
	  </li>
   	  <li>3. پیام لاگ به اپراتور شبکه کمک می‌کند از وضعیت اتصال سوئیچ‌ها مطلع شود</li>
	</ul>
</p>

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

<p dir="rtl" align="justify">توضیحات کد:</p>

```python
    def _handle_ARP(self, ev):
```

<p dir="rtl" align="justify">
	<ul dir="rtl">
	  <li>تابع پردازش بسته‌های ARP:
		<ul dir="rtl">
		  <li>متد اختصاصی برای مدیریت ترافیک ARP</li>
		  <li>ev پارامتر رویداد دریافتی از سوئیچ</li>
		</ul>
	  </li>
	</ul>
</p>

```python
        pkt, arp, dpid = ev.parsed, ev.parsed.payload, ev.dpid
```

<p dir="rtl" align="justify">
	<ul dir="rtl">
	  <li>تفکیک اطلاعات بسته:
		<ul dir="rtl">
		  <li>pkt: بسته شبکه پارس شده (لایه 2)</li>
		  <li>arp: payload بسته که حاوی اطلاعات ARP است</li>
		  <li>dpid: شناسه سوئیچ فرستنده بسته</li>
		</ul>
	  </li>
	</ul>
</p>

```python
        if arp.opcode == arp.REQUEST:
```

<p dir="rtl" align="justify">
	<ul dir="rtl">
	  <li>بررسی نوع درخواست ARP:
		<ul dir="rtl">
		  <li>اگر بسته یک درخواست ARP (ARP Request) باشد</li>
		</ul>
	  </li>
	</ul>
</p>

```python
            self._flood(ev)
```

<p dir="rtl" align="justify">
  <ul dir="rtl">
    <li>فراخوانی تابع _flood برای ارسال بسته به همه پورت‌ها (به جز پورت ورودی)</li>
  </ul>
</p>

```python
        elif arp.opcode == arp.REPLY:
```

<p dir="rtl" align="justify">
	<ul dir="rtl">
	  <li>بررسی نوع پاسخ ARP:
		<ul dir="rtl">
		  <li>اگر بسته یک پاسخ ARP (ARP Reply) باشد</li>
		</ul>
	  </li>
	</ul>
</p>

```python
            # learn from reply
            self.hosts[arp.protosrc] = (dpid, pkt.src)
```

<p dir="rtl" align="justify">
	<ul dir="rtl">
	  <li>یادگیری اطلاعات میزبان:
		<ul dir="rtl">
		  <li>کامنت نشان می‌دهد که اطلاعات از پاسخ ARP یادگرفته می‌شود</li>
		  <li>ثبت mapping آدرس IP مبدأ (protosrc) به:
			<ul>
				<li>شناسه سوئیچ (dpid)</li>
				<li>MAC آدرس مبدأ (pkt.src)</li>
			</ul>
		  </li>
		</ul>
	  </li>
	</ul>
</p>

```python
            # forward to requester if seen
            out = self.mac_to_port[dpid].get(arp.hwdst)
```

<p dir="rtl" align="justify">
	<ul dir="rtl">
	  <li>یافتن پورت مقصد:
		<ul dir="rtl">
		  <li>کامنت نشان می‌دهد که اگر درخواست‌کننده دیده شده باشد، پاسخ به او ارسال می‌شود</li>
		  <li>جستجوی پورت خروجی برای MAC آدرس مقصد (arp.hwdst) در سوئیچ فعلی</li>
		</ul>
	  </li>
	</ul>
</p>

```python
            self._unicast(dpid, out, pkt) if out else self._flood(ev)
```

<p dir="rtl" align="justify">
	<ul dir="rtl">
	  <li>ارسال پاسخ ARP:
		<ul dir="rtl">
		  <li>اگر پورت خروجی پیدا شد (out وجود دارد):
			<ul>
				<li>ارسال unicast به پورت مشخص شده</li>
			</ul>
		  </li>
		  <li>در غیر این صورت:
			<ul>
				<li>flood کردن بسته به همه پورت‌ها</li>
			</ul>
		  </li>
		</ul>
	  </li>
	</ul>
</p>

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
