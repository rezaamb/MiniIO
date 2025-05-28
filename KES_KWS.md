## روی نود اصلی MinIO میایم KES را نصب میکنیم 
```bash
wget https://github.com/minio/kes/releases/latest/download/kes-linux-amd64 -O /usr/local/bin/kes
chmod +x /usr/local/bin/kes
```
## گواهی valid wildcart از certum ساخه شد

در مسیر /etc/kes/ بنام های csdiran.crt و csdiran.key
گواهی های client.crt و client.key نیز در همین مسیرند

### برای تولید FingerPrint 
```bash
kes identity of /etc/kes/client.crt
```

### کانفیگ فایل kes در مسیر /etc/kes/config.yml
```bash
vim config.yml
```

### نمونه کانفیگ فایل استفاده شده در پروژه :
```bash
address: 0.0.0.0:7373

admin:
  identity: ae46be9c089acf9b1a7c892843257e3d2b685492a1cbb3cc81988abee884473a

tls:
  key: /etc/kes/csdiran.key
  cert: /etc/kes/csdiran.crt


policy:
  minio:
    allow:
    - /v1/key/create/*   # You can replace these wildcard '*' with a string prefix to restrict key names
    - /v1/key/generate/* # e.g. '/minio-'
    - /v1/key/decrypt/*
    - /v1/key/bulk/decrypt
    - /v1/key/list
    - /v1/status
    - /v1/metrics
    - /v1/log/audit
    - /v1/log/error


keystore:
  fs:
    path: /etc/kes/keys

```
##تعریف سرویس systemd برای KES

## فایل /etc/systemd/system/kes.service:

```bash
[Unit]
Description=MinIO KES (Key Encryption Service)
After=network.target

[Service]
ExecStart=/usr/local/bin/kes server --config /etc/kes/config.yml
Restart=always
Environment="MINIO_KMS_KES_ENDPOINT=https://Osskms.csdi.ir:7373"
Environment="MINIO_KMS_KES_CERT_FILE=/etc/kes/client.crt"
Environment="MINIO_KMS_KES_KEY_FILE=/etc/kes/client.key"
Environment="MINIO_KMS_KES_KEY_NAME=csd-minio-sse-kms-key"
RestartSec=5
User=root
Group=root


[Install]
WantedBy=multi-user.target
```

## فعال‌سازی سرویس:
```bash
systemctl daemon-reexec
systemctl enable kes
systemctl start kes
```

## پیکربندی MinIO برای اتصال به KES
## در فایل /etc/default/minio یا بخش Environment از systemd:

```bash
MINIO_VOLUMES="http://MinIO{1...4}.CSDI.PriDMZ:9000/mnt/data"
MINIO_OPTS="--console-address :9001"
MINIO_ROOT_USER=consoleAdmin
MINIO_ROOT_PASSWORD=minIoAdmInCSDI

MINIO_KMS_KES_ENDPOINT=https://Osskms.csdiran.ir:7373
MINIO_KMS_KES_CERT_FILE=/etc/kes/client.crt
MINIO_KMS_KES_KEY_FILE=/etc/kes/client.key
MINIO_KMS_KES_KEY_NAME=csd-minio-sse-kms-key
```

## راه‌اندازی مجدد MinIO

```bash
systemctl restart minio
```
## حالا برای اینکه همه ی دیتا توی 4 نود sync باشد میایم فایل updateMinio رو run میکنیم:
```bash
./updateMinio.sh
```

## اگر میخوایم بصورت cli کار کنیم باید این export هارو توی session خودمون بزنیم

