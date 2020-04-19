# IoT device gateway

## Requirements
- Raspberry PI is connected to home central router with ethernet.
- Home IoT devices are connected to Raspberry PI WiFi network.
- Raspberry PI WiFi newwork has restrictions: WiFi connected devices can only access to Raspberry PI services but not having direct internet access or direct access to home network.
- Raspberry PI device can access to internet and home network.

```
WiFi--->---RaspberryPI---X---HomeRouter--->---Internet
            - iptables           |
                |                |
                |                |
                `---->------>----`
```

# Install Ubuntu to Raspberry PI

1. Download Ubuntu Server 18.04. https://ubuntu.com/download/raspberry-pi/thank-you?version=18.04.4&architecture=armhf+raspi3

2. Write image to SD card with `Win32DiskImager` utility. https://ubuntu.com/tutorials/create-an-ubuntu-image-for-a-raspberry-pi-on-windows#2-on-your-windows-machine

3. Start Raspberry PI with SD card.

4. Login with `ubuntu/ubuntu`. First login requires to change password. Set password `<DEFINE_PASSWORD>`. After password has been changed raspberry logout your session.

5. Login again with `ubuntu/<DEFINE_PASSWORD>`.

6. `sudo su`

7. `apt-get update`

8. `apt-get upgrade`

9. `apt-get install ifupdown iw hostapd dnsmasq isc-dhcp-server iptables`

10. `nano /etc/hostapd/hostapd.conf`
```
interface=wlan0
driver=nl80211
ssid=iot-gateway
hw_mode=g
channel=1
macaddr_acl=0
auth_algs=1
ignore_broadcast_ssid=0
wpa=2
wpa_passphrase=<DEFINE_PASSWORD>
wpa_key_mgmt=WPA-PSK
wpa_pairwise=TKIP
rsn_pairwise=CCMP
```

11. `nano /etc/default/hostapd`
```
DAEMON_CONF="/etc/hostapd/hostapd.conf"
```

12. `systemctl unmask hostapd`
```
ubuntu@ubuntu:~$ sudo systemctl unmask hostapd
Removed /etc/systemd/system/hostapd.service.
ubuntu@ubuntu:~$
```
13. `systemctl enable hostapd`
```
ubuntu@ubuntu:~$ sudo systemctl enable hostapd
Synchronizing state of hostapd.service with SysV service script with /lib/systemd/systemd-sysv-install.
Executing: /lib/systemd/systemd-sysv-install enable hostapd
ubuntu@ubuntu:~$
```

14. `systemctl start hostapd`
```
ubuntu@ubuntu:~$ sudo systemctl start hostapd
ubuntu@ubuntu:~$
```

15. `nano /etc/default/isc-dhcp-server`
```
# Path to dhcpd's config file (default: /etc/dhcp/dhcpd.conf).
#DHCPDv4_CONF=/etc/dhcp/dhcpd.conf
#DHCPDv6_CONF=/etc/dhcp/dhcpd6.conf

INTERFACESv4="wlan0"
INTERFACESv6=""
```

16. `nano /etc/dhcp/dhcpd.conf`
```
#option domain-name “example.org”;
#option domain-name-servers ns1.example.org, ns2.example.org;

default-lease-time 600;
max-lease-time 7200;

authoritative;

subnet 10.10.0.0 netmask 255.255.255.0 {
  range 10.10.0.2 10.10.0.16;
  option domain-name-servers 8.8.4.4, 8.8.8.8;
  option routers 10.10.0.1;
}
```

17. `nano /etc/network/interfaces`
```
auto wlan0
iface wlan0 inet static
address 10.10.0.1
netmask 255.255.255.0
```

18. Run following commands.
```
ifdown wlan0 && sudo ifup wlan0
service isc-dhcp-server stop
service hostapd stop
service isc-dhcp-server start
service hostapd start
systemctl restart isc-dhcp-server.service
service isc-dhcp-server status
```

