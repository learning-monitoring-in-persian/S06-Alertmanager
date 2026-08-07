[فارسی](README-persian.md) | [English](README.md)

# راه‌اندازی Alertmanager

الرت‌منیجر (Alertmanager) وظیفه مدیریت هشدارهایی (Alerts) رو بر عهده داره که از طرف برنامه‌های کلاینت مثل سرور پرومتئوس ارسال میشن. این ابزار کارهایی مثل حذف هشدارهای تکراری (Deduplicating)، دسته‌بندی (Grouping) و ارسال اون‌ها به گیرنده مناسب (مثل ایمیل، اسلک، یا وب‌هوک) رو انجام میده. همچنین کارهایی مثل بی‌صدا کردن (Silencing) و توقف موقت هشدارها (Inhibition) هم توسط Alertmanager مدیریت میشه.

> ### نکته
> با اینکه پرومتئوس رایج‌ترین منبع ارسال الرت هست، اما تنها منبع نیست! سیستم‌های دیگه‌ای مثل **Grafana Loki** (که تو آموزش‌های آینده بهش می‌پردازیم) هم می‌تونن لاگ‌ها رو ارزیابی کنن و مستقیماً برای Alertmanager هشدار بفرستن.

> ### نکته
> اگه قصد دارید **Alertmanager** رو روی ماشینی نصب کنید که از قبل **داکر** داره، یا اگه می‌خواید ابزارهای دیگه‌ای مثل **پرومتئوس** رو کنارش اجرا کنید، شدیداً پیشنهاد می‌کنم از نسخه‌ی **داکری** Alertmanager استفاده کنید. این کار ستاپ شما رو منعطف‌تر می‌کنه و نگهداری (Maintenance) اون در آینده خیلی راحت‌تر میشه.
>
> اما، اگه اون ماشین قراره فقط Alertmanager رو اجرا کنه و اصلاً داکر نداره، بهتره از بار اضافی جلوگیری کنید و نسخه باینری رو مستقیماً نصب کنید.

---

## نصب نسخه‌ی باینری Alertmanager

اول باید نسخه کامپایل شده (باینری) مخصوص لینوکس رو دانلود کنیم:

```bash
VERSION=$(curl -s https://api.github.com/repos/prometheus/alertmanager/releases/latest | grep '"tag_name"' | cut -d'"' -f4)
wget -O alertmanager.tar.gz https://github.com/prometheus/alertmanager/releases/download/${VERSION}/alertmanager-${VERSION#v}.linux-amd64.tar.gz
tar -xvf alertmanager.tar.gz
```

شما می‌تونید یک فایل کانفیگ پیش‌فرض با اسم `alertmanager.yml` رو داخل پوشه استخراج شده پیدا کنید.
اگر ماشین شما ری‌استارت بشه، پروسه متوقف میشه. برای اینکه Alertmanager رو به عنوان یک سرویس پس‌زمینه اجرا کنیم، مراحل زیر رو دنبال کنید.

## راه‌اندازی Alertmanager به‌صورت سرویس systemd

### ۱) ساخت کاربر و دایرکتوری‌ها

برای امنیت بیشتر، یک یوزر سیستمی اختصاصی بسازید:
```bash
sudo useradd --no-create-home --shell /usr/sbin/nologin alertmanager
sudo mkdir -p /etc/alertmanager /var/lib/alertmanager
```

### ۲) انتقال باینری‌ها

فایل‌های باینری رو به مسیر مناسب در سیستم منتقل کنید:
```bash
sudo mv alertmanager-*/alertmanager /usr/local/bin/
sudo mv alertmanager-*/amtool /usr/local/bin/
rm -rf alertmanager*
```

### ۳) انتقال فایل کانفیگ

فایل `/etc/alertmanager/alertmanager.yml` را با یک کانفیگ پایه بسازید (این کانفیگ به عنوان مثال، الرت‌ها را به یک وب‌هوک لوکال می‌فرستد):

```yaml
route:
  group_by: ['alertname']
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 1h
  receiver: 'web.hook'

receivers:
  - name: 'web.hook'
    webhook_configs:
      - url: 'http://127.0.0.1:5001/'
        send_resolved: true
        username: Infrustructure Alerting
```

