# follfy
follfy
# 2.1 Базовая настройка имен и IPv4
HQ-SRV

hostnamectl set-hostname hq-srv.au-team.irpo

nmcli con mod eth0 ipv4.addresses 192.168.10.2/27 ipv4.gateway 192.168.10.1 ipv4.dns 192.168.10.2 ipv4.method manual

nmcli con up eth0

BR-SRV

hostnamectl set-hostname br-srv.au-team.irpo

nmcli con mod eth0 ipv4.addresses 192.168.30.2/28 ipv4.gateway 192.168.30.1 ipv4.dns 192.168.10.2 ipv4.method manual

nmcli con up eth0

HQ-CLI

hostnamectl set-hostname hq-cli.au-team.irpo

адрес получает по DHCP после настройки HQ-RTR

ISP

hostnamectl set-hostname isp.au-team.irpo

ip addr add 172.16.1.1/28 dev eth1

ip addr add 172.16.2.1/28 dev eth2

ip link set eth1 up

ip link set eth2 up

HQ-RTR
hostnamectl set-hostname hq-rtr.au-team.irpo

ip addr add 172.16.1.2/28 dev eth0

ip link set eth0 up

ip route add default via 172.16.1.1

BR-RTR

hostnamectl set-hostname br-rtr.au-team.irpo

ip addr add 172.16.2.2/28 dev eth0

ip addr add 192.168.30.1/28 dev eth1

ip link set eth0 up

ip link set eth1 up

ip route add default via 172.16.2.1

# 2.2 Настройка доступа к сети Интернет на ISP

включение маршрутизации на ISP

sysctl -w net.ipv4.ip_forward=1

echo 'net.ipv4.ip_forward = 1' > /etc/sysctl.d/99-forward.conf

внешний интерфейс eth0 получает адрес от провайдера по DHCP
nmcli con mod eth0 ipv4.method auto

nmcli con up eth0

NAT для выхода HQ и BR через внешний интерфейс ISP

iptables -t nat -A POSTROUTING -s 172.16.1.0/28 -o eth0 -j MASQUERADE

iptables -t nat -A POSTROUTING -s 172.16.2.0/28 -o eth0 -j MASQUERADE

сохранение правил, если установлен iptables-services

service iptables save || iptables-save > /etc/sysconfig/iptables
Проверка результата:

•	С ISP доступен внешний адрес: ping 77.88.8.8.

•	С HQ-RTR и BR-RTR доступен ISP.

•	На ISP включен net.ipv4.ip_forward=1.

# 2.3 Создание локальных учетных записей

В тексте задания указано создание пользователя remote_user, но далее пароль и права задаются для sshuser. Для выполнения требований создается пользователь sshuser с UID 2026, так как именно он используется в SSH-доступе.
на HQ-SRV и BR-SRV

useradd -u 2026 -m sshuser

printf 'P@ssw0rd

P@ssw0rd

' | passwd sshuser

usermod -aG wheel sshuser

echo 'sshuser ALL=(ALL) NOPASSWD: ALL' > /etc/sudoers.d/sshuser

chmod 440 /etc/sudoers.d/sshuser

на HQ-RTR и BR-RTR

useradd -m net_admin

printf 'P@ssw0rd

P@ssw0rd

' | passwd net_admin
usermod -aG wheel net_admin

echo 'net_admin ALL=(ALL) NOPASSWD: ALL' > /etc/sudoers.d/net_admin

chmod 440 /etc/sudoers.d/net_admin

# 2.4 Настройка VLAN и маршрутизации в HQ
на HQ-RTR создаются VLAN-интерфейсы

modprobe 8021q

ip link add link eth1 name eth1.100 type vlan id 100

ip link add link eth1 name eth1.200 type vlan id 200

ip link add link eth1 name eth1.999 type vlan id 999

ip addr add 192.168.10.1/27 dev eth1.100

ip addr add 192.168.20.1/27 dev eth1.200

ip addr add 192.168.99.1/29 dev eth1.999

ip link set eth1 up

ip link set eth1.100 up

ip link set eth1.200 up

ip link set eth1.999 up

