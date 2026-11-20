# Гайд Демо 2027 (Debian 13 / PNETLab)

## 0. Подготовка стенда (НА ВСЕХ УСТРОЙСТВАХ)
Вход: `root` / `Test123`. Отключаем CD-ROM репозиторий, чтобы пакеты ставились из интернета.

```bash
sed -i 's/^deb cdrom/#deb cdrom/' /etc/apt/sources.list
apt update
```
*(Если нет интернета на этапе подготовки: `echo "nameserver 8.8.8.8" > /etc/resolv.conf`)*

---

## МОДУЛЬ 1. НАСТРОЙКА СЕТЕВОЙ ИНФРАСТРУКТУРЫ

### 1.1. Имена устройств (FQDN)
```bash
# ISP
hostnamectl set-hostname isp.au-team.irpo; exec bash
```
```bash
# HQ-RTR
hostnamectl set-hostname hq-rtr.au-team.irpo; exec bash
```
```bash
# HQ-SRV
hostnamectl set-hostname hq-srv.au-team.irpo; exec bash
```
```bash
# HQ-CLI
hostnamectl set-hostname hq-cli.au-team.irpo; exec bash
```
```bash
# BR-RTR
hostnamectl set-hostname br-rtr.au-team.irpo; exec bash
```
```bash
# BR-FW
hostnamectl set-hostname br-fw.au-team.irpo; exec bash
```
```bash
# BR-SRV
hostnamectl set-hostname br-srv.au-team.irpo; exec bash
```

### 1.2. Таблица IP-адресации
| Устройство | Интерфейсы и IP | Шлюз |
| :--- | :--- | :--- |
| **ISP** | `ens3` (DHCP), `ens4` (`172.16.1.1/28`), `ens5` (`172.16.2.1/28`) | DHCP |
| **HQ-RTR** | `ens3` (`172.16.1.2/28`), `vlan100` (`192.168.100.1/27`), `vlan200` (`192.168.200.1/28`), `vlan999` (`192.168.99.1/29`) | `172.16.1.1` |
| **HQ-SRV** | `ens3` (`192.168.100.2/27`) | `192.168.100.1` |
| **HQ-CLI** | `ens3` (временно `192.168.200.2/28`, затем DHCP → `192.168.200.5/28`) | `192.168.200.1` |
| **BR-RTR** | `ens3` (`172.16.2.2/28`), `ens4` (`172.16.10.1/30`) | `172.16.2.1` |
| **BR-FW** | `ens3` (`172.16.10.2/30`), `ens4` (`172.16.20.1/28`) | `172.16.10.1` |
| **BR-SRV** | `ens3` (`172.16.20.2/28`) | `172.16.20.1` |

### 1.3. IP Forwarding и NAT (ISP, HQ-RTR, BR-RTR, BR-FW)
Выполнить на всех 4 маршрутизаторах:
```bash
echo "net.ipv4.ip_forward=1" > /etc/sysctl.d/sysctl.conf
sysctl --system

cat << 'EOF' > /etc/nftables.conf
#!/usr/sbin/nft -f
flush ruleset

table ip nat {
    chain postrouting {
        type nat hook postrouting priority 100; policy accept;
        meta l4proto { gre, ipip, ospf } counter return
        masquerade
    }
}

table inet filter {
    chain input { type filter hook input priority filter; }
    chain forward { type filter hook forward priority filter; }
    chain output { type filter hook output priority filter; }
}
EOF
```

### 1.4. Настройка `/etc/network/interfaces`

**ISP:**
```bash
cat << 'EOF' > /etc/network/interfaces
source /etc/network/interfaces.d/*
auto lo
iface lo inet loopback

auto ens3
iface ens3 inet dhcp

auto ens4
iface ens4 inet static
address 172.16.1.1/28

auto ens5
iface ens5 inet static
address 172.16.2.1/28

post-up nft -f /etc/nftables.conf
EOF
systemctl restart networking && ip -br a
```

**HQ-RTR (до настройки VLAN):**
```bash
cat << 'EOF' > /etc/network/interfaces
source /etc/network/interfaces.d/*
auto lo
iface lo inet loopback

auto ens3
iface ens3 inet static
address 172.16.1.2/28
gateway 172.16.1.1

auto ens4
iface ens4 inet static
address 192.168.100.1/27

auto ens5
iface ens5 inet static
address 192.168.200.1/28

auto ens6
iface ens6 inet static
address 192.168.99.1/29

post-up nft -f /etc/nftables.conf
EOF
systemctl restart networking && ip -br a
```

