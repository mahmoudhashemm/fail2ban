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