sysctl -w net.ipv4.ip_forward=1

echo 'net.ipv4.ip_forward = 1' > /etc/sysctl.d/99-forward.conf

# 2.5 Настройка безопасного SSH-доступа

на HQ-SRV и BR-SRV

cp /etc/openssh/sshd_config /etc/openssh/sshd_config.bak 2>/dev/null || cp /etc/ssh/sshd_config /etc/ssh/sshd_config.bak

cat >> /etc/ssh/sshd_config <<'EOF'

Port 2026

AllowUsers sshuser

MaxAuthTries 2

Banner /etc/issue.net

PermitRootLogin no

EOF

echo 'Authorized access only' > /etc/issue.net

systemctl restart sshd || systemctl restart ssh

ss -tulpn | grep 2026

# 2.6 Настройка GRE-туннеля между HQ и BR

HQ-RTR

ip tunnel add gre1 mode gre remote 172.16.2.2 local 172.16.1.2 ttl 255

ip addr add 10.10.10.1/30 dev gre1

ip link set gre1 up

BR-RTR

ip tunnel add gre1 mode gre remote 172.16.1.2 local 172.16.2.2 ttl 255

ip addr add 10.10.10.2/30 dev gre1

ip link set gre1 up

проверка

ping 10.10.10.2    # с HQ-RTR

ping 10.10.10.1    # с BR-RTR

#  2.7 Настройка OSPF с парольной защитой

установка FRR на HQ-RTR и BR-RTR

apt-get install -y frr

sed -i 's/ospfd=no/ospfd=yes/' /etc/frr/daemons

systemctl enable --now frr

HQ-RTR

vtysh <<'EOF'

conf t

router ospf

 ospf router-id 1.1.1.1
 
 network 10.10.10.0/30 area 0
 
 network 192.168.10.0/27 area 0
 
 network 192.168.20.0/27 area 0
 
 network 192.168.99.0/29 area 0
 
 passive-interface default
 
 no passive-interface gre1
 
 area 0 authentication message-digest
 
exit

interface gre1

 ip ospf authentication message-digest
 
 ip ospf message-digest-key 1 md5 P@ssw0rd
 
exit

end

write

EOF


BR-RTR

vtysh <<'EOF'

conf t

router ospf

 ospf router-id 2.2.2.2
 
 network 10.10.10.0/30 area 0
 
 network 192.168.30.0/28 area 0
 
 passive-interface default
 
 no passive-interface gre1
 
 area 0 authentication message-digest
 
exit

interface gre1

 ip ospf authentication message-digest
 
 ip ospf message-digest-key 1 md5 P@ssw0rd
exit

end

write

EOF

# 2.8 Настройка NAT на HQ-RTR и BR-RTR

HQ-RTR

iptables -t nat -A POSTROUTING -s 192.168.10.0/27 -o eth0 -j MASQUERADE

iptables -t nat -A POSTROUTING -s 192.168.20.0/27 -o eth0 -j MASQUERADE

iptables -t nat -A POSTROUTING -s 192.168.99.0/29 -o eth0 -j MASQUERADE

BR-RTR

iptables -t nat -A POSTROUTING -s 192.168.30.0/28 -o eth0 -j MASQUERADE

сохранение

service iptables save || iptables-save > /etc/sysconfig/iptables

# 2.9 Настройка DHCP для HQ-CLI

HQ-RTR

apt-get install -y dhcp-server

cat > /etc/dhcp/dhcpd.conf <<'EOF'

option domain-name "au-team.irpo";

option domain-name-servers 192.168.10.2;

default-lease-time 600;

max-lease-time 7200;

authoritative;


subnet 192.168.20.0 netmask 255.255.255.224 {

  range 192.168.20.10 192.168.20.30;
  
  option routers 192.168.20.1;
  
  option domain-name-servers 192.168.10.2;
  
  option domain-name "au-team.irpo";
  
}

EOF

указать интерфейс eth1.200 для dhcpd, если требуется в конкретной ОС

systemctl enable --now dhcpd

systemctl restart dhcpd

# 2.10 Настройка DNS на HQ-SRV

HQ-SRV