**HQ-SRV:**
```bash
cat << 'EOF' > /etc/network/interfaces
source /etc/network/interfaces.d/*
auto lo
iface lo inet loopback

auto ens3
iface ens3 inet static
address 192.168.100.2/27
gateway 192.168.100.1
EOF
systemctl restart networking && ip -br a
```

**HQ-CLI (временно статика до пункта 1.10):**
```bash
cat << 'EOF' > /etc/network/interfaces
source /etc/network/interfaces.d/*
auto lo
iface lo inet loopback

auto ens3
iface ens3 inet static
address 192.168.200.2/28
gateway 192.168.200.1
EOF
systemctl restart networking && ip -br a
```

**BR-RTR:**
```bash
cat << 'EOF' > /etc/network/interfaces
source /etc/network/interfaces.d/*
auto lo
iface lo inet loopback

auto ens3
iface ens3 inet static
address 172.16.2.2/28
gateway 172.16.2.1

auto ens4
iface ens4 inet static
address 172.16.10.1/30

post-up nft -f /etc/nftables.conf
EOF
systemctl restart networking && ip -br a
```

**BR-FW:**
```bash
cat << 'EOF' > /etc/network/interfaces
source /etc/network/interfaces.d/*
auto lo
iface lo inet loopback

auto ens3
iface ens3 inet static
address 172.16.10.2/30
gateway 172.16.10.1

auto ens4
iface ens4 inet static
address 172.16.20.1/28

post-up nft -f /etc/nftables.conf
EOF
systemctl restart networking && ip -br a
```

**BR-SRV:**
```bash
cat << 'EOF' > /etc/network/interfaces
source /etc/network/interfaces.d/*
auto lo
iface lo inet loopback

auto ens3
iface ens3 inet static
address 172.16.20.2/28
gateway 172.16.20.1
EOF
systemctl restart networking && ip -br a
```
*(Если слетают адреса: `systemctl restart NetworkManager && systemctl restart networking`)*

### 1.5. Локальные учетные записи и sudo

**HQ-RTR и BR-RTR** (пароль `P@ssword` / в КИМ `P@ssw0rd`):
```bash
adduser net_admin
usermod -aG sudo net_admin
echo "net_admin ALL=(ALL:ALL) NOPASSWD:ALL" >> /etc/sudoers
```

**HQ-SRV и BR-SRV** (UID `2027`, пароль `P@ssword` / в КИМ `P@ssw0rd`):
```bash
adduser --uid 2027 sshuser
usermod -aG sudo sshuser
echo "sshuser ALL=(ALL:ALL) NOPASSWD:ALL" >> /etc/sudoers
```

### 1.6. Коммутация VLAN (Open vSwitch на HQ-RTR)
```bash
apt install -y openvswitch-switch openssh-server
ovs-vsctl add-br hq-sw
ovs-vsctl add-port hq-sw ens4 tag=100
ovs-vsctl add-port hq-sw ens5 tag=200
ovs-vsctl add-port hq-sw ens6 tag=999
ovs-vsctl add-port hq-sw vlan100 tag=100 -- set interface vlan100 type=internal
ovs-vsctl add-port hq-sw vlan200 tag=200 -- set interface vlan200 type=internal
ovs-vsctl add-port hq-sw vlan999 tag=999 -- set interface vlan999 type=internal
```
Обновляем `/etc/network/interfaces` на **HQ-RTR** и перезагружаем (чтобы не задвоились IP):
```bash
cat << 'EOF' > /etc/network/interfaces
source /etc/network/interfaces.d/*
auto lo
iface lo inet loopback

auto ens3
iface ens3 inet static
address 172.16.1.2/28
gateway 172.16.1.1

auto vlan100
iface vlan100 inet static
address 192.168.100.1/27

auto vlan200
iface vlan200 inet static
address 192.168.200.1/28

auto vlan999
iface vlan999 inet static
address 192.168.99.1/29

post-up nft -f /etc/nftables.conf
post-up ip link set hq-sw up
EOF
reboot
```

