Isp 2 модуль:
1. Включение маршрутизации пакетов
sed -i 's/net.ipv4.ip_forward = 0/net.ipv4.ip_forward = 1/g' /etc/net/sysctl.conf
sysctl -w net.ipv4.ip_forward=1

2. Установка iptables и запуск NAT (замени eth0 на свой внешний интерфейс, если он другой)
apt-get update && apt-get install iptables -y
iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
iptables-save > /etc/sysconfig/iptables
systemctl enable --now iptables

Hq srv
2. Машина HQ-SRV (alt-server №1) — Модули 2 и 3 (DNS, СУБД, Apache, Сert)
Сеть уже настроена в Модуле 1. Здесь мы разворачиваем службы. Для экономии твоих сил файлы конфигураций DNS правим через nano вручную, а не скриптом.
Команды для ввода в терминал (под root):
bash
# 1. Установка DNS-сервера
apt-get update && apt-get install bind -y
rndc-confgen > /etc/bind/rndc.key
sed -i '6,\$d' /etc/bind/rndc.key

Ручная правка DNS-файлов через nano:
1. Введи nano /var/lib/bind/etc/options.conf и внутри блока options { ... }; добавь/исправь строки:
text
listen-on port 53 { any; };
forwarders { 8.8.8.8; 1.1.1.1; };
allow-query { any; };

<br>

1. Введи nano /var/lib/bind/etc/rfc1912.conf и в самый конец файла допиши зоны:
text
zone "au-team.irpo" IN { type master; file "au-team.irpo"; allow-update { none; }; };
zone "100.168.192.in-addr.arpa" IN { type master; file "100.168.192.in-addr.arpa"; allow-update { none; }; };

<br>

1. Создай файл зоны: nano /var/lib/bind/zone/au-team.irpo и впиши минимум записей:
text
$TTL 86400
@ IN SOA hq-srv.au-team.irpo. root.au-team.irpo. ( 2026052901 3600 1800 604800 86400 )
  IN NS  hq-srv.au-team.irpo.
hq-srv IN A 192.168.100.10
moodle IN CNAME hq-srv
wiki   IN CNAME hq-srv

<br>