apt-get install -y bind bind-utils

cat > /etc/named.conf <<'EOF'

options {

    directory "/var/lib/bind";
    
    recursion yes;
    
    allow-query { any; };
    
    forwarders { 77.88.8.7; 77.88.8.3; };
    
};

zone "au-team.irpo" IN {

    type master;
    
    file "au-team.irpo.zone";
    
};

zone "10.168.192.in-addr.arpa" IN {

    type master;
    
    file "192.168.10.zone";
    
};

zone "20.168.192.in-addr.arpa" IN {

    type master;
    
    file "192.168.20.zone";
    
};

EOF

cat > /var/lib/bind/au-team.irpo.zone <<'EOF'

$TTL 86400

@ IN SOA hq-srv.au-team.irpo. root.au-team.irpo. (

  2026010101 3600 900 604800 86400 )
  
@       IN NS hq-srv.au-team.irpo.

hq-rtr  IN A 192.168.10.1

br-rtr  IN A 192.168.30.1

hq-srv  IN A 192.168.10.2

hq-cli  IN A 192.168.20.10

br-srv  IN A 192.168.30.2

docker  IN A 172.16.1.1

web     IN A 172.16.2.1

mon     IN A 192.168.10.2

EOF

cat > /var/lib/bind/192.168.10.zone <<'EOF'

$TTL 86400

@ IN SOA hq-srv.au-team.irpo. root.au-team.irpo. (

  2026010101 3600 900 604800 86400 )
  
@ IN NS hq-srv.au-team.irpo.

1 IN PTR hq-rtr.au-team.irpo.

2 IN PTR hq-srv.au-team.irpo.

EOF

cat > /var/lib/bind/192.168.20.zone <<'EOF'

$TTL 86400

@ IN SOA hq-srv.au-team.irpo. root.au-team.irpo. (

  2026010101 3600 900 604800 86400 )
  
@ IN NS hq-srv.au-team.irpo.

10 IN PTR hq-cli.au-team.irpo.

EOF

named-checkconf

named-checkzone au-team.irpo /var/lib/bind/au-team.irpo.zone

systemctl enable --now named || systemctl enable --now bind

systemctl restart named || systemctl restart bind


# 2.11 Настройка часового пояса

timedatectl set-timezone Europe/Moscow

timedatectl status

# 3.1 Samba DC на BR-SRV

BR-SRV

apt-get install -y samba samba-dc samba-client krb5-workstation oddjob oddjob-mkhomedir

systemctl stop smb nmb winbind 2>/dev/null || true

mv /etc/samba/smb.conf /etc/samba/smb.conf.bak 2>/dev/null || true


samba-tool domain provision   --realm=AU-TEAM.IRPO   --domain=AU   --server-role=dc   --dns-backend=SAMBA_INTERNAL   --adminpass='P@ssw0rd'

systemctl enable --now samba

создание пользователей hquser1-hquser5

for i in 1 2 3 4 5; do

  samba-tool user create hquser$i 'P@ssw0rd'
  
done

группа hq и добавление пользователей

samba-tool group add hq

for i in 1 2 3 4 5; do

  samba-tool group addmembers hq hquser$i
  
done

проверка

samba-tool domain info 127.0.0.1

samba-tool user list

samba-tool group listmembers hq

Ввод HQ-CLI в домен выполняется после настройки DNS-клиента на HQ-CLI, чтобы имя au-team.irpo разрешалось через BR-SRV или настроенный DNS.
HQ-CLI

apt-get install -y realmd sssd adcli krb5-workstation samba-common-tools oddjob oddjob-mkhomedir

realm discover au-team.irpo

realm join au-team.irpo -U Administrator

realm list

# 3.2 Ограниченные права sudo для группы hq

HQ-CLI: разрешить группе hq запускать только cat, grep, id

cat > /etc/sudoers.d/hq-limited <<'EOF'

%hq ALL=(ALL) NOPASSWD: /usr/bin/cat, /usr/bin/grep, /usr/bin/id

EOF

chmod 440 /etc/sudoers.d/hq-limited

visudo -c

проверка под доменным пользователем