### 1.7. Безопасный SSH (HQ-SRV и BR-SRV)
```bash
apt install -y openssh-server
cat << 'EOF' >> /etc/ssh/sshd_config
Port 2027
AllowUsers sshuser
MaxAuthTries 2
Banner /etc/ssh_banner
EOF
echo "Authorized access only" > /etc/ssh_banner
systemctl restart ssh
```

### 1.8. GRE-туннель (HQ-RTR и BR-RTR)
Настраиваем через `nmtui` → *Редактировать подключение* → *Добавить* → *Туннель IP*:
* **HQ-RTR:** Профиль `gre`, Устройство `tun1`, Режим `GRE`, Родительский `ens3`, Локальный `172.16.1.2`, Удаленный `172.16.2.2`, IPv4 (Вручную): `10.10.10.1/30`.
* **BR-RTR:** Профиль `gre`, Устройство `tun1`, Режим `GRE`, Родительский `ens3`, Локальный `172.16.2.2`, Удаленный `172.16.1.2`, IPv4 (Вручную): `10.10.10.2/30`.

На **обоих** роутерах:
```bash
echo "post-up ip link set gre0 up" >> /etc/network/interfaces
nmcli connection modify gre ip-tunnel.ttl 64
systemctl restart networking
```

### 1.9. Динамическая маршрутизация OSPF (HQ-RTR, BR-RTR, BR-FW)
На всех трёх устройствах:
```bash
apt install -y frr
sed -i 's/ospfd=no/ospfd=yes/' /etc/frr/daemons
systemctl enable --now frr
systemctl restart frr
```

**HQ-RTR** (`vtysh`):
```text
conf t
int gre0
 ip ospf message-digest-key 1 md5 P@ssword
 exit
router ospf
 router-id 1.1.1.1
 area 0 authentication message-digest
 network 10.10.10.0/30 area 0
 network 192.168.100.0/27 area 0
 network 192.168.200.0/28 area 0
 network 192.168.99.0/29 area 0
 exit
exit
wr
```

**BR-RTR** (`vtysh`):
```text
conf t
int gre0
 ip ospf message-digest-key 1 md5 P@ssword
 exit
int ens4
 ip ospf message-digest-key 1 md5 P@ssword
 exit
router ospf
 router-id 2.2.2.2
 area 0 authentication message-digest
 network 10.10.10.0/30 area 0
 network 172.16.10.0/30 area 0
 exit
exit
wr
```

**BR-FW** (`vtysh`):
```text
conf t
int ens3
 ip ospf message-digest-key 1 md5 P@ssword
 exit
router ospf
 router-id 3.3.3.3
 area 0 authentication message-digest
 network 172.16.10.0/30 area 0
 network 172.16.20.0/28 area 0
 exit
exit
wr
```
Делаем `reboot` на всех трёх и проверяем:
```bash
vtysh -c "show ip ospf neighbor"
```

### 1.10. DHCP-сервер (HQ-RTR) и клиент (HQ-CLI)

**HQ-RTR:**
```bash
apt install -y isc-dhcp-server
sed -i 's/INTERFACESv4=""/INTERFACESv4="vlan200"/' /etc/default/isc-dhcp-server
cat << 'EOF' > /etc/dhcp/dhcpd.conf
option domain-name "au-team.irpo";
option domain-name-servers 192.168.100.2;
default-lease-time 600;
max-lease-time 7200;
ddns-update-style none;
authoritative;

subnet 192.168.200.0 netmask 255.255.255.240 {
  range 192.168.200.2 192.168.200.14;
  option routers 192.168.200.1;
}

host hq-cli {
  option dhcp-client-identifier "hq-cli";
  fixed-address 192.168.200.5;
}
EOF
systemctl restart isc-dhcp-server
```

**HQ-CLI:**
```bash
apt install -y isc-dhcp-client
cat << 'EOF' > /etc/network/interfaces
source /etc/network/interfaces.d/*
auto lo
iface lo inet loopback

auto ens3
iface ens3 inet dhcp
EOF
echo 'send dhcp-client-identifier "hq-cli";' >> /etc/dhcp/dhclient.conf
systemctl restart networking
```
*(Если получит 2 IP — удалить старое подключение в `nmtui`)*

