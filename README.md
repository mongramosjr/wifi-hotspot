# wifi-hotspot

deploying a wifi hotspot with captive portal using coovachilli in raspbian and ubuntu

## Requirements

In order to build a captive portal solution, we will need the follwoing:


* **Raspberry Pi** – a low cost, credit-card sized computer 

* **CoovaChilli** – a feature rich software access controller that provides a captive portal environment.

* **hostapd** – a software access point capable of turning normal network interface cards into access points and authentication servers.

* **Freeradius** – a radius server for provisioning and accounting.

* **MySQL** – a database server backing the radius server.

* **Nginx** – a proxy server.

* **daloRadius** – an advanced RADIUS web platform aimed at managing Hotspots and general-purpose ISP deployments.

## RaspberryPi

CoovaChilli needs two network interfaces, we choose eth0 and wlan0.

* eth0: The WAN interface that connect to the internet
* wlan0: The wifi interface to which client connect

## hostapd

### Install and deploy hostapd

Hostapd allows your computer to function as an Access Point (AP) WPA/WPA2 Authenticator. Since debian-based systems have pre-packaged version of hostapd, a simple command will install this package

```console
sudo apt-get install hostapd
sudo echo 'DAEMON_CONF="/etc/hostapd/hostapd.conf"' >> /etc/default/hostapd
sudo vim /etc/hostapd/hostapd.conf
```

Change the following parameters in hostapd.conf file:

```bash
interface=wlan0 # Change this to your wireless device
driver=nl80211
ssid=MyWiFiHotspot  # Change this to your SSID
#hw_mode=g
channel=1 # Enter your desired channel
macaddr_acl=0
auth_algs=1
ignore_broadcast_ssid=0
wpa=3
wpa_passphrase=0123456789password  # Change this to your wifi password
wpa_key_mgmt=WPA-PSK
wpa_pairwise=TKIP
rsn_pairwise=CCMP
```

Test and start hostapd:

```console
sudo hostapd -d /etc/hostapd/hostapd.conf
```

If all goes well, the hostapd daemon should start and not quit.

```console
sudo service hostapd restart
```

### Starting hostapd at boot time

Enable the service to start automatically at boot:

```console
sudo systemctl enable hostapd
```

If you had issues trying to start hostapd in Ubuntu desktop, run the following command:

```console
sudo nmcli radio wifi off
sudo rfkill unblock wlan
sudo systemctl start hostapd
sudo systemctl status hostapd
```
## MySQL

### Install and deploy MySQL server

Preparing to package installation. MySQL password is set at “raspbian”. Of course you can put whatever you want.

```console
sudo apt-get install -y debconf-utils
debconf-set-selections <<< 'mysql-server mysql-server/root_password password raspbian'
debconf-set-selections <<< 'mysql-server mysql-server/root_password_again password raspbian'
```
Install MySQL server.

```console
sudo apt-get install -y debhelper libssl-dev libcurl4-gnutls-dev mysql-server gcc make pkg-config iptables 
```

### Starting MySQL at boot time

Start MySQL server if it is not running.

```console
sudo systemctl start mysql
```

Make sure service start even at boot:

```console
sudo systemctl enable mysql
```

## Freeradius

FreeRadius server is also available in Ubuntu’s and debian's repo, so we can simply install it using apt-get. 

### Install and deploy freeradius server

Install required packages.

```console
sudo apt-get install -y freeradius freeradius-mysql 
```

Create radius database.

```console
mysqladmin -u root -p raspbian create radius
```

Generate database tables using MySQL schema.

```console
sudo cat /etc/freeradius/3.0/mods-config/sql/main/mysql/schema.sql | mysql -u root -p raspbian radius
```

Create MySQL radius user and set privileges on radius database:

```console
mysql -u root -p raspbian radius

GRANT ALL PRIVILEGES ON radius.* to [freeradius_db_user]@[host_address] IDENTIFIED by '[freeradius_db_password]';
```

Configure the SQL radius module:

```console
sudo vim /etc/freeradius/3.0/mods-enabled/sql
```
Change the following parameters:

```bash
driver = "rlm_sql_mysql"
dialect = "mysql"
server = "[host_address]"
port = 3306
login = "[freeradius_db_user]"
password = "[freeradius_db_password]"
radius_db = "radius"
read_clients = yes
```

Next link sql to modules available.

```console
sudo ln -s /etc/freeradius/3.0/mods-available/sql /etc/freeradius/3.0/mods-enabled/sql
```

### Testing the FreeRADIUS server 

Now test your configuration by stopping and restarting the FreeRadius in debug mode.

```console
sudo service freeradius stop
sudo freeradius -X
```

Make a connection test. For this, open another terminal and create a test user usertest with his password passwd

```console
echo "insert into radcheck (username, attribute, op, value) values ('usertest', 'Cleartext-Password', ':=', 'passwd');" | mysql -u root -praspbian radius
```

And now to test you use the command

```console
radtest usertest passwd localhost 0 testing123
```

radtest should return:

```console
Sent Access-Request Id 158 from 0.0.0.0:49930 to 127.0.0.1:1812 length 78
⋅⋅⋅User-Name = "usertest"
⋅⋅⋅User-Password = "passwd"
⋅⋅⋅NAS-IP-Address = 127.0.1.1
⋅⋅⋅NAS-Port = 0
⋅⋅⋅Message-Authenticator = 0x00
⋅⋅⋅Cleartext-Password = "passwd"
Received Access-Accept Id 158 from 127.0.0.1:1812 to 0.0.0.0:0 length 20
```

### Starting freeradius at boot time

Enable freeradius so it starts up at boot time.

```console
sudo systemctl enable freeradius
sudo systemctl start freeradius
```


## CoovaChilli

### Install first the dependencies

To make sure coova-chilli will run without any problems, we will install the dependencies first. To do so, run the following commands:

```console
sudo apt-get update
```

```console
sudo apt-get install -y -f debhelper devscripts libcurl4-gnutls-dev haserl g++ gengetopt bash-completion libtool libltdl-dev libjson-c-dev libssl-dev make cmake autoconf automake build-essential dpkg-dev
```

### Download and install the coova-chilli

Clone the project to your directory

```console
git clone https://github.com/coova/coova-chilli.git
```

Once cloned, move inside it. 

```console
cd coova-chilli
```

Then build the debian package,

```console
sudo dpkg-buildpackage -us -uc
```

After building debian package is done, install the generated .deb file:

```console
sudo dpkg -i coova-chilli_<latest_version_here>_<architecture_here>.deb
```

### Starting coova-chilli

To start chilli, run the following command

```console
sudo /etc/init.d/chilli start
```

Enable coova-chilli so it starts up at boot time.

```console
sudo systemctl enable freeradius
```

## Nginx

## daloRadius