su - 'AU\hquser1'

sudo /usr/bin/id

sudo /usr/bin/cat /etc/hostname

sudo /usr/bin/grep root /etc/passwd

sudo /usr/bin/less /etc/passwd   # должно быть запрещено

# 3.3 RAID0 на HQ-SRV

HQ-SRV

lsblk

apt-get install -y mdadm

mdadm --create /dev/md0 --level=0 --raid-devices=2 /dev/sdb /dev/sdc

mdadm --detail /dev/md0

mdadm --detail --scan > /etc/mdadm.conf

создание раздела

parted -s /dev/md0 mklabel gpt

parted -s /dev/md0 mkpart primary ext4 0% 100%

mkfs.ext4 /dev/md0p1

mkdir -p /raid

UUID=$(blkid -s UUID -o value /dev/md0p1)

echo "UUID=$UUID /raid ext4 defaults 0 0" >> /etc/fstab

mount -a

df -h /raid

# 3.4 NFS на HQ-SRV и автомонтирование на HQ-CLI

HQ-SRV

apt-get install -y nfs-server nfs-utils

mkdir -p /raid/nfs

chmod 777 /raid/nfs

cat > /etc/exports <<'EOF'

/raid/nfs 192.168.20.0/27(rw,sync,no_subtree_check,no_root_squash)

EOF

exportfs -ra

systemctl enable --now nfs-server

exportfs -v

HQ-CLI

apt-get install -y nfs-utils autofs

mkdir -p /mnt/nfs

cat >> /etc/fstab <<'EOF'

192.168.10.2:/raid/nfs /mnt/nfs nfs defaults,_netdev 0 0

EOF

mount -a

df -h /mnt/nfs

echo test > /mnt/nfs/check.txt

cat /mnt/nfs/check.txt

# 3.5 Chrony на ISP

ISP

apt-get install -y chrony

cat > /etc/chrony.conf <<'EOF'

server pool.ntp.org iburst

local stratum 5

allow 192.168.0.0/16

allow 172.16.0.0/12

allow 10.0.0.0/8

makestep 1.0 3

rtcsync

EOF

systemctl enable --now chronyd

systemctl restart chronyd

chronyc sources

клиенты HQ-SRV, HQ-CLI, BR-RTR, BR-SRV

cat > /etc/chrony.conf <<'EOF'

server 172.16.1.1 iburst

makestep 1.0 3

rtcsync

EOF

systemctl enable --now chronyd

systemctl restart chronyd

chronyc tracking

# 3.6 Ansible на BR-SRV

BR-SRV

apt-get install -y ansible openssh-clients

mkdir -p /etc/ansible

cat > /etc/ansible/hosts <<'EOF'

[servers]

hq-srv ansible_host=192.168.10.2 ansible_user=sshuser ansible_port=2026

hq-cli ansible_host=192.168.20.10 ansible_user=sshuser ansible_port=22

[routers]

hq-rtr ansible_host=172.16.1.2 ansible_user=net_admin

br-rtr ansible_host=172.16.2.2 ansible_user=net_admin

[all:vars]

ansible_ssh_common_args='-o StrictHostKeyChecking=no'

EOF

настройка ключей, если разрешено условиями стенда

ssh-keygen -t rsa -N '' -f /root/.ssh/id_rsa

ssh-copy-id -p 2026 sshuser@192.168.10.2

ssh-copy-id sshuser@192.168.20.10

ssh-copy-id net_admin@172.16.1.2

ssh-copy-id net_admin@172.16.2.2

ansible all -m ping

# 3.7 Docker-приложение на BR-SRV

BR-SRV

apt-get install -y docker docker-compose-plugin

systemctl enable --now docker

ISO Additional.iso должен быть смонтирован, в задании указаны образы в директории docker

mkdir -p /mnt/additional

mount /dev/cdrom /mnt/additional 2>/dev/null || true

cd /mnt/additional/docker

docker load -i site_latest.tar || docker load -i site_latest.docker

docker load -i mariadb_latest.tar || docker load -i mariadb_latest.docker

mkdir -p /opt/testapp

cd /opt/testapp