Result:
```
ubuntu@ubuntu:~$ sudo ifdown wlan0 && sudo ifup wlan0
ifdown: interface wlan0 not configured
ubuntu@ubuntu:~$ sudo service isc-dhcp-server stop
ubuntu@ubuntu:~$ sudo service hostapd stop
ubuntu@ubuntu:~$ sudo service isc-dhcp-server start
ubuntu@ubuntu:~$ sudo service hostapd start
ubuntu@ubuntu:~$ sudo systemctl restart isc-dhcp-server.service
ubuntu@ubuntu:~$ sudo service isc-dhcp-server status
● isc-dhcp-server.service - ISC DHCP IPv4 server
   Loaded: loaded (/lib/systemd/system/isc-dhcp-server.service; enabled; vendor preset: enabled)
   Active: active (running) since Sat 2020-04-11 08:02:09 UTC; 1s ago
     Docs: man:dhcpd(8)
 Main PID: 1675 (dhcpd)
    Tasks: 1 (limit: 1976)
   CGroup: /system.slice/isc-dhcp-server.service
           └─1675 dhcpd -user dhcpd -group dhcpd -f -4 -pf /run/dhcp-server/dhcpd.pid -cf /etc/dhcp/dhcpd.conf

Apr 11 08:02:09 ubuntu dhcpd[1675]: Sending on   LPF/wlan0/b8:27:eb:d0:4b:0e/10.10.0.0/24
Apr 11 08:02:09 ubuntu dhcpd[1675]:
Apr 11 08:02:09 ubuntu dhcpd[1675]: No subnet declaration for eth0 (192.168.1.120).
Apr 11 08:02:09 ubuntu dhcpd[1675]: ** Ignoring requests on eth0.  If this is not what
Apr 11 08:02:09 ubuntu dhcpd[1675]:    you want, please write a subnet declaration
Apr 11 08:02:09 ubuntu dhcpd[1675]:    in your dhcpd.conf file for the network segment
Apr 11 08:02:09 ubuntu dhcpd[1675]:    to which interface eth0 is attached. **
Apr 11 08:02:09 ubuntu dhcpd[1675]:
Apr 11 08:02:09 ubuntu dhcpd[1675]: Sending on   Socket/fallback/fallback-net
Apr 11 08:02:09 ubuntu dhcpd[1675]: Server starting service.
ubuntu@ubuntu:~$
```

19. Connect your client device to `iot-gateway` wifi network.

20. Check DHCP IP leases from client device or from Raspberry PI DHCP. `dhcp-lease-list`

Result:
```
ubuntu@ubuntu:~$ sudo dhcp-lease-list
To get manufacturer names please download http://standards.ieee.org/regauth/oui/oui.txt to /usr/local/etc/oui.txt
Reading leases from /var/lib/dhcp/dhcpd.leases
MAC                IP              hostname       valid until         manufacturer
===============================================================================================
00:28:f8:60:d9:e3  10.10.0.2       LAPTOP         2020-04-11 08:13:12 -NA-
ubuntu@ubuntu:~$
```

21. iptables overview. https://www.systutorials.com/port-forwarding-using-iptables/
```
PACKET IN
    |
PREROUTING--[routing]-->--FORWARD-->--POSTROUTING-->--OUT
 - nat (dst)   |           - filter      - nat (src)
               |                            |
               |                            |
              INPUT                       OUTPUT
              - filter                    - nat (dst)
               |                          - filter
               |                            |
               `----->-----[app]----->------'
```

22. `nano /etc/sysctl.conf`
```
net.ipv4.ip_forward=1
```

23. `sysctl --system`

24. `apt-get install iptables-persistent iptstate`. https://www.phildev.net/iptstate/

25. Run command `iptstate`.

26. Iptables setup. https://www.thegeekstuff.com/2011/06/iptables-rules-examples/

Create rules file: `nano rules.sh`

a) Iptables NAT forwarding. Allow wlan0 devices internet access.
```
#!/bin/sh

iptables -F
iptables -t nat -F
iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE

iptables -t nat -v -L POSTROUTING -n --line-number
iptables -L FORWARD --line-number
iptables -L -v --line-numbers

iptables-save  > /etc/iptables/rules.v4
ip6tables-save > /etc/iptables/rules.v6

iptables-restore  < /etc/iptables/rules.v4
ip6tables-restore < /etc/iptables/rules.v6
```

--OR--

b) Allow Internal Network to External network. Deny wlan0 devices internet access but allow internal network access.
```
#!/bin/sh

iptables -F
iptables -t nat -F
#iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
##iptables -A INPUT -i wlan0 -j ACCEPT
##iptables -A FORWARD -i wlan0 -j DROP
iptables -t filter -P FORWARD -i wlan0 DROP
iptables -t filter -A FORWARD -i wlan0 -o eth0 -j ACCEPT

iptables -t nat -v -L POSTROUTING -n --line-number
iptables -L FORWARD --line-number
iptables -L -v --line-numbers

iptables-save  > /etc/iptables/rules.v4
ip6tables-save > /etc/iptables/rules.v6

