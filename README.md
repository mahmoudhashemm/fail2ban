أساسيات حماية Odoo بالاعتماد على Cybrosys

Fail2Ban موجود في مستودعات Ubuntu، ونثبّته بـ:
```
sudo apt update
sudo apt install fail2ban
```

Infinitive Host
+15
Cybrosys Technologies
+15
Cybrosys Technologies
+15

الملف jail.conf يُفترض أن لا نعدّله مباشرة، بل ننسخه إلى jail.local للتخصيص.

تكوين DEFAULT:

bantime (مدة الحظر)

findtime (الفترة الزمنية لاحتساب المحاولات)

maxretry (عدد المحاولات المسموح بها)

Infinitive Host
+3
GitHub
+3
Odoo
+3
GitHub
+3
Cybrosys Technologies
+3
Odoo
+3

تفعيل jails خاصة، مثل nginx-http-auth لمراقبة عناوين الوصول، أو sshd لحماية SSH.

Wikipedia
+15
Cybrosys Technologies
+15
GitHub
+15

إعداداتك النهائية (Odoo + SSH + Docker)
1) تأكد من وجود سلسلة DOCKER-USER (ضرورية لمنع الدخول إلى الحاويات)
```
sudo iptables -nL DOCKER-USER >/dev/null 2>&1 || {
  sudo iptables -N DOCKER-USER
  sudo iptables -A DOCKER-USER -j RETURN
}

```
3) فلتر Odoo مخصص (odoo-login.conf)
```
sudo tee /etc/fail2ban/filter.d/odoo-login.conf >/dev/null <<'EOF'
[Definition]
datepattern = ^YYYY-mm-dd HH:MM:SS,ms
failregex = ^.*\sodoo\.addons\.base\.models\.res_users:\s+Login failed for db:\S+\s+login:\S+\s+from <HOST>\s*$
ignoreregex =
EOF
```
5) ملف Jail جامع (jail.local)
```
sudo tee /etc/fail2ban/jail.local >/dev/null <<'EOF'
[DEFAULT]
bantime  = 1h
findtime = 10m
maxretry = 3
banaction = iptables-multiport
chain     = DOCKER-USER
ignoreip  = 127.0.0.1/8 ::1   # أضف IPك الوثوق هنا

[odoo]
enabled  = true
port     = 8069,8072
filter   = odoo-login
logpath  = /opt/hegaze/etc/odoo-server.log
backend  = auto
action   = iptables-multiport[name=odoo, port="8069,8072", chain="DOCKER-USER"]

[sshd]
enabled  = true
port     = ssh
filter   = sshd
logpath  = /var/log/auth.log
backend  = auto
EOF
```


هذا يتبع المبدأ: لا تغيّر jail.conf مباشرة، بل اصنع نسخة مخصصة فيه، تمامًا كما ورد في مقالة Cybrosys.

4) تفعيل Fail2Ban
```
sudo systemctl restart fail2ban
sudo systemctl enable fail2ban
```

5) تحقق من الجيلز والحظر
```
sudo fail2ban-client status          # تأكد من وجود odoo و sshd
sudo fail2ban-client status odoo
sudo iptables -S DOCKER-USER         # تأكد من إضافة قواعد f2b-odoo
sudo fail2ban-client status sshd
```
التحقق العملي

جرّب 3 محاولات فاشلة على Odoo → يجب أن يتم حظر IPك ويظهر في status odoo.

جرّب بعدها SSH محاولات فاشلة → يجب حظر أيضًا.


```
sudo fail2ban-client get sshd banip
sudo fail2ban-client get sshd banned
sudo fail2ban-client get sshd bantime
sudo fail2ban-client get sshd timeleft 192.168.1.10
```

```
grep "Ban" /var/log/fail2ban.log
```
```
grep "invalid use" /var/log/auth.log
```

لو عاوز تعمل **unban (إلغاء الحظر)** لأي IP من خلال **Fail2ban**، تقدر تستخدم أحد الطريقتين:

### 1. باستخدام **fail2ban-client**

```bash
sudo fail2ban-client set sshd unbanip 192.168.1.10
```

* `sshd` = اسم الـ jail اللي معمول عليه الحظر.
* `192.168.1.10` = الـ IP اللي عاوز تفك حظره.

---

### 2. باستخدام **iptables / nftables** (لو Fail2ban بيستخدم iptables)

ممكن تفك الحظر يدويًا:

```bash
sudo iptables -D f2b-sshd -s 192.168.1.10 -j REJECT
```

أو لو شغال بـ nftables:

```bash
sudo nft delete element inet filter f2b-sshd { 192.168.1.10 }
```

---

✅ أنسب وأبسط طريقة:
استخدم أمر **fail2ban-client** لأنه بيضمن إزالة الـ IP من قائمة الحظر الخاصة بـ jail بدون لعب يدوي في iptables.

