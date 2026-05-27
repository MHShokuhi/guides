<div dir="auto">

# آموزش اشتراک‌گذاری پورت 53 برای استفاده از چند پروژه وابسته به DNS روی یک سرور

اگه می‌خواید چندین سرویس مرتبط با DNS رو یکجا ران کنید (مثلاً همزمان هم [dnstm](https://github.com/net2share/dnstm) و هم [MasterDnsVPN](https://github.com/masterking32/MasterDnsVPN)  و هم [theFeed](https://github.com/sartoopjj/thefeed) یا چیزهای دیگه روی پورت 53)، این آموزش بهتون کمک می‌کنه با [dnsdist](https://www.dnsdist.org/) این کار رو انجام بدید.

نکته ۱: ترجیحا از ufw استفاده کنید و فعالش کنید (قبل فعال کردنش پورت SSH رو یادتون نره باز کنید) قبل از شروع کار چون با iptables یکم مراحل بیشتر میشه و باید رول‌های داخلی رو پاک کنید احتمالاً.

نکته ۲: همه اینها روی Ubuntu 24.04 LTS از شرکت‌های مختلف خارجی تست شدن (از شرکت ایرانی VPS نگیرید برای دردسر کمتر).

نکته ۳: برای هر تونل یا پروژه نیازمند یه ساب‌دامنه جداگانه هستید. اگه مثلاً قصد دارید ۳تا تونل dnstm، یدونه MasterDnsVPN و یکی هم خب thefeed لازم داره پس باید اینطوری کنید توی مثلاً کلودفلر (با ابر خاموش) برای دامنه‌ای که دارید:

| Type | Name | Value |
|------|------|-------|
| A | `ns.example.com` | `YOUR_SERVER_IP` |

حالا این ۳تا (یا هر چندتا خودتون بیشتر کنید) برای dnstm:

| Type | Name | Value |
|------|------|-------|
| NS | `d1.example.com` | `ns.example.com` |
| NS | `d2.example.com` | `ns.example.com` |
| NS | `d3.example.com` | `ns.example.com` |

این برای MasterDnsVPN:

| Type | Name | Value |
|------|------|-------|
| NS | `m.example.com` | `ns.example.com` |

اینم برای theFeed:

| Type | Name | Value |
|------|------|-------|
| NS | `f.example.com` | `ns.example.com` |

کلا راهنمای هر پروژه رو اول بخونید که بدونید چطور نصب و راه‌اندازی کنید و بعد انجامش بدید. اما بریم سراغ آموزش اصلی:

## مرحله ۱: نصب dnstm

نصب dnstm با دستور زیر:

```bash
curl -sSL https://raw.githubusercontent.com/net2share/dnstm/main/install.sh | sudo bash
```

که راهنمای کاملش رو توی ریپو پروژه بخونید و بعد کانفیگ هر تونلی که لازم دارید و قرار دادنش روی حالت روتر چندگانه که بتونید همه چی رو تست کنید. کارتون که تموم شد و همه چیز بدون مشکل کار می‌کرد، برید سراغ مراحل بعد.

## مرحله ۲: شناسایی پورت‌های فعال dnstm
برای اینکه ببینید سرویس‌های dnstm الان دارن روی چه پورت‌های داخلی و با چه ساب‌دامنه‌هایی کار می‌کنن، این فایل رو باز کنید و جزئیاتش رو یه گوشه داشته باشید یا فایل رو دانلود کنید.

```text
/etc/dnstm/config.json
```

اونجا هر تگ تونل (tag) و دامنه (domain) و پورت (port) مشخصه و لازمشون داریم. البته تگ برای این هست که راحت‌تر همه چی رو متوجه بشیم و الزامی نیست. برای مثال یه تیکه اینطوری ممکنه ببینید:

```json
"tag": "storm-eagle",
"enabled": true,
"transport": "dnstt",
"backend": "socks",
"domain": "d1.example.com",
"port": 5310,
"dnstt": {
"mtu": 1232,
"private_key": "/etc/dnstm/tunnels/storm-hawk/server.key"
```

که مشخصات برای هر تونل هست. ما فقط به `storm-eagle` (تگ)، `d1.example.com`  (دامنه) و `5310` (پورت) نیاز داریم. چون فرض هم کردیم ۳تا تونل می‌خوایم درست کنیم، پس اسکریپت پورت‌های `5310`، `5311` و `5312` رو به تونل‌ها اختصاص داده.

## مرحله ۳: حذف روتر dnstm

ما دیگه نیازی به روتر dnstm نداریم و قراره کار رو با dnsdist انجام بدیم و پورت 53 رو بدیم بهش، پس برای آزاد کردنش باید متوقفش کنیم و اجرا شدنش بعد از هربار ریستارت سرور رو هم خاموش کنیم با این دو دستور:

```bash
sudo systemctl stop dnstm-dnsrouter
sudo systemctl disable dnstm-dnsrouter
```

### مرحله ۴: نصب MasterDnsVPN

الان می‌تونیم MasterDnsVPN رو با دستور زیر نصب کنیم:

```bash
sudo bash <(curl -Ls https://raw.githubusercontent.com/masterking32/MasterDnsVPN/main/server_linux_install.sh)
```

که بعد از نصب، موقتاً سرویسش رو متوقف می‌کنیم:

```bash
sudo systemctl stop masterdnsvpn
```

## مرحله ۵: تنظیم MasterDnsVPN

برای مستر فرض می‌کنیم پورت `5350` رو قراره براش استفاده کنیم. پس توی فایل `server_config.toml` که احتمالاً توی فولدر `root` یا هر یوزری که باهاش کار می‌کنید هست، بازش کنید:

```bash
sudo nano server_config.toml
```

این تنظیمات رو پیدا کنید و مقادیرش رو اینطوری تغییر بدید:

```toml
UDP_HOST = "127.0.0.1"
UDP_PORT = 5350
```
حالا دوباره پروژه رو اجرا می‌کنیم:

```bash
sudo systemctl restart masterdnsvpn
```

## مرحله ۶: نصب و تنظیم theFeed

برای نصب theFeed از دستور زیر استفاده کنید:

```bash
sudo bash -c "$(curl -Ls https://raw.githubusercontent.com/sartoopjj/thefeed/main/scripts/install.sh)"
```

خوبی theFeed اینه که وقت نصب می‌تونید آدرس و پورت رو مشخص کنید و فرض می‌کنیم `5351` چیز خوبی هست، پس وقتی ازمون می‌پرسه که آدرس و پورت چی باشه (خودش پیشفرض `0.0.0.0:53` هست که این رو نمی‌خوایم)، اینطوری وارد می‌کنیم:

```text
127.0.0.1:5351
```

البته اگه یادتون رفت یا هرچی، مقدارش رو بعداً توی این فایل می‌تونید تغییر بدید:

```text
/opt/thefeed/data/thefeed.env
```

## مرحله ۷: نصب dnsdist

حالا dnsdist رو برای مدیریت پورت 53 و هدایت هر ابزار به پورت داخلی ابزار مدیریت پورت رو نصب می‌کنیم:

```bash
sudo apt update && sudo apt install dnsdist -y
```

## مرحله ۸: تنظیم dnsdist

خب ما با کمک dnsdist می‌تونیم بگیم هر ساب‌دامنه به کدوم پورت داخلی فرستاده بشه. پس این فایل رو باز کنید:

```bash
sudo nano /etc/dnsdist/dnsdist.conf
```

کل محتویاتش رو پاک کنید و اینا رو بگذارید توش. در نظر داشته باشید که پورت‌ها و ساب‌دامنه‌ها باید مطابق چیزی باشه که از مرحله ۲ پیدا کرده یا از مراحل ۵ و ۶ تنظیم کرده بودیم. اون مقادیر داخل pool و در مقابلش PoolAction رو هم برای اینکه بعداً راحت‌تر باشید، برابر با مقدار tag برای ۳تا (هرچندتا) تونل dnstm قرار بدید که مثلاً در نهایت اینطوری بشه:

```conf
setACL('0.0.0.0/0')

setLocal('0.0.0.0:53')

newServer({address='127.0.0.1:5310', pool='dnstm1'})
newServer({address='127.0.0.1:5311', pool='dnstm2'})
newServer({address='127.0.0.1:5312', pool='dnstm3'})
newServer({address='127.0.0.1:5350', pool='masterdns'})
newServer({address='127.0.0.1:5351', pool='thefeed'})

addAction("d1.example.com.", PoolAction("dnstm1"))
addAction("d2.example.com.", PoolAction("dnstm2"))
addAction("d3.example.com.", PoolAction("dnstm3"))
addAction("m.example.com.", PoolAction("masterdns"))
addAction("f.example.com.", PoolAction("thefeed"))

newServer({address='1.1.1.1', pool='cloudflare'})

addAction(AllRule(), PoolAction("cloudflare"))
```

### نکات مهم تنظیم dnsdist
- مقدار `pool` با `PoolAction` مربوط به هر تونل باید یکی باشه که هدایت درست انجام بشه.
- یعنی در بخش `newServer` پورت‌های داخلی سرویس‌های فعلی رو وارد کنید.
- و بعد در بخش `addAction` ساب‌دامنه مربوط به هر سرویس رو بنویس
پورت `5350` رو برای MasterDnsVPN در نظر گرفته بودیم.
- پورت `5351` هم برای theFeed بود.
- در نهایت اگر تعداد تونل یا ابزار بیشتری دارید، طبق همین الگو، در `addAction` و `newServer` اضافه کنید و اگه کمتر هست خب پاک کنید چیز بی‌استفاده نمونه.


## مرحله ۹: تایید تنظیمات dnsdist

برای تست فایل کانفیگی که نوشتیم برای اینکه ارور نداشته باشه این دستور رو بزنید:

```bash
dnsdist --check-config
```

اگه نوشت:

> Configuration '/etc/dnsdist/dnsdist.conf' OK!

یعنی کار درست انجام شده و تبریک می‌گم، اگه اروری هم دیدید، برید و تصحیحش کنید. در نهایت dnsdist رو فعال و ریستارت می‌کنیم:

```bash
sudo systemctl enable dnsdist
sudo systemctl restart dnsdist
```

## مرحله ۱۰: اطمینان از عدم تداخل
برای اینکه مطمئن بشید فقط dnsdist داره به پورت 53 سرور گوش میده، این دستور رو بزنید:

```bash
sudo ss -tulnp | grep ":53 "
```
باید فقط dnsdist به tcp و udp در حال LISTEN باشه.

## مرحله ۱۱: بررسی پس از آپدیت پروژه‌ها

اگه هرچیزی آپدیت داد، ممکنه تنظیمات خودش رو برگردونه (theFeed این کار رو نمی‌کنه و اوکی هست) و بنابراین بعد از هر آپدیت مراحل ۳ و ۵ رو باید انجام بدید و در نهایت مرحله ۱۰ رو برای چک کردن اینکه همه چیز درست هست.


</div>