iptables-restore  < /etc/iptables/rules.v4
ip6tables-restore < /etc/iptables/rules.v6
```
> Remember following rule only deny direct Internet access but there are no restrictions to access gateway device services. Recommend to add port filtering.

c) Make the script file executable. `chmod +x rules.sh`

d) Run `./rules.sh` to install iptables rules.

25. Verify iptables.

Gateway device:
- `ping google.com`
- `ping 10.10.0.2`

Wlan client device:
- Goto 10.10.0.2 device and `ping 10.10.0.1`

> Congratulations you have now installed gateway device having internal and external network.

# Optional usefull services to install

## FTP Server install
https://www.techrepublic.com/article/how-to-quickly-setup-an-ftp-server-on-ubuntu-18-04/

1. sudo su

2. `apt-get install vsftpd`

3. Start `systemctl start vsftpd`.

4. Enable the service `systemctl enable vsftpd`.
```
root@ubuntu:/home/ubuntu# systemctl enable vsftpd
Synchronizing state of vsftpd.service with SysV service script with /lib/systemd/systemd-sysv-install.
Executing: /lib/systemd/systemd-sysv-install enable vsftpd
root@ubuntu:/home/ubuntu#
```

5. `useradd -m ftpuser`

6. `passwd ftpuser`

7. FTP user: `ftpuser/<DEFINE_PASSWORD>`

8. `mv /etc/vsftpd.conf /etc/vsftpd.conf.orig`

9. `nano /etc/vsftpd.conf`
```
listen=NO
listen_ipv6=YES
anonymous_enable=NO
local_enable=YES
write_enable=YES
local_umask=022
dirmessage_enable=YES
use_localtime=YES
xferlog_enable=YES
connect_from_port_20=YES
chroot_local_user=YES
secure_chroot_dir=/var/run/vsftpd/empty
pam_service_name=vsftpd
rsa_cert_file=/etc/ssl/certs/ssl-cert-snakeoil.pem
rsa_private_key_file=/etc/ssl/private/ssl-cert-snakeoil.key
ssl_enable=NO
pasv_enable=Yes
pasv_min_port=10000
pasv_max_port=10100
allow_writeable_chroot=YES
```

10. `service vsftpd status`
```
root@ubuntu:/home/ftpuser# service vsftpd status
● vsftpd.service - vsftpd FTP server
   Loaded: loaded (/lib/systemd/system/vsftpd.service; enabled; vendor preset: enabled)
   Active: active (running) since Sun 2020-04-19 14:32:27 UTC; 21min ago
  Process: 7558 ExecStartPre=/bin/mkdir -p /var/run/vsftpd/empty (code=exited, status=0/SUCCESS)
 Main PID: 7559 (vsftpd)
    Tasks: 3 (limit: 1976)
   CGroup: /system.slice/vsftpd.service
           ├─7559 /usr/sbin/vsftpd /etc/vsftpd.conf
           ├─7654 /usr/sbin/vsftpd /etc/vsftpd.conf
           └─7656 /usr/sbin/vsftpd /etc/vsftpd.conf

Apr 19 14:32:27 ubuntu systemd[1]: Starting vsftpd FTP server...
Apr 19 14:32:27 ubuntu systemd[1]: Started vsftpd FTP server.
root@ubuntu:/home/ftpuser#
```

## Install NodeJS

1. Install NodeJS. https://github.com/nodesource/distributions#debinstall.

`curl -sL https://deb.nodesource.com/setup_13.x | sudo -E bash -`
`sudo apt-get install -y nodejs`

2. `node -v`
```
ubuntu@ubuntu:~$ node -v
v13.12.0
ubuntu@ubuntu:~$
```

3. `npm -v`
```
ubuntu@ubuntu:~$ npm -v
6.14.4
ubuntu@ubuntu:~$
```

4. https://tecadmin.net/install-latest-nodejs-npm-on-ubuntu/

5. `npm install express`

6. `nano server.js`. Add following code to file.
```
var express = require('express');
var app = express();
var bodyParser = require('body-parser');
app.use(bodyParser.urlencoded({ extended: true }));
app.use(bodyParser.json());
var port = process.env.PORT || 8080;
const router = express.Router();
router.get('/', function(req, res) {
  console.log('Request reveiced.');
  res.json({ message: 'Hello world!' });
});

app.use('/api', router);
app.listen(port);
console.log('Listening on port ' + port);
```

7. `node server.js`
```
root@ubuntu:/home/ubuntu# node server.js
Listening on port 8080
^C
root@ubuntu:/home/ubuntu#
```

8. Test REST api from wlan0 device with url `http://10.10.0.1:8080/api`
```
{"message":"Hello world!"}
```

## Install Python

1. https://linuxize.com/post/how-to-install-python-3-8-on-ubuntu-18-04/

2. `apt-get install software-properties-common`

3. `add-apt-repository ppa:deadsnakes/ppa`. When prompted press `Enter` to continue.

4. `apt-get install python3.8`

5. Verify python installation `python3.8 --version`
```
root@ubuntu:/home/ubuntu# python3.8 --version
Python 3.8.2
root@ubuntu:/home/ubuntu#
```

## MQTT (Message Queuing Telemetry Transport)
- https://www.digitalocean.com/community/tutorials/how-to-install-and-secure-the-mosquitto-mqtt-messaging-broker-on-ubuntu-18-04
- http://www.eclipse.org/paho/

## Node-Red
- https://nodered.org/

## OpenHab
- https://www.openhab.org/