cat > docker-compose.yml <<'EOF'

services:

  testapp:
  
    image: site_latest
    
    container_name: testapp
    
    restart: always
    
    ports:
    
      - "8080:8080"
      
    environment:
    
      DB_HOST: db
      
      DB_NAME: testdb
      
      DB_USER: test
      
      DB_PASS: P@ssw0rd
      
    depends_on:
    
      - db
      
  db:
    image: mariadb_latest
    
    container_name: db
    
    restart: always
    
    environment:
    
      MARIADB_DATABASE: testdb
      
      MARIADB_USER: test
      
      MARIADB_PASSWORD: P@ssw0rd
      
      MARIADB_ROOT_PASSWORD: P@ssw0rd
      
EOF

docker compose up -d

docker ps

curl http://127.0.0.1:8080

В тексте задания есть опечатка: основной контейнер testapp должен называться tespapp. Для максимального соответствия при необходимости можно заменить container_name: testapp на container_name: tespapp, но логически приложение называется testapp.


# 3.8 Apache и MariaDB на HQ-SRV

HQ-SRV

apt-get install -y apache2 mariadb-server php php-mysql php-mysqli

systemctl enable --now mariadb

systemctl enable --now httpd || systemctl enable --now apache2

mysql <<'EOF'

CREATE DATABASE webdb CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;

CREATE USER 'web'@'localhost' IDENTIFIED BY 'P@ssw0rd';

GRANT ALL PRIVILEGES ON webdb.* TO 'web'@'localhost';

FLUSH PRIVILEGES;

EOF

mkdir -p /mnt/additional

mount /dev/cdrom /mnt/additional 2>/dev/null || true

mysql webdb < /mnt/additional/web/dump.sql

cp /mnt/additional/web/index.php /var/www/html/index.php

cp -r /mnt/additional/web/images /var/www/html/images

в index.php указываются параметры:

host=localhost, database=webdb, user=web, password=P@ssw0rd

systemctl restart httpd || systemctl restart apache2

curl http://127.0.0.1

# 3.9 Статическая трансляция портов

HQ-RTR: внешний порт 8080 на веб-приложение HQ-SRV, порт 2026 на SSH HQ-SRV

iptables -t nat -A PREROUTING -i eth0 -p tcp --dport 8080 -j DNAT --to-destination 192.168.10.2:80

iptables -t nat -A PREROUTING -i eth0 -p tcp --dport 2026 -j DNAT --to-destination 192.168.10.2:2026

iptables -A FORWARD -p tcp -d 192.168.10.2 --dport 80 -j ACCEPT

iptables -A FORWARD -p tcp -d 192.168.10.2 --dport 2026 -j ACCEPT

BR-RTR: внешний порт 8080 на testapp BR-SRV, порт 2026 на SSH BR-SRV

iptables -t nat -A PREROUTING -i eth0 -p tcp --dport 8080 -j DNAT --to-destination 192.168.30.2:8080

iptables -t nat -A PREROUTING -i eth0 -p tcp --dport 2026 -j DNAT --to-destination 192.168.30.2:2026

iptables -A FORWARD -p tcp -d 192.168.30.2 --dport 8080 -j ACCEPT

iptables -A FORWARD -p tcp -d 192.168.30.2 --dport 2026 -j ACCEPT

service iptables save || iptables-save > /etc/sysconfig/iptables

# 3.10 Nginx как обратный прокси на ISP
ISP

apt-get install -y nginx

cat > /etc/nginx/conf.d/reverse.conf <<'EOF'

server {

    listen 80;
    
    server_name web.au-team.irpo;
    
    location / {
    
        proxy_pass http://172.16.1.2:8080;
        
        proxy_set_header Host $host;
        
        proxy_set_header X-Real-IP $remote_addr;
        
    }
    
}

server {

    listen 80;
    
    server_name docker.au-team.irpo;
    
    location / {
    
        proxy_pass http://172.16.2.2:8080;
        
        proxy_set_header Host $host;
        
        proxy_set_header X-Real-IP $remote_addr;
        
    }
    
}

EOF

nginx -t

systemctl enable --now nginx

systemctl reload nginx

