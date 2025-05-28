# نصب MinIO با 4 نود و فعال‌سازی Self-Heal خودکار در systemd

این راهنما نحوه‌ی نصب و پیکربندی MinIO به‌صورت توزیع‌شده (Distributed Mode) با 4 نود مستقل را به همراه فعال‌سازی دستور خودکار heal در زمان راه‌اندازی سرویس MinIO، ارائه می‌دهد.

4 سرور داریم :
172.16.5.225
172.16.5.226
172.16.5.227
172.16.5.228

---

## پیش‌نیازها
- 4 سرور با IP یا DNS مشخص (مثال: `MinIO1.CSDI.PriDMZ`, ..., `MinIO4.CSDI.PriDMZ`)
- دایرکتوری داده‌ی مشترک در هر سرور: `/mnt/data`
- یوزر `minio-user` با دسترسی اجرا و خواندن/نوشتن در `/mnt/data`
- Systemd برای مدیریت سرویس
- دسترسی root برای نصب ابزارها

---

## 0. نصب MinIO و mc

### نصب باینری MinIO:
```bash
wget https://dl.min.io/server/minio/release/linux-amd64/minio -O /usr/local/bin/minio
chmod +x /usr/local/bin/minio
```

### ایجاد یوزر مخصوص اجرا:
```bash
sudo useradd -r -s /sbin/nologin minio-user
sudo mkdir -p /mnt/data
sudo chown -R minio-user:minio-user /mnt/data
```

### نصب mc (MinIO Client):
```bash
wget https://dl.min.io/client/mc/release/linux-amd64/mc -O /usr/local/bin/mc
chmod +x /usr/local/bin/mc
```

---

## 1. تنظیم فایل `/etc/minio/minio.conf` روی هر 4 نود

```ini
MINIO_VOLUMES="http://MinIO{1...4}.CSDI.PriDMZ:9000/mnt/data"
MINIO_OPTS="--console-address :9001"
MINIO_ROOT_USER=minioadmin
MINIO_ROOT_PASSWORD=minioadmin
```

---

## 2. ساخت فایل سرویس Systemd: `/etc/systemd/system/minio.service`

```ini
[Unit]
Description=MinIO
Wants=network-online.target
After=network-online.target

[Service]
User=minio-user
Group=minio-user
ProtectProc=invisible
EnvironmentFile=/etc/minio/minio.conf
Restart=always
LimitNOFILE=65536
ExecStart=/usr/local/bin/minio server $MINIO_VOLUMES $MINIO_OPTS
#ExecStartPost=/bin/bash -c 'sleep 5 && HOME=/tmp /usr/local/bin/mc alias set local http://localhost:9000 consoleAdmin minIoAdmInCSDI && HOME=/tmp /usr/local/bin/mc admin heal -r local'

[Install]
WantedBy=multi-user.target
```

> نکته: `sleep 5` برای اطمینان از بالا آمدن کامل MinIO قبل از اجرای دستور heal استفاده شده است.

---

## 3. فعال‌سازی و اجرای سرویس

```bash
sudo systemctl daemon-reload
sudo systemctl enable minio
sudo systemctl start minio
```

---

## 4. بررسی وضعیت

```bash
sudo systemctl status minio
journalctl -u minio -n 50
```

---

## 5. بررسی نهایی و تست

در یکی از نودها:

```bash
mc alias set local http://localhost:9000 consoleAdmin minIoAdmInCSDI
mc admin info local
mc admin heal info local
```

---

## 6. نکته پایانی درباره Self-Heal
- MinIO در حالت توزیع‌شده قابلیت auto-heal دارد، اما فقط هنگام read/write.
- اجرای `mc admin heal` بعد از بازگشت نود خاموش‌شده، ضروری است اگر می‌خواهید اطمینان 100٪ داشته باشید.
- در این راهنما این دستور به‌صورت اتوماتیک بعد از هر start شدن MinIO اجرا می‌شود.

---


## 7. کپی کردن کلیدهای عمومی هر نود اصلی بر روی نودهای دیگر

```bash
cat ~/.ssh/id_rsa.pub
```
---
سپس شناسه را در مسیر root/.ssh/authorized_keys/ روی هر 3 نود قرار میدهیم


---

## 8.سپس تنظیمات ldaps را در مسیر زیر پیاده میکنیم:

```bash
vim /etc/minio/config/iam/ldap.json
```
و سپس :

