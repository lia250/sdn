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
				<ul dir="rtl">
					<li>flood کردن بسته به همه پورت‌ها</li>
				</ul>
			  </li>
		</ul>
	  </li>
	</ul>
</p>

## <p dir="rtl" align="justify">نکات کلیدی عملکرد:</p>

<p dir="rtl" align="justify">
	<ul dir="rtl">
	  <li>1. این تابع دو نوع بسته ARP را پردازش می‌کند:
		<ul dir="rtl">
		  <li>ARP Request (درخواست resolve آدرس)</li>
		  <li>ARP Reply (پاسخ به درخواست)</li>
		</ul>
	  </li>
		  <li>2. برای درخواست‌های ARP:
		<ul dir="rtl">
		  <li>همیشه flood می‌شوند تا میزبان مقصد بتواند پاسخ دهد</li>
		</ul>
	  </li>
	  	  <li>3. برای پاسخ‌های ARP:
		<ul dir="rtl">
		  <li>اطلاعات میزبان در ساختار داده hosts ذخیره می‌شود</li>
		  <li>سعی می‌کند پاسخ را فقط به درخواست‌کننده ارسال کند (unicast)</li>
		  <li>اگر درخواست‌کننده شناخته شده نباشد، پاسخ flood می‌شود</li>
		</ul>
	  </li>
	  	  <li>4. یادگیری خودکار:
		<ul dir="rtl">
		  <li>سیستم به صورت خودکار mapping بین IP و MAC را از پاسخ‌های ARP یاد می‌گیرد</li>
		  <li>این اطلاعات بعداً برای مسیریابی بسته‌های IP استفاده می‌شود</li>
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


<p dir="rtl" align="justify">توضیحات کد:</p>

```python
    def _shortest(self, s, d):
```

<p dir="rtl" align="justify">
	<ul dir="rtl">
	  <li>تابع یافتن کوتاه‌ترین مسیر:
		<ul dir="rtl">
		  <li>ورودی: شناسه سوئیچ مبدأ (s) و مقصد (d)</li>
		  <li>خروجی: لیستی از شناسه سوئیچ‌های مسیر</li>
		</ul>
	  </li>
	</ul>
</p>

```python
        try:    return nx.shortest_path(self.graph, s, d)
```

<p dir="rtl" align="justify">
	<ul dir="rtl">
	  <li>محاسبه مسیر با NetworkX:
		<ul dir="rtl">
		  <li>استفاده از تابع shortest_path کتابخانه NetworkX</li>
		  <li>محاسبه کوتاه‌ترین مسیر در گراف توپولوژی (self.graph)</li>
		</ul>
	  </li>
	</ul>
</p>

```python
        except: return None
```

<p dir="rtl" align="justify">
	<ul dir="rtl">
	  <li>مدیریت خطا:
		<ul dir="rtl">
		  <li>اگر مسیری وجود نداشته باشد (مثلاً سوئیچ‌ها disconnected باشند)</li>
		  <li>مقدار None برگردانده می‌شود</li>
		</ul>
	  </li>
	</ul>
</p>

```python
    def _install_path(self, path, src_mac, dst_mac, src_ip, dst_ip):
```

<p dir="rtl" align="justify">
	<ul dir="rtl">
	  <li>تابع نصب قوانین جریان در مسیر:
		<ul dir="rtl">
		  <li>ورودی‌ها:
			  <ul dir="rtl">
				<li>path: لیست سوئیچ‌های مسیر</li>
				<li>src_mac: MAC آدرس مبدأ</li>
				  <li>dst_mac: MAC آدرس مقصد</li>
				  <li>src_ip: IP مبدأ</li>
				  <li>dst_ip: IP مقصد</li>
			</ul>
		  </li>		
		  </li>
		</ul>
	  </li>
	</ul>
</p>

```python
        for i, sw in enumerate(path):
```

<p dir="rtl" align="justify">
	<ul dir="rtl">
	  <li>حلقه روی تمام سوئیچ‌های مسیر:
		<ul dir="rtl">
		  <li>i: اندیس سوئیچ در مسیر</li>
		  <li>sw: شناسه سوئیچ فعلی</li>
		</ul>
	  </li>
	</ul>
</p>

```python
            if i < len(path)-1:   next_sw = path[i+1]; out = self.topology[sw][next_sw]
            else:                 out = self.mac_to_port[sw][dst_mac]
```

<p dir="rtl" align="justify">
	<ul dir="rtl">
	  <li>تعیین پورت خروجی:
		<ul dir="rtl">
		  <li>برای سوئیچ‌های میانی: پورت به سوئیچ بعدی در مسیر</li>
		  <li>برای سوئیچ آخر: پورت به میزبان مقصد (از جدول mac_to_port)</li>
		</ul>
	  </li>
	</ul>
