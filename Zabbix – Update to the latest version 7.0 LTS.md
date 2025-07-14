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
systemctl stop postgresql-13
```

نصب TimescaleDB
حالا TimescaleDB رو نصب می‌کنیم، ولی حتما باید همون نسخه قبلی باشه، چون در غیر اینصورت ارتقا کار نمی‌کنه.
```bash
VERSION=$(rpm -qa | grep timescaledb-2-postgresql-13 | head -1 | sed -n 's/.*-\([0-9]\+\.[0-9]\+\.[0-9]\+\)-.*/\1/p') && dnf install -y timescaledb-2-loader-postgresql-16-$VERSION-0.el9.x86_64 timescaledb-2-postgresql-16-$VERSION-0.el9.x86_64
```

مقداردهی اولیه PostgreSQL 16
```bash
/usr/pgsql-16/bin/postgresql-16-setup initdb
```

اجرای ارتقا با کاربر postgres

 We will run the provided update script, which will check for compatibility and fix internal tables if لازم بود.
 ```bash
su - postgres
```


انتقال فایل‌های کانفیگ
```bash
cat /var/lib/pgsql/13/data/pg_hba.conf > /var/lib/pgsql/16/data/pg_hba.conf

cat /var/lib/pgsql/13/data/postgresql.conf > /var/lib/pgsql/16/data/postgresql.conf
```

اجرای مهاجرت دیتابیس
```bash
/usr/pgsql-16/bin/pg_upgrade -b /usr/pgsql-13/bin -B /usr/pgsql-16/bin -d /var/lib/pgsql/13/data -D /var/lib/pgsql/16/data -k
```


خروج از کاربر postgres
```bash
logout
```

غیرفعال‌کردن نسخه قدیمی
```bash
systemctl disable postgresql-13.service
```

فعال و شروع PostgreSQL 16

```bash
systemctl enable postgresql-16.service --now
```


به‌روزرسانی TimescaleDB در دیتابیس Zabbix

```bash
su - postgres
psql -X
\c zabbix
ALTER EXTENSION timescaledb UPDATE;
exit
```



پاکسازی و حذف نسخه‌های قدیمی PostgreSQL

```bash
./delete_old_cluster.sh
rm -rf 13 delete_old_cluster.sh
logout
dnf remove postgresql13-*
rm -rf /usr/pgsql-13/
```


افزودن ریپو و آپدیت Zabbix
```bash
rpm -Uvh https://repo.zabbix.com/zabbix/7.0/rocky/9/x86_64/zabbix-release-latest.el9.noarch.rpm
dnf clean all
dnf update zabbix-* -y
```


شروع Zabbix و آپدیت دیتابیس

```bash
systemctl start zabbix-server.service
```
بعد از شروع، Zabbix جداول جدید مثل history_bin رو میسازه.


توقف مجدد Zabbix
```bash
systemctl stop zabbix-server.service
```

اجرای اسکریپت TimescaleDB


```bash
cat /usr/share/zabbix-sql-scripts/postgresql/timescaledb/schema.sql | sudo -u zabbix psql zabbix
```


شروع مجدد سرویس‌ها
```bash
systemctl start zabbix-server.service
systemctl start httpd
```


بررسی لاگ Zabbix


```bash
tail -f /var/log/zabbix/zabbix_server.log
```


بهینه‌سازی PostgreSQL

```bash
timescaledb-tune --pg-config=/usr/pgsql-16/bin/pg_config --max-conns=100
systemctl restart postgresql-16.service
```