```bash
{
  "identity_ldap": {
    "server_addr": "pridmzdc-srv.CSDI.PriDMZ:636",
    "username_format": "CN=%s,CN=Users,DC=CSDI,DC=PriDMZ",
    "user_dn_search_base_dn": "DC=CSDI,DC=PriDMZ",
    "user_dn_search_filter": "(sAMAccountName=%s)",
    "bind_dn": "CN=MinioLDAPUser,OU=Minio,OU=Service Users,DC=CSDI,DC=PriDMZ",
    "bind_password": "Mos@1404!2025Sph",
    "tls_skip_verify": true,
    "server_insecure": false,
    "sts_expiry": "1h",
    "group_search_base_dn": "DC=CSDI,DC=PriDMZ",
    "group_search_filter": "(&(objectClass=group)(member=%d))"
  }
}
```
---
روی هر 4 نود پیاده سازی میکنیم.

---

## 9. ساخت فایل updateminio.sh برای اجرای فایل های اجرایی از یک نود روی نود های دیگر
```bash
#!/bin/bash
/usr/bin/rsync -avz /etc/minio/ -e ssh root@172.16.5.226:/etc/minio
/usr/bin/rsync -avz /etc/ssl/certs/*.* -e ssh root@172.16.5.226:/etc/ssl/certs/
/usr/bin/rsync -avz /etc/systemd/system/minio.service -e ssh root@172.16.5.226:/etc/systemd/system/

/usr/bin/rsync -avz /etc/minio/ -e ssh root@172.16.5.227:/etc/minio
/usr/bin/rsync -avz /etc/ssl/certs/*.* -e ssh root@172.16.5.227:/etc/ssl/certs/
/usr/bin/rsync -avz /etc/systemd/system/minio.service -e ssh root@172.16.5.227:/etc/systemd/system/

/usr/bin/rsync -avz /etc/ssl/certs/*.* -e ssh root@172.16.5.228:/etc/ssl/certs/
/usr/bin/rsync -avz /etc/minio/ -e ssh root@172.16.5.228:/etc/minio
/usr/bin/rsync -avz /etc/systemd/system/minio.service -e ssh root@172.16.5.228:/etc/systemd/system/

/usr/bin/systemctl  daemon-reload
/usr/bin/systemctl -H root@172.16.5.227 daemon-reload
/usr/bin/systemctl -H root@172.16.5.228 daemon-reload
/usr/bin/systemctl -H root@172.16.5.226 daemon-reload

/usr/bin/systemctl  restart minio
/usr/bin/systemctl -H root@172.16.5.227 restart minio
/usr/bin/systemctl -H root@172.16.5.228 restart minio
/usr/bin/systemctl -H root@172.16.5.226 restart minio


/usr/bin/sleep 3
/usr/local/bin/mc alias set local http://localhost:9000 consoleAdmin minIoAdmInCSDI
ssh root@172.16.5.226 "/usr/local/bin/mc alias set local http://localhost:9000 consoleAdmin minIoAdmInCSDI"
ssh root@172.16.5.227 "/usr/local/bin/mc alias set local http://localhost:9000 consoleAdmin minIoAdmInCSDI"
ssh root@172.16.5.228 "/usr/local/bin/mc alias set local http://localhost:9000 consoleAdmin minIoAdmInCSDI"

/usr/bin/systemctl  status minio

```


---

سپس به فایل updateminio.sh دسترسی های مورد نظر را داده و آنرا اجرا میکنیم:
```bash
chmod +x updateminio.sh
./updateminio.sh
```
---


## 10. ست کردن Alias 
```bash
mc alias set local http://localhost:9000 consoleAdmin minIoAdmInCSDI
```
---
برای مشاهده لیست alias ها

```bash
mc alias list
``` 
---
---
## 11.دیدن لیست user policy ها 
```bash
mc admin policy list local
``` 
---
شبیه به چنین چیزی رو نمایش میدهد: 
```bash
consoleAdmin
diagnostics
readonly
readwrite
writeonly
```

---

## 12. دادن Access Group به یوزر policy ها
---
Access group to consoleAdmin:
```bash
mc idp ldap policy attach local consoleAdmin --group "CN=MinioConsoleAdmin,OU=Minio,OU=Service Users,DC=CSDI,DC=PriDMZ"
```
Access group ReadWrite:
```bash
mc idp ldap policy attach local readwrite --group "CN=MinioReadWrite,OU=Minio,OU=Service Users,DC=CSDI,DC=PriDMZ"
```
Access group ReadOnly:
```bash
mc idp ldap policy attach local readonly --group "CN=MinioReadOnly,OU=Minio,OU=Service Users,DC=CSDI,DC=PriDMZ"
```

---