# 3.11 Web-based аутентификация
ISP

apt-get install -y apache2-utils || apt-get install -y httpd-tools

htpasswd -bc /etc/nginx/.htpasswd WEB 'P@ssw0rd'

добавить в server web.au-team.irpo:

auth_basic "Authorized users only";

auth_basic_user_file /etc/nginx/.htpasswd;

nginx -t

systemctl reload nginx

# 3.12 Установка Яндекс Браузера на HQ-CLI

HQ-CLI

вариант установки из локального rpm/deb пакета, если он есть на стенде

apt-get install -y ./yandex-browser*.rpm || apt-get install -y ./yandex-browser*.deb

проверка

yandex-browser --version

# 4.1 Импорт пользователей в домен

BR-SRV

mkdir -p /mnt/additional

mount /dev/cdrom /mnt/additional 2>/dev/null || true

пример CSV: username,password,firstname,lastname

импорт выполняется скриптом, если структура users.csv стандартная

while IFS=',' read -r username password firstname lastname; do

  [ "$username" = "username" ] && continue
  
  samba-tool user create "$username" "$password" --given-name="$firstname" --surname="$lastname"
  
done < /mnt/additional/users.csv

samba-tool user list

# 4.2 Центр сертификации и сертификаты

Требование задания: использовать отечественные алгоритмы шифрования и выдавать сертификаты на 30 дней. В Linux это может быть реализовано с использованием OpenSSL с поддержкой ГОСТ-движка, если он установлен в образе. Если ГОСТ-движок отсутствует в стенде, следует использовать доступный криптопровайдер, предусмотренный организаторами.

HQ-SRV: подготовка каталога CA

mkdir -p /root/ca/{certs,private,csr,newcerts}

chmod 700 /root/ca/private

touch /root/ca/index.txt

echo 1000 > /root/ca/serial

пример корневого сертификата на доступных алгоритмах стенда

openssl req -x509 -newkey rsa:4096 -days 365 -nodes   -keyout /root/ca/private/ca.key   -out /root/ca/certs/ca.crt   -subj "/C=RU/O=AU Team/CN=AU-TEAM-CA"

сертификат для web.au-team.irpo

openssl req -newkey rsa:2048 -nodes   -keyout /root/ca/private/web.key   -out /root/ca/csr/web.csr   -subj "/C=RU/O=AU Team/CN=web.au-team.irpo"

openssl x509 -req -in /root/ca/csr/web.csr   -CA /root/ca/certs/ca.crt -CAkey /root/ca/private/ca.key -CAcreateserial   -out /root/ca/certs/web.crt -days 30

сертификат для docker.au-team.irpo

openssl req -newkey rsa:2048 -nodes   -keyout /root/ca/private/docker.key   -out /root/ca/csr/docker.csr   -subj "/C=RU/O=AU Team/CN=docker.au-team.irpo"

openssl x509 -req -in /root/ca/csr/docker.csr   -CA /root/ca/certs/ca.crt -CAkey /root/ca/private/ca.key -CAcreateserial   -out /root/ca/certs/docker.crt -days 30

На HQ-CLI корневой сертификат CA импортируется в системное хранилище доверенных сертификатов или в хранилище браузера.

HQ-CLI

cp ca.crt /etc/pki/ca-trust/source/anchors/au-team-ca.crt

update-ca-trust extract || update-ca-certificates

ISP: пример HTTPS reverse proxy

mkdir -p /etc/nginx/certs

скопировать web.crt/web.key/docker.crt/docker.key на ISP

cat > /etc/nginx/conf.d/reverse-ssl.conf <<'EOF'