### 1.11. DNS-сервер (HQ-SRV)
**HQ-SRV:**
```bash
apt install -y dnsmasq
cat << 'EOF' > /etc/dnsmasq.conf
interface=ens3
domain=au-team.irpo
server=8.8.8.8
listen-address=192.168.100.2
no-resolv
no-hosts
server=/au-team.irpo/172.16.20.2

address=/hq-rtr.au-team.irpo/192.168.100.1
address=/hq-srv.au-team.irpo/192.168.100.2
address=/hq-cli.au-team.irpo/192.168.200.5
address=/br-rtr.au-team.irpo/172.16.10.1
address=/br-fw.au-team.irpo/172.16.20.1
address=/br-srv.au-team.irpo/172.16.20.2
address=/docker.au-team.irpo/172.16.2.1
address=/web.au-team.irpo/172.16.1.1

ptr-record=2.100.168.192.in-addr.arpa,hq-srv.au-team.irpo
ptr-record=2.20.16.172.in-addr.arpa,br-srv.au-team.irpo
EOF
echo "nameserver 127.0.0.1" > /etc/resolv.conf
systemctl restart dnsmasq
```

**На HQ-RTR, HQ-CLI, BR-RTR, BR-FW, BR-SRV:**
```bash
echo "nameserver 192.168.100.2" > /etc/resolv.conf
```

### 1.12. Часовой пояс (НА ВСЕХ УСТРОЙСТВАХ)
```bash
timedatectl set-timezone Asia/Krasnoyarsk
```

---

## МОДУЛЬ 2. ОРГАНИЗАЦИЯ СЕТЕВОГО АДМИНИСТРИРОВАНИЯ

### 2.1. Samba AD DC (BR-SRV) и ввод HQ-CLI

**BR-SRV** *(проверь, что в `/etc/resolv.conf` стоит `192.168.100.2`)*:
```bash
apt install -y samba smbclient winbind libnss-winbind krb5-user net-tools samba-ad-provision
# Kerberos: AU-TEAM.IRPO -> br-srv.au-team.irpo -> br-srv.au-team.irpo
mv /etc/samba/smb.conf /etc/samba/smb.conf.bak
systemctl stop smbd nmbd winbind samba-ad-dc
samba-tool domain provision --use-rfc2307 --interactive
# Везде Enter, пароль Administrator: P@ssword
```
Политика паролей, запуск и пользователи:
```bash
samba-tool domain passwordsettings set --min-pwd-length=8 --account-lockout-threshold=5 --account-lockout-duration=0 --reset-account-lockout-after=30
systemctl start smbd nmbd winbind samba-ad-dc
cp /var/lib/samba/private/krb5.conf /etc/krb5.conf
samba-tool group add hq
for i in 1 2 3 4 5; do
  samba-tool user create hquser$i P@ssword
  samba-tool group addmembers hq hquser$i
done
```

**HQ-CLI:**
```bash
apt install -y realmd sssd sssd-tools libnss-sss libpam-sss adcli packagekit krb5-user samba-common
# Kerberos: AU-TEAM.IRPO -> br-srv.au-team.irpo -> br-srv.au-team.irpo
realm join -U administrator au-team.irpo
pam-auth-update --enable mkhomedir
realm permit --all
echo '%hq@au-team.irpo ALL=(ALL) /usr/bin/cat, /usr/bin/grep, /usr/bin/id' > /etc/sudoers.d/hq-users
```

### 2.2. Служба времени Chrony

**ISP:**
```bash
apt install -y chrony
sed -i 's/^pool/#pool/' /etc/chrony/chrony.conf
cat << 'EOF' >> /etc/chrony/chrony.conf
server ntp1.vniiftri.ru iburst prefer minstratum 5
local stratum 5
allow 0.0.0.0/0
EOF
systemctl restart chronyd && chronyc sources
```

**HQ-SRV и HQ-CLI:**
```bash
apt install -y chrony
sed -i 's/^pool/#pool/' /etc/chrony/chrony.conf
echo "server 172.16.1.1 iburst" >> /etc/chrony/chrony.conf
systemctl restart chronyd && chronyc sources
```

**BR-RTR и BR-SRV:**
```bash
apt install -y chrony
sed -i 's/^pool/#pool/' /etc/chrony/chrony.conf
echo "server 172.16.2.1 iburst" >> /etc/chrony/chrony.conf
systemctl restart chronyd && chronyc sources
```