</p>

```python
            # forward flow
            fm = of.ofp_flow_mod()
```

<p dir="rtl" align="justify">
	<ul dir="rtl">
	  <li>ساخت قانون جریان برای جهت رفت:
		<ul dir="rtl">
		  <li>ایجاد یک شیء ofp_flow_mod برای تعریف قانون جریان</li>
		</ul>
	  </li>
	</ul>
</p>

```python
            fm.match = of.ofp_match(dl_type=0x0800, nw_src=src_ip, nw_dst=dst_ip)
```

<p dir="rtl" align="justify">
	<ul dir="rtl">
	  <li>تعریف تطابق (match):
		<ul dir="rtl">
		  <li>dl_type=0x0800: فقط بسته‌های IPv4</li>
		  <li>nw_src و nw_dst: تطابق با آدرس‌های IP مبدأ و مقصد</li>
		</ul>
	  </li>
	</ul>
</p>

```python
            fm.actions.append(of.ofp_action_output(port=out))
```

<p dir="rtl" align="justify">
	<ul dir="rtl">
	  <li>تعریف اکشن:
		<ul dir="rtl">
		  <li>ارسال بسته از پورت خروجی محاسبه شده</li>
		</ul>
	  </li>
	</ul>
</p>

```python
            core.openflow.sendToDPID(sw, fm)
```

<p dir="rtl" align="justify">
	<ul dir="rtl">
	  <li>ارسال قانون به سوئیچ:
		<ul dir="rtl">
		  <li>ارسال قانون جریان به سوئیچ مربوطه</li>
		</ul>
	  </li>
	</ul>
</p>

```python
            # reverse flow
            fm_b = of.ofp_flow_mod()
```

<p dir="rtl" align="justify">
	<ul dir="rtl">
	  <li>ساخت قانون جریان برای جهت برگشت:
		<ul dir="rtl">
		  <li>مشابه جهت رفت، اما برای ترافیک معکوس</li>
		</ul>
	  </li>
	</ul>
</p>

```python
fm_b.match = of.ofp_match(dl_type=0x0800, nw_src=dst_ip, nw_dst=src_ip)
```

<p dir="rtl" align="justify">
	<ul dir="rtl">
	  <li>تعریف تطابق برای جهت برگشت:
		<ul dir="rtl">
		  <li>آدرس‌های مبدأ و مقصد معکوس شده‌اند</li>
		</ul>
	  </li>
	</ul>
</p>

```python
            out_b = self.topology[sw][path[i-1]] if i else self.mac_to_port[sw][src_mac]
```

<p dir="rtl" align="justify">
	<ul dir="rtl">
	  <li>تعیین پورت خروجی جهت برگشت:
		<ul dir="rtl">
		  <li>برای سوئیچ‌های میانی: پورت به سوئیچ قبلی در مسیر</li>
		  <li>برای سوئیچ اول: پورت به میزبان مبدأ</li>
		</ul>
	  </li>
	</ul>
</p>


```python
            fm_b.actions.append(of.ofp_action_output(port=out_b))
            core.openflow.sendToDPID(sw, fm_b)
```

<p dir="rtl" align="justify">
	<ul dir="rtl">
	  <li>تعریف و ارسال قانون برگشت:
		<ul dir="rtl">
		  <li>مشابه جهت رفت، اما با پارامترهای معکوس</li>
		</ul>
	  </li>
	</ul>
</p>

## <p dir="rtl" align="justify">نکات کلیدی عملکرد:</p>

<p dir="rtl" align="justify">
	<ul dir="rtl">
	  <li>1. تابع _shortest:
		<ul dir="rtl">
		  <li>از الگوریتم‌های پیشرفته NetworkX برای یافتن کوتاه‌ترین مسیر استفاده می‌کند</li>
		  <li>به صورت خودکار خطاها را مدیریت می‌کند</li>
		</ul>
	  </li>
	  <li>2. تابع _install_path:
		<ul dir="rtl">
		  <li>برای هر مسیر دو سری قانون ایجاد می‌کند (رفت و برگشت)</li>
		  <li>قوانین به صورت دقیق بر اساس آدرس‌های IP و MAC فیلتر می‌شوند</li>
		  <li>از ofp_flow_mod برای تعریف قوانین استفاده می‌کند</li>
		  <li>قوانین مستقیماً به سوئیچ‌های مربوطه ارسال می‌شوند</li>
		</ul>
	  </li>
	  <li>3. بهینه‌سازی:
		<ul dir="rtl">
		  <li>نصب همزمان قوانین رفت و برگشت باعث کاهش تاخیر در ارتباطات دوطرفه می‌شود</li>
		  <li>تطابق دقیق بر اساس IP/MAC از تداخل قوانین جلوگیری می‌کند</li>
		</ul>
	  </li>
	</ul>