server {

    listen 443 ssl;
    
    server_name web.au-team.irpo;
    
    ssl_certificate /etc/nginx/certs/web.crt;
    
    ssl_certificate_key /etc/nginx/certs/web.key;
    
    location / { proxy_pass http://172.16.1.2:8080; }
    
}

server {

    listen 443 ssl;
    
    server_name docker.au-team.irpo;
    
    ssl_certificate /etc/nginx/certs/docker.crt;
    
    ssl_certificate_key /etc/nginx/certs/docker.key;
    
    location / { proxy_pass http://172.16.2.2:8080; }
    
}

EOF

nginx -t

systemctl reload nginx

# 4.3 Переход с GRE на защищенный туннель WireGuard

HQ-RTR и BR-RTR

apt-get install -y wireguard-tools

генерация ключей на каждом маршрутизаторе

wg genkey | tee /etc/wireguard/private.key | wg pubkey > /etc/wireguard/public.key

HQ-RTR /etc/wireguard/wg0.conf

cat > /etc/wireguard/wg0.conf <<'EOF'

[Interface]

Address = 10.20.20.1/30

PrivateKey = <PRIVATE_KEY_HQ>

ListenPort = 51820

[Peer]
PublicKey = <PUBLIC_KEY_BR>

AllowedIPs = 10.20.20.2/32,192.168.30.0/28

Endpoint = 172.16.2.2:51820

PersistentKeepalive = 25

EOF

BR-RTR /etc/wireguard/wg0.conf

cat > /etc/wireguard/wg0.conf <<'EOF'

[Interface]
Address = 10.20.20.2/30

PrivateKey = <PRIVATE_KEY_BR>

ListenPort = 51820

[Peer]

PublicKey = <PUBLIC_KEY_HQ>

AllowedIPs = 10.20.20.1/32,192.168.10.0/27,192.168.20.0/27,192.168.99.0/29

Endpoint = 172.16.1.2:51820

PersistentKeepalive = 25

EOF

systemctl enable --now wg-quick@wg0

wg show

После запуска wg0 OSPF переносится с gre1 на wg0. Интерфейс gre1 можно отключить после проверки соседства OSPF через защищенный канал.

HQ-RTR и BR-RTR: в FRR заменить no passive-interface gre1 на wg0

vtysh

conf t

router ospf

 passive-interface default
 
 no passive-interface wg0
 
exit

interface wg0

 ip ospf authentication message-digest
 
 ip ospf message-digest-key 1 md5 P@ssw0rd
 
end

write

# 4.4 Межсетевой экран на HQ-RTR и BR-RTR

базовый пример для HQ-RTR и BR-RTR

iptables -P INPUT DROP

iptables -P FORWARD DROP
iptables -P OUTPUT ACCEPT

разрешение established/related

iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT

iptables -A FORWARD -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT

разрешенные протоколы

iptables -A INPUT -p icmp -j ACCEPT

iptables -A FORWARD -p icmp -j ACCEPT

iptables -A FORWARD -p tcp -m multiport --dports 80,443 -j ACCEPT

iptables -A FORWARD -p udp --dport 53 -j ACCEPT

iptables -A FORWARD -p tcp --dport 53 -j ACCEPT

iptables -A FORWARD -p udp --dport 123 -j ACCEPT

iptables -A INPUT -p udp --dport 51820 -j ACCEPT

ssh для администрирования при необходимости

iptables -A INPUT -p tcp --dport 2026 -j ACCEPT

service iptables save || iptables-save > /etc/sysconfig/iptables

# 4.5 Принт-сервер CUPS

HQ-SRV

apt-get install -y cups cups-pdf

systemctl enable --now cups

cupsctl --remote-admin --remote-any --share-printers

lpadmin -p PDF -E -v cups-pdf:/ -m everywhere 2>/dev/null || lpadmin -p PDF -E -v cups-pdf:/

lpstat -p

HQ-CLI

apt-get install -y cups-client

lpadmin -p PDF -E -v ipp://192.168.10.2/printers/PDF -m everywhere

lpadmin -d PDF

lpstat -d

# 4.6 Централизованное логирование rsyslog и ротация

HQ-SRV: сервер rsyslog

apt-get install -y rsyslog logrotate

mkdir -p /opt

cat > /etc/rsyslog.d/10-remote.conf <<'EOF'

module(load="imudp")

input(type="imudp" port="514")

module(load="imtcp")

input(type="imtcp" port="514")

$template RemoteLogs,"/opt/%HOSTNAME%/%PROGRAMNAME%.log"

*.warning ?RemoteLogs

& stop

EOF

systemctl restart rsyslog

клиенты HQ-RTR, BR-RTR, BR-SRV

cat > /etc/rsyslog.d/90-remote-client.conf <<'EOF'

*.warning @@192.168.10.2:514

EOF

systemctl restart rsyslog

logger -p warning "test warning from $(hostname)"

logrotate на HQ-SRV

cat > /etc/logrotate.d/remote-opt <<'EOF'

/opt/*/*.log {

    weekly
    
    rotate 8
    
    compress
    
    missingok
    
    notifempty
    
    size 10M
    
    create 0640 root root
    
}

EOF

logrotate -d /etc/logrotate.d/remote-opt

# 4.7 Мониторинг устройств
Для выполнения задания удобно использовать Zabbix, так как он отображает ЦП, ОЗУ, диск и имеет веб-интерфейс с учетной записью администратора. Допускается использовать другое открытое ПО при выполнении всех требований.
HQ-SRV: пример установки Zabbix из пакетов, если они доступны в репозитории стенда

apt-get install -y zabbix-server-mysql zabbix-web zabbix-agent mariadb-server

mysql <<'EOF'

CREATE DATABASE zabbix CHARACTER SET utf8mb4 COLLATE utf8mb4_bin;

CREATE USER 'zabbix'@'localhost' IDENTIFIED BY 'P@ssw0rd';

GRANT ALL PRIVILEGES ON zabbix.* TO 'zabbix'@'localhost';

FLUSH PRIVILEGES;

EOF

импорт начальной схемы зависит от расположения пакета в ОС

zcat /usr/share/doc/zabbix-server-mysql*/create.sql.gz | mysql -uzabbix -pP@ssw0rd zabbix

sed -i 's/# DBPassword=/DBPassword=P@ssw0rd/' /etc/zabbix/zabbix_server.conf

systemctl enable --now zabbix-server zabbix-agent httpd || systemctl enable --now zabbix-server zabbix-agent apache2

DNS-запись mon.au-team.irpo добавлена в зоне DNS на HQ-SRV

доступ только для сети HQ:

iptables -A INPUT -p tcp -s 192.168.10.0/27 --dport 80 -j ACCEPT

iptables -A INPUT -p tcp -s 192.168.20.0/27 --dport 80 -j ACCEPT

iptables -A INPUT -p tcp --dport 80 -j DROP

# 4.8 Инвентаризация через Ansible

BR-SRV

mkdir -p /etc/ansible/PC-INFO

файл плейбука из Additional.iso размещается в /etc/ansible, ниже пример реализации

cat > /etc/ansible/inventory_pc.yml <<'EOF'

all:

  hosts:
  
    hq-srv:
    
      ansible_host: 192.168.10.2
      
      ansible_user: sshuser
      
      ansible_port: 2026
      
    hq-cli:
    
      ansible_host: 192.168.20.10
      
      ansible_user: sshuser
      
EOF

cat > /etc/ansible/pc-info.yml <<'EOF'

- name: Collect PC information
  
  hosts: all
  
  gather_facts: yes
  
  tasks:
  
    - name: Save host information to yaml file
      
      copy:
      
        dest: "/etc/ansible/PC-INFO/{{ inventory_hostname }}.yml"
      
        content: |
      
          hostname: {{ ansible_hostname }}
      
          fqdn: {{ ansible_fqdn }}
      
          default_ipv4: {{ ansible_default_ipv4.address | default('unknown') }}
      
      delegate_to: localhost
      
EOF

cd /etc/ansible

ansible-playbook -i inventory_pc.yml pc-info.yml

ls -l /etc/ansible/PC-INFO

cat /etc/ansible/PC-INFO/hq-srv.yml

# 4.9 Fail2Ban для SSH

HQ-SRV

apt-get install -y fail2ban

cat > /etc/fail2ban/jail.local <<'EOF'

[sshd]

enabled = true

port = 2026

filter = sshd

logpath = /var/log/secure

maxretry = 3

bantime = 60

findtime = 600

EOF

systemctl enable --now fail2ban

systemctl restart fail2ban

fail2ban-client status

fail2ban-client status sshd