تحب أديك أمر يطلعلك كل الـ IPs المحظورة دلوقتي عشان تختار منهم تعملهم unban؟




كل الـ jails مع بعض (بـ loop)

لو عاوز قائمة بكل الـ IPs المحظورة من كل الـ jails مرة واحدة:

```
for jail in $(sudo fail2ban-client status | grep "Jail list:" | cut -d: -f2 | tr ',' ' '); do
    echo "=== $jail ==="
    sudo fail2ban-client status $jail | grep "Banned IP list"
done
```

لو عاوز تبحث عن IP محدد في كل الـ jails مرة واحدة

تقدر تستخدم loop بسيط:
```

for jail in $(sudo fail2ban-client status | grep "Jail list:" | cut -d: -f2 | tr ',' ' '); do
    echo "Checking jail: $jail"
    sudo fail2ban-client status $jail | grep "Banned IP list" | grep -q "192.168.1.10" && echo ">>> Found in $jail"
done
```

ده هيعرضلك في أنهي jail الـ IP محظور. ✅




تمام ✅ أهو الملخص كامل بالأوامر بالترتيب + الإضافة اللي طلبتها 👇

---

## 🔹 أوامر iptables و Fail2Ban (مرجع سريع)

### 1️⃣ عرض القواعد

```bash
sudo iptables -L DOCKER-USER -n -v --line-numbers
sudo iptables -S
```

---

### 2️⃣ تنظيف (Flush) القواعد الخاصة بـ Fail2Ban

```bash
sudo iptables -F f2b-sshd
sudo iptables -F f2b-odoo
```

---

### 3️⃣ إعادة تحميل Fail2Ban

```bash
sudo fail2ban-client reload
```

---

### 4️⃣ إضافة قاعدة سماح لموقع الضرائب (ETA IP)

```bash
sudo iptables -I DOCKER-USER 1 -o eth0 -p tcp -d 164.160.67.14 --dport 443 -j ACCEPT
```

* دي بتسمح بخروج HTTPS (بورت 443) من السيرفر عبر `eth0` إلى الـ IP `164.160.67.14`.

---

### 5️⃣ حفظ القواعد

```bash
sudo iptables-save > /etc/iptables/rules.v4
```

---

### 6️⃣ استرجاع القواعد يدويًا

```bash
sudo iptables-restore < /etc/iptables/rules.v4
```

---

تحب أضيفلك برضه نسخة للأمر ده لكن تسمح لمجموعة **كل IPs بتاعة `id.eta.gov.eg`** أوتوماتيك (loop بالـ bash) بدل ما تضيفهم واحد واحد؟




مظبوط 👍 عندك حق، الأفضل تنظّف القواعد المكررة عشان الجدول يبقى أوضح وأسهل في الإدارة.

---

### 🔍 القواعد المكررة عندك:

* **Fail2Ban SSH (dports 22)**
  موجودة في السطور: **3, 4, 5, 6, 7, 9, 10, 11, 12**
  كلها بنفس الشكل → تقدر تسيب واحدة بس وتلغي الباقي.

* **Fail2Ban Odoo (dports 8069, 8072)**
  موجودة في السطرين: **8 و 13**
  نفس الوظيفة → واحدة كفاية.

---

### ✅ طريقة الحذف

بما إن الأرقام بتتغير بعد كل حذف، لازم تمسحهم من **تحت لفوق**.
مثلاً تبدأ من السطر 13 وبعدين 12 … لحد ما تسيب نسخة واحدة فقط من كل Rule.

مثال:

```bash
sudo iptables -D DOCKER-USER 13
sudo iptables -D DOCKER-USER 12
sudo iptables -D DOCKER-USER 11
sudo iptables -D DOCKER-USER 10
sudo iptables -D DOCKER-USER 9
sudo iptables -D DOCKER-USER 7
sudo iptables -D DOCKER-USER 6
sudo iptables -D DOCKER-USER 5
sudo iptables -D DOCKER-USER 4
sudo iptables -D DOCKER-USER 3
```

(كل مرة اعمل `iptables -L DOCKER-USER -n -v --line-numbers` قبل الحذف عشان تتأكد من الرقم الجديد).

---

### 💡 اقتراح عملي

لو عايز تختصر:

* امسح كل القواعد اللي تخص `f2b-sshd` و `f2b-odoo`.
* بعدين سيب Fail2Ban هو يعيد إنشائها بشكل مرتب لما يعمل reload:

  ```bash
  sudo systemctl restart fail2ban
  ```

---

تحب أديك أوامر جاهزة تعمل **إعادة تشغيل Fail2Ban** بحيث يعيد إنشاء القواعد بشكل نظيف من غير التكرار؟
