Version Check - بررسی نسخه‌ها
اول، نسخه تمام کامپوننت‌هایی که Zabbix بهشون وابسته هست رو چک می‌کنیم تا مطمئن بشیم دقیقاً با ماتریس سازگاری رسمی Zabbix 7.0 مطابقت دارند.
```bash
php-fpm -v
psql -V
```
Backup of Zabbix Configuration Files - بکاپ فایل‌های کانفیگ زبیکس
قبل از ادامه کار، از تمام فایل‌های تنظیمات Zabbix سرور و پروکسی بکاپ می‌گیریم.
```bash
sudo cp -R /etc/zabbix/ ~/zabbix_backup/etc_zabbix
sudo cp -R /usr/lib/zabbix/alertscripts/ ~/zabbix_backup/alertscripts
sudo cp -R /usr/lib/zabbix/externalscripts/ ~/zabbix_backup/externalscripts
sudo cp -R /usr/share/zabbix/ ~/zabbix_backup/share_zabbix
sudo cp /etc/apache2/apache2.conf ~/zabbix_backup/apache2.conf
sudo cp /etc/apache2/sites-available/zabbix.conf ~/zabbix_backup/zabbix.conf

```
متوقف کردن سرویس‌ها
اول سرویس Zabbix سرور و وب سرور رو متوقف می‌کنیم تا هیچ نوشتنی به دیتابیس صورت نگیره.

```bash
sudo systemctl stop zabbix-server
sudo systemctl stop apache2
```
بروزرسانی PHP
زبیکس ۷ به نسخه PHP بین ۸.۰ تا ۸.۳ نیاز داره. الان ۸.۰ داریم ولی باز بروزرسانی انجام می‌دیم.
با تغییر ماژول dnf در سطح سیستم عامل بروزرسانی می‌کنیم. نسخه قبل از آخرین (مثلاً ۸.۲) پایدارتره و نسخه آخر ممکنه باگ داشته باشه.
```bash
sudo apt install software-properties-common
sudo add-apt-repository ppa:ondrej/php -y
sudo apt update
sudo apt install php8.2 php8.2-fpm php8.2-cli php8.2-mysql php8.2-pgsql php8.2-xml php8.2-mbstring php8.2-bcmath php8.2-curl
sudo systemctl restart php8.2-fpm
```
بروزرسانی دیتابیس
ابتدا سرویس PostgreSQL فعلی رو متوقف می‌کنیم:

```bash
systemctl stop postgresql
```
در Ubuntu برای نصب PostgreSQL 16 ابتدا باید مخزن PostgreSQL را اضافه کنی:
```bash
sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list'
wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add -
sudo apt update
sudo apt install postgresql-16
```


نصب TimescaleDB
حالا TimescaleDB رو نصب می‌کنیم، ولی حتما باید همون نسخه قبلی باشه، چون در غیر اینصورت ارتقا کار نمی‌کنه.اول باید نسخه TimescaleDB که روی PostgreSQL 13 داشتی رو چک کنی:
```bash
dpkg -l | grep timescaledb
```
سپس نسخه مشابه را برای PostgreSQL 16 نصب کن:
```bash
sudo apt install timescaledb-2-postgresql-16
```

مقداردهی اولیه PostgreSQL 16

(این مسیر ممکنه با نسخه سیستم شما متفاوت باشد، مسیر دقیق را با pg_lsclusters چک کن)
```bash
sudo /usr/lib/postgresql/16/bin/initdb -D /var/lib/postgresql/16/main
```

اجرای ارتقا با کاربر postgres

 We will run the provided update script, which will check for compatibility and fix internal tables if لازم بود.
 ```bash
su - postgres
```


انتقال فایل‌های کانفیگ
```bash
sudo cp /etc/postgresql/13/main/pg_hba.conf /etc/postgresql/16/main/
sudo cp /etc/postgresql/13/main/postgresql.conf /etc/postgresql/16/main/
```

اجرای مهاجرت دیتابیس
```bash
/sudo pg_upgrade \
  -b /usr/lib/postgresql/13/bin \
  -B /usr/lib/postgresql/16/bin \
  -d /var/lib/postgresql/13/main \
  -D /var/lib/postgresql/16/main \
  -k

```


خروج از کاربر postgres
```bash
logout
```

غیرفعال‌کردن نسخه قدیمی
```bash
systemctl disable postgresql-13.service
sudo systemctl disable postgresql@13-main
sudo systemctl stop postgresql@13-main
```

فعال و شروع PostgreSQL 16

```bash
sudo systemctl enable postgresql@16-main
sudo systemctl start postgresql@16-main
```


به‌روزرسانی TimescaleDB در دیتابیس Zabbix

```bash
sudo -i -u postgres psql
\c zabbix
ALTER EXTENSION timescaledb UPDATE;
\q
```



پاکسازی و حذف نسخه‌های قدیمی PostgreSQL

```bash
sudo apt remove postgresql-13 postgresql-client-13 postgresql-contrib-13
sudo rm -rf /var/lib/postgresql/13/
sudo rm -rf /etc/postgresql/13/

```


افزودن ریپو و آپدیت Zabbix
```bash
wget https://repo.zabbix.com/zabbix/7.0/ubuntu/pool/main/z/zabbix-release/zabbix-release_7.0-1+ubuntu$(lsb_release -rs)_all.deb
sudo dpkg -i zabbix-release_7.0-1+ubuntu$(lsb_release -rs)_all.deb
sudo apt update
sudo apt install --only-upgrade zabbix-server-pgsql zabbix-frontend-php zabbix-agent

```


شروع Zabbix و آپدیت دیتابیس

```bash
sudo systemctl start zabbix-server

```
بعد از شروع، Zabbix جداول جدید مثل history_bin رو میسازه.


توقف مجدد Zabbix
```bash
sudo systemctl stop zabbix-server

```

اجرای اسکریپت TimescaleDB


```bash
cat /usr/share/zabbix-sql-scripts/postgresql/timescaledb/schema.sql | sudo -u zabbix psql zabbix

```


شروع مجدد سرویس‌ها
```bash
sudo systemctl start zabbix-server
sudo systemctl start apache2

```


بررسی لاگ Zabbix


```bash
sudo tail -f /var/log/zabbix/zabbix_server.log

```


بهینه‌سازی PostgreSQL

```bash
sudo timescaledb-tune --pg-config=/usr/lib/postgresql/16/bin/pg_config --max-conns=100
sudo systemctl restart postgresql@16-main

```