### 2.3. Ansible (BR-SRV)
Предварительно поставить `openssh-server` на **HQ-CLI, BR-FW, BR-RTR** и создать `sshuser` с правами `sudo` на **HQ-CLI** и **BR-FW**.

**BR-SRV:**
```bash
apt install -y ansible sshpass
mkdir -p /etc/ansible
printf "[defaults]\nhost_key_checking=False\n" > /etc/ansible/ansible.cfg
cat << 'EOF' > /etc/ansible/hosts
[all]
hq-rtr ansible_host=192.168.100.1 ansible_user=net_admin ansible_password=P@ssword
hq-srv ansible_host=192.168.100.2 ansible_user=sshuser ansible_password=P@ssword ansible_port=2027
hq-cli ansible_host=192.168.200.5 ansible_user=sshuser ansible_password=P@ssword
br-rtr ansible_host=172.16.10.1 ansible_user=net_admin ansible_password=P@ssword
br-fw ansible_host=172.16.10.2 ansible_user=sshuser ansible_password=P@ssword

[all:vars]
ansible_python_interpreter=/usr/bin/python3.13
ansible_become_password=P@ssword
EOF
ansible all -m ping
```

### 2.4. Веб-приложение в Docker (BR-SRV)
```bash
apt install -y docker.io docker-compose
wget [http://192.168.227.92:8081/Additional.iso](http://192.168.227.92:8081/Additional.iso)
# Извлечь Additional.iso в /root/Additional
docker image load -i /root/Additional/docker/site_latest.tar
docker image load -i /root/Additional/docker/mariadb_latest.tar
mkdir -p /root/testapp && cd /root/testapp
cp /root/Additional/compose.yaml /root/testapp/
docker compose up -d && docker ps
```

### 2.5. Веб-приложение Apache + MariaDB (HQ-SRV)
```bash
apt install -y apache2 mariadb-server mariadb-client php8.4 php8.4-mysqli
wget [http://192.168.227.92:8081/Additional.iso](http://192.168.227.92:8081/Additional.iso)
# Извлечь Additional.iso в /root/Additional
cp /root/Additional/web/dump.sql /tmp/
mysql -u root -e "CREATE DATABASE webdb; CREATE USER 'web'@'localhost' IDENTIFIED BY 'P@ssword'; GRANT ALL PRIVILEGES ON webdb.* TO 'web'@'localhost'; FLUSH PRIVILEGES;"
mysql -u root webdb < /tmp/dump.sql
cp /root/Additional/web/index.php /root/Additional/web/logo.png /var/www/html/
rm -f /var/www/html/index.html
chown -R www-data:www-data /var/www/html
chmod -R 755 /var/www/html
nano /var/www/html/index.php
# Указать: $servername="localhost"; $username="web"; $password="P@ssword"; $dbname="webdb";
systemctl restart apache2
```

### 2.6. Проброс портов DNAT (HQ-RTR и BR-RTR)

**HQ-RTR:**
```bash
cat << 'EOF' > /etc/nftables.conf
#!/usr/sbin/nft -f
flush ruleset

table ip nat {
    chain prerouting {
        type nat hook prerouting priority 0; policy accept;
        iifname "ens3" tcp dport 8080 dnat to 192.168.100.2:80
        iifname "ens3" tcp dport 2027 dnat to 192.168.100.2:2027
    }
    chain postrouting {
        type nat hook postrouting priority 100; policy accept;
        meta l4proto { gre, ipip, ospf } counter return
        masquerade
    }
}
EOF
systemctl restart nftables
```

**BR-RTR:**
```bash
cat << 'EOF' > /etc/nftables.conf
#!/usr/sbin/nft -f
flush ruleset

table ip nat {
    chain prerouting {
        type nat hook prerouting priority 0; policy accept;
        iifname "ens3" tcp dport 8080 dnat to 172.16.20.2:8080
        iifname "ens3" tcp dport 2027 dnat to 172.16.20.2:2027
    }
    chain postrouting {
        type nat hook postrouting priority 100; policy accept;
        meta l4proto { gre, ipip, ospf } counter return
        masquerade
    }
}
EOF
systemctl restart nftables
```

### 2.7. Яндекс Браузер (HQ-CLI)
Скачать `Yandex.deb` в `/root`:
```bash
apt install -y ./Yandex.deb
yandex-browser-stable --no-sandbox
```

---