</p>


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


## <p dir="rtl" align="justify"توضیحات کد:</p>

```python
    def _flood(self, ev):
```

<p dir="rtl" align="justify">
	<ul dir="rtl">
	  <li>تابع Flood (ارسال به همه پورت‌ها):
		<ul dir="rtl">
		  <li>برای زمانی که مسیر مشخصی وجود ندارد</li>
		  <li>ev: شیء رویداد دریافتی از سوئیچ</li>
		</ul>
	  </li>
	</ul>
</p>

```python
        ev.connection.send(of.ofp_packet_out(
```

<p dir="rtl" align="justify">
	<ul dir="rtl">
	  <li>ساخت و ارسال بسته خروجی:
		<ul dir="rtl">
		  <li>استفاده از ofp_packet_out برای ساخت بسته OpenFlow</li>
		  <li>ارسال از طریق اتصال سوئیچ (ev.connection)</li>
		</ul>
	  </li>
	</ul>
</p>

```python
            data=ev.ofp, in_port=ev.port,
```

<p dir="rtl" align="justify">
	<ul dir="rtl">
	  <li>تنظیم پارامترهای بسته:
		<ul dir="rtl">
		  <li>data: محتوای بسته اصلی (ev.ofp)</li>
		  <li>in_port: پورت ورودی که بسته از آن دریافت شده</li>
		</ul>
	  </li>
	</ul>
</p>

```python
            actions=[of.ofp_action_output(port=of.OFPP_FLOOD)]))
```

<p dir="rtl" align="justify">
	<ul dir="rtl">
	  <li>تعریف اکشن Flood:
		<ul dir="rtl">
		  <li>OFPP_FLOOD: ثابت مشخص‌کننده flood کردن بستهd</li>
		  <li>بسته به همه پورت‌ها (به جز پورت ورودی) ارسال می‌شود</li>
		</ul>
	  </li>
	</ul>
</p>

```python
    def _unicast(self, dpid, port, pkt):
```

<p dir="rtl" align="justify">
	<ul dir="rtl">
	  <li>تابع ارسال تک‌پخشی (Unicast):
		<ul dir="rtl">
		  <li>برای ارسال بسته به یک پورت خاص</li>
		  <li>پارامترها:
			<ul>
				<li>dpid: شناسه سوئیچ مقصد</li>
				<li>port: پورت خروجی در سوئیچ</li>
				<li>pkt: بسته شبکه</li>
			</ul>
		  </li>
		</ul>
	  </li>
	</ul>
</p>

```python
        if port is None: return self._flood(pkt)   # fallback
```

<p dir="rtl" align="justify">
	<ul dir="rtl">
	  <li>بررسی fallback به حالت Flood:
		<ul dir="rtl">
		  <li>اگر پورت خروجی مشخص نباشد (None)</li>
		  <li>از تابع _flood به عنوان راهکار جایگزین استفاده می‌کند</li>
		</ul>
	  </li>
	</ul>
</p>

```python
        core.openflow.sendToDPID(dpid, of.ofp_packet_out(
```

<p dir="rtl" align="justify">
	<ul dir="rtl">
	  <li>ارسال هدفمند به سوئیچ خاص:
		<ul dir="rtl">
		  <li>استفاده از sendToDPID برای ارسال به سوئیچ مشخص</li>
		  <li>ساخت بسته خروجی OpenFlow</li>
		</ul>
	  </li>
	</ul>
</p>

```python
            data=pkt.pack(), actions=[of.ofp_action_output(port=port)]))
```

<p dir="rtl" align="justify">
	<ul dir="rtl">
	  <li>تنظیم پارامترهای بسته:
		<ul dir="rtl">
		  <li>data: بسته شبکه بسته‌بندی شده (pkt.pack())</li>
		  <li>actions: اکشن ارسال به پورت مشخص (port)</li>
		</ul>
	  </li>
	</ul>
</p>

## <p dir="rtl" align="justify">نکات کلیدی عملکرد:</p>