> ### نکته
> ابزار Alertmanager پارامترها و تنظیمات بسیار گسترده‌ای برای گیرنده‌های مختلف (مثل اسلک، ایمیل، وب‌هوک و PagerDuty)، مسیردهی‌های پیچیده (Routing trees) و قوانین توقف هشدار داره. شما می‌تونید لیست کامل پارامترهای کانفیگ و توضیحاتشون رو در [مستندات رسمی کانفیگ Alertmanager](https://prometheus.io/docs/alerting/latest/configuration/) ببینید.

```bash
sudo chown -R alertmanager:alertmanager /etc/alertmanager /var/lib/alertmanager
```

### ۴) ساخت فایل سرویس

فایل `/etc/systemd/system/alertmanager.service` را با محتوای زیر بسازید:

یک فایل جدید در مسیر `/etc/systemd/system/alertmanager.service` ایجاد کنید:
```ini
[Unit]
Description=Alertmanager
Wants=network-online.target
After=network-online.target

[Service]
User=alertmanager
Group=alertmanager
Type=simple
ExecStart=/usr/local/bin/alertmanager \
  --config.file=/etc/alertmanager/alertmanager.yml \
  --storage.path=/var/lib/alertmanager

Restart=always

[Install]
WantedBy=multi-user.target
```

در نهایت سرویس systemd رو ری‌لود کرده و Alertmanager رو استارت کنید:
```bash
sudo systemctl daemon-reload
sudo systemctl enable alertmanager
sudo systemctl start alertmanager
```

---

## راه‌اندازی Alertmanager با Docker Compose (پیشنهادی)

اگر ترجیح میدید از داکر استفاده کنید، اجرای Alertmanager بسیار ساده‌ست. فایل `docker-compose.yml` رو با محتوای زیر بسازین:

```yaml
services:
  alertmanager:
    image: prom/alertmanager:latest
    container_name: alertmanager
    restart: unless-stopped
    ports:
      - "9093:9093"
    volumes:
      - ./alertmanager.yml:/etc/alertmanager/alertmanager.yml
```

مطمئن بشید که فایل کانفیگ `alertmanager.yml` رو کنار همین فایل کامپوز دارید، و بعد دستور زیر رو اجرا کنید:
```bash
docker compose up -d
```

---

## کانفیگ Prometheus برای ارسال الرت‌ها

به صورت پیش‌فرض، پرومتئوس نمی‌دونه هشدارهایی که فایر میکنه رو باید کجا بفرسته. شما باید بهش تنظیمات Alertmanager رو بدید.

فایل `prometheus.yml` رو باز کنید و بلاک `alerting` رو بهش اضافه کنید:

```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

# این بلاک رو برای متصل شدن به الرت منیجر اضافه کنید
alerting:
  alertmanagers:
    - static_configs:
        - targets:
          # اگه الرت‌منیجر روی یه ماشین دیگه‌ست، آی‌پی اون رو جایگزین کنید
          - localhost:9093

rule_files:
  - "rules/*.yml"

scrape_configs:
  - job_name: "prometheus"
    static_configs:
      - targets: ["localhost:9090"]
```

پرومتئوس رو ری‌استارت کنید تا تنظیمات اعمال بشه.

---

## یکپارچه‌سازی با گرافانا

گرافانا می‌تونه به Alertmanager به عنوان یک Data source متصل بشه. این قابلیت بهتون اجازه میده تا مستقیماً از داخل محیط کاربری گرافانا، الرت‌ها و Silences خودتون رو ببینید و مدیریتشون کنید.

### روش گرافیکی (از طریق پنل)
۱. گرافانا رو باز کنید.
۲. به بخش **Connections -> Data sources** برید.
۳. روی **Add data source** کلیک کنید و **Alertmanager** رو انتخاب کنید.
۴. در فیلد HTTP URL، آدرس `http://localhost:9093` (یا آی‌پی سرور الرت‌منیجر) رو وارد کنید.
۵. اسکرول کنید پایین و روی **Save & test** کلیک کنید.

### روش اتوماتیک با فایل (Provisioning)
اگه گرافانا رو از طریق فایل‌ها مدیریت می‌کنید (مثلاً توی ستاپ داکر کمپوز)، می‌تونید دیتاسورس Alertmanager رو به صورت اتوماتیک Provision کنید.

یک فایل به اسم `alertmanager.yaml` داخل پوشه پروویژنینگ گرافانا (مثلاً در مسیر `./grafana/provisioning/datasources/`) ایجاد کنید:

```yaml
apiVersion: 1

datasources:
  - name: Alertmanager
    type: alertmanager
    access: proxy
    url: http://alertmanager:9093 # ip_address:port :اگه توی یک شبکه داکر نیستن، اینطور استفاده کنید
    isDefault: false
    jsonData:
      implementation: prometheus
```

گرافانا رو ری‌استارت کنید تا دیتاسورس جدید به صورت اتوماتیک اضافه بشه.