## МОДУЛЬ 3. ЭКСПЛУАТАЦИЯ ОБЪЕКТОВ СЕТЕВОЙ ИНФРАСТРУКТУРЫ

### 3.1. Импорт пользователей (BR-SRV)
```bash
cd /root/Additional
bash import.sh
samba-tool user list
```

### 3.2. Обратный прокси и Web-аутентификация (ISP)
```bash
apt install -y nginx apache2-utils
htpasswd -bc /etc/nginx/.htpasswd WEB P@ssword
cat << 'EOF' > /etc/nginx/sites-available/proxy
server {
    listen 80;
    server_name web.au-team.irpo;
    auth_basic "Restricted Access";
    auth_basic_user_file /etc/nginx/.htpasswd;
    location / {
        proxy_pass [http://172.16.1.2:8080](http://172.16.1.2:8080);
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
server {
    listen 80;
    server_name docker.au-team.irpo;
    location / {
        proxy_pass [http://172.16.2.2:8080](http://172.16.2.2:8080);
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
EOF
ln -s /etc/nginx/sites-available/proxy /etc/nginx/sites-enabled/
sed -i 's/# server_names_hash_bucket_size 64;/server_names_hash_bucket_size 64;/' /etc/nginx/nginx.conf
systemctl restart nginx
```

### 3.3. Центр сертификации (HQ-SRV) и HTTPS (ISP)

**HQ-SRV** (генерация сертификатов на 30 дней):
```bash
mkdir -p /root/ca && cd /root/ca
openssl req -x509 -newkey rsa:2048 -keyout ca.key -out ca.crt -days 30 -nodes -subj "/CN=au-team.irpo"
openssl req -newkey rsa:2048 -keyout web.key -out web.csr -nodes -subj "/CN=web.au-team.irpo" -addext "subjectAltName=DNS:web.au-team.irpo"
openssl x509 -req -in web.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out web.crt -days 30 -sha256 -copy_extensions copy
openssl req -newkey rsa:2048 -keyout docker.key -out docker.csr -nodes -subj "/CN=docker.au-team.irpo" -addext "subjectAltName=DNS:docker.au-team.irpo"
openssl x509 -req -in docker.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out docker.crt -days 30 -sha256 -copy_extensions copy
```

**ISP** (подготовка SSH и папки SSL):
```bash
apt install -y openssh-server
mkdir -p /etc/nginx/ssl
echo "PermitRootLogin yes" >> /etc/ssh/sshd_config
systemctl restart ssh
```

**HQ-SRV** (копирование на ISP и HQ-CLI):
```bash
scp /root/ca/*.crt /root/ca/*.key root@172.16.1.1:/etc/nginx/ssl/
scp /root/ca/ca.crt sshuser@192.168.200.5:/tmp/
```

**ISP** (перевод Nginx на HTTPS):
```bash
cat << 'EOF' > /etc/nginx/sites-enabled/proxy
server {
    listen 443 ssl;
    server_name web.au-team.irpo;
    ssl_certificate /etc/nginx/ssl/web.crt;
    ssl_certificate_key /etc/nginx/ssl/web.key;
    auth_basic "Restricted Access";
    auth_basic_user_file /etc/nginx/.htpasswd;
    location / {
        proxy_pass [http://172.16.1.2:8080](http://172.16.1.2:8080);
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
server {
    listen 443 ssl;
    server_name docker.au-team.irpo;
    ssl_certificate /etc/nginx/ssl/docker.crt;
    ssl_certificate_key /etc/nginx/ssl/docker.key;
    location / {
        proxy_pass [http://172.16.2.2:8080](http://172.16.2.2:8080);
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header Accept-Encoding "";
        sub_filter '[http://docker.au-team.irpo](http://docker.au-team.irpo)' '[https://docker.au-team.irpo](https://docker.au-team.irpo)';
        sub_filter_once off;
    }
}
EOF
nginx -t && systemctl restart nginx
```
**HQ-CLI:** В Яндекс Браузере открыть `browser://settings/certificates` → *Локальные сертификаты* → *Установленные вами* → *Импорт* в *Доверенные* (`/tmp/ca.crt`), перезапустить браузер.

### 3.4. Мониторинг atop (HQ-SRV)
```bash
apt install -y atop
sed -i 's/^LOGINTERVAL=.*/LOGINTERVAL=420/' /etc/default/atop
systemctl enable --now atop
systemctl restart atop
```

