# نصب MinIO با 4 نود و فعال‌سازی Self-Heal خودکار در systemd

این راهنما نحوه‌ی نصب و پیکربندی MinIO به‌صورت توزیع‌شده (Distributed Mode) با 4 نود مستقل را به همراه فعال‌سازی دستور خودکار heal در زمان راه‌اندازی سرویس MinIO، ارائه می‌دهد.

---

## پیش‌نیازها
- 4 سرور با IP یا DNS مشخص (مثال: `MinIO1.CSDI.PriDMZ`, ..., `MinIO4.CSDI.PriDMZ`)
- دایرکتوری داده‌ی مشترک در هر سرور: `/mnt/data`
- فایل باینری MinIO در مسیر: `/usr/local/bin/minio`
- یوزر `minio-user` با دسترسی لازم
- Systemd برای مدیریت سرویس

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
ExecStartPost=/bin/bash -c 'sleep 5 && HOME=/tmp /usr/local/bin/mc alias set local http://localhost:9000 minioadmin minioadmin && HOME=/tmp /usr/local/bin/mc admin heal -r local'

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
mc alias set local http://localhost:9000 minioadmin minioadmin
mc admin info local
mc admin heal info local
```

---

## 6. نکته پایانی درباره Self-Heal
- MinIO در حالت توزیع‌شده قابلیت auto-heal دارد، اما فقط هنگام read/write.
- اجرای `mc admin heal` بعد از بازگشت نود خاموش‌شده، ضروری است اگر می‌خواهید اطمینان 100٪ داشته باشید.
- در این راهنما این دستور به‌صورت اتوماتیک بعد از هر start شدن MinIO اجرا می‌شود.

---

برای اطمینان بیشتر می‌توانید لاگ‌های مربوط به heal را به `/var/log/minio-heal.log` هدایت کنید یا آن را در یک cron job جداگانه زمان‌بندی کنید.