```bash
export KES_SERVER=https://127.0.0.1:7373        
export KES_CLIENT_CERT="./client.crt"       
export KES_CLIENT_KEY="./client.key"
export MINIO_ROOT_USER="consoleAdmin"
export MINIO_ROOT_PASSWORD="minIoAdmInCSDI"
export MINIO_KMS_KES_ENDPOINT="https://Osskms.csdiran.ir:7373"
export MINIO_KMS_KES_KEY_NAME=csd-minio-sse-kms-key
export MINIO_KMS_KES_ENDPOINT=https://Osskms.csdiran.ir:7373

```
## برای sync data مخصوصا keys بین 4 نود میایم برای نودهای دیگر توی نود اصلی cronjob تعریف میکنیم:
```bash
crontab -e
```
```bash
*/5 * * * * /usr/bin/rsync -avz /etc/kes/keys/ -e ssh root@172.16.5.226:/etc/kes/keys/
*/5 * * * * /usr/bin/rsync -avz /etc/kes/keys/ -e ssh root@172.16.5.227:/etc/kes/keys/
*/5 * * * * /usr/bin/rsync -avz /etc/kes/keys/ -e ssh root@172.16.5.228:/etc/kes/keys/
```
### ایجاد یک کلید رمزنگاری جدید در سرویس KES (Key Encryption Service) با شناسه‌ی my-key-1
```bash
kes key create <KEY-NAME>
```




### برای نشان دادن لیست کلیدها
```bash
ls -lha /etc/kes/keys
```

### برای مشاهده کلید ساخته شده
```bash
cat /etc/kes/keys/<KEY-NAME>
```

### دریافت یک Data Encryption Key (DEK) از سرویس KES، با استفاده از کلید اصلی (KEK) به نام my-key-1
```bash
kes key dek <KEY-NAME>
```



#### برای اتصال به MinIO :
```bash
mc alias set local http://127.0.0.1:9000 consoleAdmin minIoAdmInCSDI
```
### ساخت باکت 
```bash
mc mb local/<BUCKET-NAME>
```
### ساخت فایل ساده برای تست
```bash
echo "this is a test for Encrypting" > sample.txt 
```
### مشاهده فایل تستی ساخته شده
```bash
cat sample.txt
```
### کپی فایل به داخل باکت ساخته شده
```bash
mc cp sample.txt local/<BUCKET-NAME>
```
### لیست کردن محتوای داخل bucket
```bash
mc ls local/<BUCKET-NAME>
```
### بررسی اطلاعات دقیق فایل
```bash
mc stat local/<BUCKET-NAME>/sample.txt
```


---
### فعال‌سازی رمزنگاری روی یک bucket
```bash
mc encrypt set sse-kms "<KEY-NAME>" local/<BUCKET-NAME>
```
### بررسی وضعیت رمزنگاری یک bucket	
```bash
mc encrypt info local/<BUCKET-NAME>
```
---


# تست رمز نگاری واقعی
```bash
mc cp sample.txt local/<BUCKET-NAME>/sample_encrypt.txt
```
```bash
mc ls local/<BUCKET-NAME>
```
```bash
mc stat local/<BUCKET-NAME>/sample_encrypt.txt
```
# برای کار روی محیط UI میاریم وارد MinIO میشیم
```bash
Buckets > Summary > Encryption (Disabled)
```

## برای enable کردن این قسمت 
```bash
Encryption Type > SSE-KMS > KMS KEY ID 
```
## برای ساخت key جدید توی همین قسمت روی Add Key کلیک میکنیم


## در ادامه چند command ضروری برای CLI :
### حذف باکت خالی 
```bash
mc rb <ALIAS>/<BUCKET_NAME> 
```

###  حذف باکت به‌همراه محتوای آن 
```bash
mc rb --force <ALIAS>/<BUCKET_NAME>
```

### حذف کلید رمزنگاری در KES 
```bash
kes key rm <KEY_NAME>
```

### لیست کلیدهای موجود 
```bash
kes key list
mc admin kms key list myminio
```

### حذف باکت به‌همراه تمام محتویاتش 
```bash
mc rb --force <alias>/<bucket-name>
```

### لیست آبجکت های درون باکت 
```bash
mc ls --recursive <alias>/<bucket-name>
```


### حذف آبجکت های درون باکت 
```bash
mc rm --recursive --force <alias>/<bucket-name>
```

###حذف یوزر محلی (Local user) در MinIO با mc 
```bash
mc admin user remove <alias> <username>
```

### لیست همه ی alias ها 
```bash
mc alias list
mc ls <alias>/<bucket-name>
```