<p dir="rtl" align="justify">
	<ul dir="rtl">
	  <li>1. تابع _flood:
		<ul dir="rtl">
		  <li>برای کشف شبکه و زمانی که اطلاعات مسیریابی وجود ندارد استفاده می‌شود</li>
		  <li>از مکانیزم استاندارد OpenFlow برای flood کردن استفاده می‌کند</li>
		  <li>به صورت خودکار از ارسال به پورت ورودی جلوگیری می‌کند (Loop Avoidance)</li>
		</ul>
	  </li>
	  	<li>2. تابع _unicast:
		<ul dir="rtl">
		  <li>برای ارسال هدفمند بسته‌ها استفاده می‌شود</li>
		  <li>دارای مکانیزم fallback به حالت flood هنگام عدم وجود اطلاعات مسیریابی</li>
		  <li>از pack() برای سریالایز کردن بسته قبل از ارسال استفاده می‌کند</li>
		</ul>
	  </li>
	  	<li>3. تفاوت‌های کلیدی:
		<ul dir="rtl">
		  <li>_flood از اتصال موجود سوئیچ (ev.connection) استفاده می‌کند</li>
		  <li>_unicast از مکانیزم ارسال بر اساس DPID استفاده می‌کند</li>
		  <li>_flood برای رویدادهای لحظه‌ای، _unicast برای ارسال‌های برنامه‌ریزی شده</li>
		</ul>
	  </li>
	  	<li>4. بهینه‌سازی‌ها:
		<ul dir="rtl">
		  <li>کاهش سربار پردازش با استفاده مستقیم از بسته‌های دریافتی</li>
		  <li>استفاده از مکانیزم‌های استاندارد OpenFlow برای عملکرد بهینه</li>
		</ul>
	  </li>
	</ul>
</p>

# <p dir="rtl" align="justify">بخش 8: راه‌اندازی کنترلر</p>

```python
def launch():
    core.registerNew(AdaptiveRouting)
```

<p dir="rtl" align="justify">توضیحات کد:</p>

```python
def launch():
```

<p dir="rtl" align="justify">
	<ul dir="rtl">
	  <li>تابع راه‌اندازی اصلی کنترلر:
		<ul dir="rtl">
		  <li>تابع استاندارد POX که هنگام اجرای کامپوننت فراخوانی می‌شود</li>
		  <li>نقطه ورود اصلی برای فعال‌سازی ماژول کنترلر</li>
		</ul>
	  </li>
	</ul>
</p>

```python
    core.registerNew(AdaptiveRouting)
```

<p dir="rtl" align="justify">
	<ul dir="rtl">
	  <li>ثبت کامپوننت کنترلر:
		<ul dir="rtl">
		  <li>core.registerNew(): متد اصلی POX برای ثبت کامپوننت‌های جدید</li>
		  <li>AdaptiveRouting: کلاس اصلی کنترلر که به عنوان پارامتر داده می‌شود</li>
		</ul>
	  </li>
	</ul>
</p>

## <p dir="rtl" align="justify">نکات کلیدی عملکرد:</p>

<p dir="rtl" align="justify">
	<ul dir="rtl">
	  <li>1. الزامات POX:
		<ul dir="rtl">
		  <li>هر ماژول POX باید تابع launch() داشته باشد</li>
		  <li>این تابع توسط هسته اصلی POX هنگام بارگذاری ماژول فراخوانی می‌شود</li>
		</ul>
	  </li>
	  	<li>2. مکانیزم ثبت:
		<ul dir="rtl">
		  <li>registerNew یک نمونه جدید از کلاس AdaptiveRouting ایجاد می‌کند</li>
		  <li>این نمونه به سیستم event-driven POX متصل می‌شود</li>
		</ul>
	  </li>
	  	<li>3. زمانبندی اجرا:
		<ul dir="rtl">
		  <li>این تابع فقط یکبار هنگام راه‌اندازی کنترلر اجرا می‌شود</li>
		  <li>مسئولیت اصلی آن تنظیم اولیه و فعال‌سازی کنترلر است</li>
		</ul>
	  </li>
	  	<li>4. سادگی طراحی:
		<ul dir="rtl">
		  <li>با وجود سادگی، این تابع نقش حیاتی در معماری POX دارد</li>
		  <li>تمام initializationهای پیچیده در constructor کلاس AdaptiveRouting انجام می‌شود</li>
		</ul>
	  </li>
	  	<li>5. تک نقطه ورود:
		<ul dir="rtl">
		  <li>این تابع به عنوان تنها نقطه ورود خارجی برای ماژول عمل می‌کند</li>
		  <li>تمام Dependencyها و تنظیمات اولیه به صورت کپسوله شده در کلاس اصلی مدیریت می‌شوند</li>
		</ul>
	  </li>
	</ul>
</p>