### 3.5. Принт-сервер CUPS (HQ-SRV и HQ-CLI)

**HQ-SRV:**
```bash
apt install -y cups printer-driver-cups-pdf
nano /etc/cups/cupsd.conf
# Изменить Listen localhost:631 -> Listen 0.0.0.0:631
# Внутри <Location/> добавить Allow all
systemctl enable --now cups && systemctl restart cups
```

**HQ-CLI:**
```bash
apt install -y cups-client
lpadmin -p PDF -E -v ipp://192.168.100.2:631/printers/PDF
lpadmin -d PDF && lpstat -d
```

### 3.6. Инвентаризация через Ansible (BR-SRV)
```bash
mkdir -p /etc/ansible/PC-INFO
cp /root/Additional/playbook/get_hostname_address.yml /etc/ansible/
ansible-playbook -i /etc/ansible/hosts /etc/ansible/get_hostname_address.yml
ls -l /etc/ansible/PC-INFO/
```

### 3.7. Защита SSH fail2ban (HQ-SRV)
```bash
apt install -y fail2ban
cat << 'EOF' > /etc/fail2ban/jail.local
[sshd]
enabled = true
port = 2027
maxretry = 3
bantime = 1m
EOF
systemctl enable --now fail2ban && systemctl restart fail2ban
```

### 3.8. Резервное копирование UrBackup (HQ-SRV и HQ-CLI)

**HQ-SRV** *(при установке указать путь `/backup`)*:
```bash
wget [https://hndl.urbackup.org/Server/latest/urbackup-server_amd64.deb](https://hndl.urbackup.org/Server/latest/urbackup-server_amd64.deb) -O /tmp/urbackup-server.deb
apt install -y /tmp/urbackup-server.deb sshfs
```

**HQ-CLI:**
```bash
mkdir -p /backup && chmod 777 /backup
```

**HQ-SRV** *(монтирование хранилища и установка клиента, выбрать вариант 5)*:
```bash
systemctl stop urbackupsrv
sshfs sshuser@192.168.200.5:/backup /backup -o allow_other -o default_permissions -o uid=$(id -u urbackup) -o gid=$(id -g urbackup)
systemctl start urbackupsrv
wget "[https://hndl.urbackup.org/Client/2.5.29/UrBackup%20Client%20Linux%202.5.29.sh](https://hndl.urbackup.org/Client/2.5.29/UrBackup%20Client%20Linux%202.5.29.sh)" -O /tmp/urbackup-client.sh
bash /tmp/urbackup-client.sh
```
**Веб-интерфейс (`http://192.168.100.2:55414`):** создать админа `irpoadmin` (`P@ssword`), группу `irpo`, добавить клиента `192.168.100.2`, указать каталог `/etc`, привязать к `irpo`, запустить *Полный файловый бэкап*.

---

## МОДУЛЬ 4. ДИАГНОСТИКА И ОБСЛУЖИВАНИЕ ИНФРАСТРУКТУРЫ

### 4.1. Тест пропускной способности iperf3
*(При установке отвечать «Нет» на запуск демона)*

**HQ-SRV:**
```bash
apt install -y iperf3
iperf3 -s
```
**BR-SRV:**
```bash
apt install -y iperf3
iperf3 -c 192.168.100.2 -t 10 | tee /tmp/speedtest.txt
```

### 4.2. Планировщик Cron (HQ-SRV)
```bash
(crontab -l 2>/dev/null; echo "0 8 * * * systemctl restart dnsmasq") | crontab -
crontab -l
```

### 4.3. Автовосстановление FRR через Ansible (BR-SRV)
```bash
wget [http://192.168.227.92:8081/frr_recovery.yml](http://192.168.227.92:8081/frr_recovery.yml) -O /etc/ansible/frr_recovery.yml
ansible-playbook -i /etc/ansible/hosts /etc/ansible/frr_recovery.yml
```

### 4.4. Мониторинг процессов htop (HQ-SRV)
```bash
apt install -y htop
htop
# Внутри: F4 -> apache2 -> Enter
```

### 4.5. Анализ журнала SSH (HQ-SRV)
```bash
journalctl -u ssh | grep "Failed password" > /tmp/failed_logins.txt
cat /tmp/failed_logins.txt
```
