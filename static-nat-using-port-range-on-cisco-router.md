
---
# Static NAT using Port Range on Cisco Router
---

## Video Walkthroughs
Static NAT using Port Range on Cisco Router - 

## Scenario:

![alt text](https://github.com/hackd-art/networking-tips-and-tricks/blob/main/static-nat-using-port-range-on-cisco-router.png)

1. 3rd Party Public Server (100.100.100.100) should be able to access the server box inside customer network
2. Static NAT (Private: 10.11.12.14 - Public: 64.65.66.67) should be implemented but should only allow below ports inbound:

  TCP 5001<br>
  TCP/UDP 5002<br>
  UDP 5003<br>
  UDP 10000-10999<br>

## Create and Configure VM instance

```shell
# create VM image in GCP shell
gcloud compute images create nested-ubuntu-focal --source-image-family=ubuntu-2004-lts --source-image-project=ubuntu-os-cloud --licenses https://www.googleapis.com/compute/v1/projects/vm-options/global/licenses/enable-vmx
```

#### VM Creation Notes:
- VM permission to all Cloud API
- Firewall Rule to allow source 0.0.0.0/0 to all instances on the network port 22,80

```shell
sudo -i
passwd
wget -O - https://www.eve-ng.net/focal/install-eve.sh | bash -i
apt update && apt upgrade -y
reboot

# Ctrl+C
sudo -i
# enter root password 2x then set everything to default

# Wait for 5 min then retry
# After prompt is back, manually turn off VM then turn it back on

# It takes around 3-5 minutes after VM is powered on before it will be accessible
```

## Enable HTTPS Connectivity to Eve-NG

```shell
sudo a2enmod ssl

sudo openssl req -x509 -nodes -days 3650 -newkey rsa:2048 -keyout /etc/ssl/private/apache-selfsigned.key -out /etc/ssl/certs/apache-selfsigned.crt

cat << EOF > /etc/apache2/sites-enabled/default-ssl.conf
<IfModule mod_ssl.c>
    <VirtualHost _default_:443>
        ServerAdmin webmaster@localhost
        DocumentRoot /opt/unetlab/html/
        ErrorLog /opt/unetlab/data/Logs/ssl-error.log
        CustomLog /opt/unetlab/data/Logs/ssl-access.log combined
        Alias /Exports /opt/unetlab/data/Exports
        Alias /Logs /opt/unetlab/data/Logs
        SSLEngine on
        SSLCertificateFile    /etc/ssl/certs/apache-selfsigned.crt
        SSLCertificateKeyFile /etc/ssl/private/apache-selfsigned.key
        <FilesMatch "\.(cgi|shtml|phtml|php)$">
                SSLOptions +StdEnvVars
        </FilesMatch>
        <Directory /usr/lib/cgi-bin>
                SSLOptions +StdEnvVars
        </Directory>
        <Location /html5/>
                Order allow,deny
                Allow from all
                ProxyPass http://127.0.0.1:8080/guacamole/ flushpackets=on
                ProxyPassReverse http://127.0.0.1:8080/guacamole/
        </Location>

        <Location /html5/websocket-tunnel>
                Order allow,deny
                Allow from all
                ProxyPass ws://127.0.0.1:8080/guacamole/websocket-tunnel
                ProxyPassReverse ws://127.0.0.1:8080/guacamole/websocket-tunnel
        </Location>
    </VirtualHost>
</IfModule>
EOF

/etc/init.d/apache2 restart

# to disable SSL
sudo a2dismod ssl
/etc/init.d/apache2 restart

# to disable HTTP access
nano /etc/apache2/ports.conf
/etc/init.d/apache2 restart
```


## Configuring DHCP and Internet Access for Lab Nodes

#### Internet Access

```shell
vim /etc/init.d/guacd

# insert somewhere after #!/bin/sh
ip address add 192.168.255.1/24 dev pnet9
echo 1 > /proc/sys/net/ipv4/ip_forward
iptables -t nat -A POSTROUTING -o pnet0 -s 192.168.255.0/24 -j MASQUERADE
```

#### DHCP Service

```shell
# install dhcp server
apt update
apt install isc-dhcp-server -y

# configure interface
vim /etc/default/isc-dhcp-server

INTERFACESv4="pnet9"

# configure dhcp service
vim /etc/dhcp/dhcpd.conf

default-lease-time 600;
max-lease-time 7200;

authoritative;

subnet 192.168.255.0 netmask 255.255.255.0 {
  range 192.168.255.101 192.168.255.200;
  option domain-name-servers 8.8.8.8;
  option routers 192.168.255.1;
}

# remove dhcp lease files
rm -rf /var/lib/dhcp/*

# start and enable dhcp service
systemctl start isc-dhcp-server
systemctl enable isc-dhcp-server
```
