# follfy
follfy
2.1 Базовая настройка имен и IPv4
На каждом устройстве задается полное доменное имя. Далее на серверах и маршрутизаторах настраиваются IP-адреса согласно принятому адресному плану.
# HQ-SRV
hostnamectl set-hostname hq-srv.au-team.irpo
nmcli con mod eth0 ipv4.addresses 192.168.10.2/27 ipv4.gateway 192.168.10.1 ipv4.dns 192.168.10.2 ipv4.method manual
nmcli con up eth0

# BR-SRV
hostnamectl set-hostname br-srv.au-team.irpo
nmcli con mod eth0 ipv4.addresses 192.168.30.2/28 ipv4.gateway 192.168.30.1 ipv4.dns 192.168.10.2 ipv4.method manual
nmcli con up eth0

# HQ-CLI
hostnamectl set-hostname hq-cli.au-team.irpo
# адрес получает по DHCP после настройки HQ-RTR

# ISP
hostnamectl set-hostname isp.au-team.irpo
ip addr add 172.16.1.1/28 dev eth1
ip addr add 172.16.2.1/28 dev eth2
ip link set eth1 up
ip link set eth2 up

# HQ-RTR
hostnamectl set-hostname hq-rtr.au-team.irpo
ip addr add 172.16.1.2/28 dev eth0
ip link set eth0 up
ip route add default via 172.16.1.1

# BR-RTR
hostnamectl set-hostname br-rtr.au-team.irpo
ip addr add 172.16.2.2/28 dev eth0
ip addr add 192.168.30.1/28 dev eth1
ip link set eth0 up
ip link set eth1 up
ip route add default via 172.16.2.1