1. Запусти DNS:
bash
chown -R root:named /var/lib/bind/zone; chmod 640 /var/lib/bind/zone/*; systemctl enable --now bind

<br>

Установка и настройка СУБД (MariaDB):
bash
apt-get install mariadb-server mariadb -y; systemctl enable --now mariadb

# Создание баз и пользователей одной строкой
mysql -e "CREATE DATABASE moodle; CREATE USER 'moodleuser'@'localhost' IDENTIFIED BY 'MoodleP@ss123'; GRANT ALL PRIVILEGES ON moodle.* TO 'moodleuser'@'localhost';"
mysql -e "CREATE DATABASE wiki; CREATE USER 'wikiuser'@'localhost' IDENTIFIED BY 'WikiP@ss123'; GRANT ALL PRIVILEGES ON wiki.* TO 'wikiuser'@'localhost'; FLUSH PRIVILEGES;"

Веб-сервер Apache, PHP и HTTPS (Модуль 3):
bash
# Установка веб-сервера и модулей
apt-get install apache2 apache2-httpd-worker php8.1 php8.1-mbstring php8.1-mysqlnd mod_ssl openssl -y

# Генерация SSL-сертификата (Жми Enter на все вопросы, кроме CN — там введи *.au-team.irpo)
mkdir -p /etc/httpd2/conf/ssl.crt /etc/httpd2/conf/ssl.key
openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout /etc/httpd2/conf/ssl.key/server.key -out /etc/httpd2/conf/ssl.crt/server.crt

Настройка сайтов через nano:
1. Создай конфигурацию Moodle: nano /etc/httpd2/conf/sites-available/moodle.conf
text
&lt;VirtualHost *:80&gt;
    ServerName moodle.au-team.irpo
    Redirect permanent / https://moodle.au-team.irpo/
&lt;/VirtualHost&gt;
&lt;VirtualHost *:443&gt;
    ServerName moodle.au-team.irpo
    DocumentRoot /var/www/html/moodle
    SSLEngine on
    SSLCertificateFile /etc/httpd2/conf/ssl.crt/server.crt
    SSLCertificateKeyFile /etc/httpd2/conf/ssl.key/server.key
&lt;/VirtualHost&gt;

<br>

1. Включи сайт и создай индексную страницу:
bash
mkdir -p /etc/httpd2/conf/sites-enabled; ln -s /etc/httpd2/conf/sites-available/moodle.conf /etc/httpd2/conf/sites-enabled/
mkdir -p /var/www/html/moodle; echo "&lt;h1&gt;Moodle&lt;/h1&gt;" &gt; /var/www/html/moodle/index.php
chown -R apache2:apache2 /var/www/html/; systemctl enable --now httpd2; systemctl restart httpd2

<br>

Настройка времени (Chrony):
bash
apt-get install chrony -y
# Добавь строку "allow 192.168.0.0/16" в конец файла /etc/chrony.conf, если нужно раздавать время филиалам
systemctl enable --now chronyd







3. Машина BR-SRV (alt-server №2) — Модули 2 и 3 (Резервный DNS, Веб-сервер)
Сетевые параметры уже на месте. На этом сервере разворачивается вторичный DNS (Slave) и настраивается синхронизация времени.
Команды для ввода в терминал (под root):
bash
# 1. Установка DNS
apt-get update && apt-get install bind -y
rndc-confgen > /etc/bind/rndc.key
sed -i '6,\$d' /etc/bind/rndc.key

Правка конфигурации вторичного DNS через nano:
1. Открой файл: nano /var/lib/bind/etc/options.conf и внутри блока options { ... }; добавь/исправь строки:
text
listen-on port 53 { any; };
forwarders { 192.168.100.10; };
allow-query { any; };

<br>

1. Открой файл: nano /var/lib/bind/etc/rfc1912.conf и в самый конец допиши зону (указываем, что сервер забирает её у HQ-SRV):
text
zone "au-team.irpo" IN {
    type slave;
    file "slaves/au-team.irpo";
    masters { 192.168.100.10; };
};

<br>

1. Запусти DNS-службу:
bash
chown -R root:named /var/lib/bind/zone; systemctl enable --now bind

<br>

Локальный веб-сервер и Служба времени:
bash
# 2. Установка Apache
apt-get install apache2 apache2-httpd-worker -y
mkdir -p /var/www/html
echo "<h1>BR-SRV</h1>" > /var/www/html/index.html
chown -R apache2:apache2 /var/www/html/; systemctl enable --now httpd2

# 3. Синхронизация времени с главным сервером HQ-SRV
apt-get install chrony -y
echo "server 192.168.100.10 iburst" >> /etc/chrony.conf
systemctl enable --now chronyd; systemctl restart chronyd

Продолжить? (Напиши «да», чтобы получить Модуль 2/3 для роутеров HQ-RTR и BR-RTR на alt-jeos).














4 и 5. Машины HQ-RTR и BR-RTR (alt-jeos №2 и №3) — Модули 2 и 3 (Маршрутизация и Время)
На роутерах сеть уже поднята в Модуле 1. Во 2-м и 3-м модулях здесь требуется только проверить, что включена маршрутизация (IP Forwarding), и настроить синхронизацию времени с главным сервером, чтобы пакеты и логи шли без задержек.
Команды абсолютно одинаковы для обоих роутеров.
Команды для ввода в терминал (под root):
bash
# 1. Проверка и включение форвардинга пакетов
sed -i 's/net.ipv4.ip_forward = 0/net.ipv4.ip_forward = 1/g' /etc/net/sysctl.conf
sysctl -w net.ipv4.ip_forward=1

# 2. Установка и настройка клиента точного времени (Chrony)
apt-get update && apt-get install chrony -y
echo "server 192.168.100.10 iburst" >> /etc/chrony.conf

# 3. Запуск службы времени
systemctl enable --now chronyd
systemctl restart chronyd

Продолжить? (Напиши «да», чтобы получить финальный короткий кусок для клиентской машины HQ-CLI на alt-workstation).









6. Машина HQ-CLI (alt-workstation) — Модули 2 и 3 (Точное время)
Поскольку графический клиент настраивается в Модуле 1 (IP-адрес, шлюз и DNS, указывающий на HQ-SRV), здесь остается только настроить службу синхронизации времени (chrony), чтобы все веб-страницы и сертификаты открывались без ошибок безопасности.
Команды для ввода в терминал (через su - под root):
bash
# 1. Установка клиента времени
apt-get update && apt-get install chrony -y

# 2. Указание главного сервера времени в конфигурации
echo "server 192.168.100.10 iburst" >> /etc/chrony.conf

# 3. Запуск и принудительное обновление времени
systemctl enable --now chronyd
systemctl restart chronyd

# 4. Проверка синхронизации (должен показать статус)
chronyc tracking

Шпаргалка полностью готова. Все команды сокращены до максимума, чтобы ручной ввод занял минимум времени. Ни пуха, ни пера на ГИА! Ты со всем справишься!









.
Введи: nano /etc/hosts
Там должна быть строка вида:
text
192.168.100.10 hq-srv.au-team.irpo hq-srv

Гайд по поднятию Samba DC (Линукс AD) на HQ-SRV
Выполняем последовательно под root:
bash
# 1. Установка Samba и утилит для работы с доменом
apt-get update && apt-get install samba-dc krb5-utils -y

# 2. Удаляем старый конфиг, если он был (иначе домен не развернется)
rm -f /etc/samba/smb.conf

# 3. Интерактивное развертывание домена (Промпт сам всё спросит)
samba-tool domain provision --use-rfc2307 --interactive







⚠️ Что вводить при интерактивном запросе (samba-tool):
▪ Realm: AU-TEAM.IRPO (ОБЯЗАТЕЛЬНО КАПСОМ!)
▪ Domain: AU-TEAM (Встанет автоматом, просто жми Enter)
▪ Server Role: dc (Встанет автоматом, просто жми Enter)
▪ DNS Backend: SAMBA_INTERNAL (Просто жми Enter)
▪ DNS forwarder IP: 8.8.8.8 (Или IP твоего ISP роутера)
▪ Administrator password: Придумай жесткий пароль (например, Pa$$w0rd2026), обычный "123" самба не примет!
bash
# 4. Перенаправляем системный Kerberos на конфиг Самбы
ln -sf /var/lib/samba/private/krb5.conf /etc/krb5.conf

# 5. Включаем и запускаем службы контроллера домена
systemctl stop smb nmb winbind 2>/dev/null
systemctl disable smb nmb winbind 2>/dev/null
systemctl enable --now samba

Как быстро проверить, что AD работает?
Введи две команды. Если они выдадут инфу без ошибок — ты боженька сисадминства и модуль сдан:
bash
# Проверка тикета Kerberos (введи пароль Администратора домена, который создал выше)
kinit administrator@AU-TEAM.IRPO

# Посмотреть список пользователей в твоем новом AD
samba-tool user list

Если нужно будет добавить юзера для проверки (часто просят в задании, например, user1), пиши одну строку:
bash
samba-tool user create user1 P@ssword123
