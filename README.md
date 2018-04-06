# wifi-hotspot

deploying a wifi hotspot with captive portal using coovachilli in raspbian and ubuntu

## Requirements

In order to build a captive portal solution, we will need the following:

* **Raspbian/Ubuntu**  – A Linux distribution. In this article, we will be using Raspbian release 2017-11-29 or Ubuntu 16.04. Later versions should work fine.

* **Raspberry Pi** – a low cost, credit-card sized computer 

* **CoovaChilli** – a feature rich software access controller that provides a captive portal environment.

* **hostapd** – a software access point capable of turning normal network interface cards into access points and authentication servers.

* **FreeRADIUS** – a radius server for provisioning and accounting.

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

Preparing to package installation. MySQL password is set at “**raspbian**”. Of course you can put whatever you want.

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

## FreeRADIUS

FreeRadius server is also available in Ubuntu’s and debian's repo, so we can simply install it using apt-get. 

### Install and deploy FreeRADIUS server

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

### Configure FreeRADIUS

Edit client definition.

```console
sudo vim /etc/freeradius/3.0/clients.conf
```

The shared secret use to "encrypt" and "sign" packets between the NAS and FreeRADIUS. This secret must be changed from the default, otherwise it is not a secret anymore!

```bash
client localhost { 
	... 
	secret = radtesting123 # Change this to your shared secret
	...
}
```

Configure the SQL radius module.

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


Configure the default virtual server under sites-available:

```console
sudo vim /etc/freeradius/3.0/sites-available/default
```

Change the following:

```bash
'-sql' to sql
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
radtest usertest passwd localhost 0 radtesting123
```

radtest should return:

```console
Sent Access-Request Id 158 from 0.0.0.0:49930 to 127.0.0.1:1812 length 78
    User-Name = "usertest"
    User-Password = "passwd"
    NAS-IP-Address = 127.0.1.1
    NAS-Port = 0
    Message-Authenticator = 0x00
    Cleartext-Password = "passwd"
Received Access-Accept Id 158 from 127.0.0.1:1812 to 0.0.0.0:0 length 20
```

### Starting FreeRADIUS at boot time

Enable freeradius so it starts up at boot time.

```console
sudo systemctl enable freeradius
sudo systemctl start freeradius
```


## CoovaChilli

### Install first the dependencies

To make sure CoovaChilli will run without any problems, we will install the dependencies first. To do so, run the following commands:

```console
sudo apt-get update
```

```console
sudo apt-get install -y -f debhelper devscripts libcurl4-gnutls-dev haserl g++ gengetopt bash-completion libtool libltdl-dev libjson-c-dev libssl-dev make cmake autoconf automake build-essential dpkg-dev
```

### Download and install the CoovaChilli

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

### Starting CoovaChilli

To start CoovaChilli, run the following command

```console
sudo /etc/init.d/chilli start
```

Enable CoovaChilli so it starts up at boot time.

```console
sudo systemctl enable chilli
```

### Configure CoovaChilli
All configuration files are located under /etc/chilli. You will need to create a config file with your sites modifications.

```console
sudo cp -v /etc/chilli/defaults /etc/chilli/config
sudo vim /etc/chilli/config
```

Change the following parameters to match your environment.

```bash
###
#   Local Network Configurations
HS_WANIF=eth0            # WAN Interface toward the Internet
HS_LANIF=wlan0            # Subscriber Interface for client devices
HS_NETWORK=10.10.10.0      # HotSpot Network (must include HS_UAMLISTEN)
HS_NETMASK=255.255.255.0   # HotSpot Network Netmask
HS_UAMLISTEN=10.10.10.1    # HotSpot IP Address (on subscriber network)
HS_UAMPORT=3990            # HotSpot UAM Port (on subscriber network)
HS_UAMUIPORT=4990          # HotSpot UAM "UI" Port (on subscriber network, for embedded portal)

... 

# OpenDNS Servers
HS_DNS1=10.10.10.1         # Set this to be the HotSpot IP Address if dnsmasq or BIND is running locally, otherwise use other DNS server
HS_DNS2=208.67.220.220

...

###
#   HotSpot settings for simple Captive Portal
HS_NASID=nas01			# NAS ID
HS_RADIUS=localhost		# FreeRadius server
HS_RADIUS2=localhost		# FreeRadius server

HS_UAMALLOW=10.10.10.0/24

HS_RADSECRET=radtesting123    	# Set to be your RADIUS shared secret
HS_UAMSECRET=uamtesting123    	# Set to be your UAM secret

...

###
#   Firewall issues
#   
#   Uncomment the following to add ports to the allowed local ports list
#   The up.sh script will allow these local ports to be used, while the default
#   is to block all unwanted traffic to the tun/tap. 
HS_TCP_PORTS="80 443"

...

###
#   Standard configurations
HS_ADMUSR=coovachillispot
HS_ADMPWD=coovachillispot
```
Edit the file /etc/chilli/up.sh with execution permission:

```bash
#Enable NAT
iptables -I POSTROUTING -t nat -o $HS_WANIF -j MASQUERADE
#Others iptables rules when chilli come up
```

### Test it out

Restart CoovaChilli for the latest changes to be effected.

```console
sudo /etc/init.d/chilli stop
sudo /etc/init.d/chilli start
```

## Nginx

Nginx is one of the most popular web servers in the world and is responsible for hosting some of the largest and highest-traffic sites on the internet. It is more resource-friendly than Apache in most cases and can be used as a web server or a reverse proxy.

Nginx is available in Ubuntu's and debian's default repositories, so the installation is rather straight forward.

```console
sudo apt-get install -y php-mysql php-pear php-gd php-db php-fpm libgd2-xpm-dev libpcrecpp0v5 libxpm4 nginx php-memcache
```

### Starting nginx at boot time

enable the service to start up at boot

```console
sudo systemctl enable nginx
```

## daloRADIUS

### Download daloRADIUS

We configure daloRADIUS hotspotlogin into your desired webroot folder (/var/www/hotspot.example.com).

```console
wget -c https://github.com/lirantal/daloradius/archive/master.zip -O daloradius.master.zip
unzip daloradius.master.zip
```

Create webroot folder for the hotspot domain name, copy the captive portal login page and move daloradius in that webroot folder

```console
sudo mkdir /var/www/hotspot.example.com
cp -r daloradius-master/contrib/chilli/portal2/hotspotlogin/* /var/www/hotspot.example.com/
sudo mv daloradius-master /var/www/hotspot.example.com/daloradius
```

Modify the MySQL database for FreeRadius
```console
mysql -u root -praspbian radius < /var/www/hotspot.example.com/daloradius/contrib/db/fr2-mysql-daloradius-and-freeradius.sql
```

Re-insert our test user.
```console
echo "insert into radcheck (username, attribute, op, value) values ('usertest', 'Cleartext-Password', ':=', 'passwd');" | mysql -u root -praspbian radius
```

### Create a server block in nginx

Create the server block file that will tell Nginx on how to process the hotspot login web app and daloRADIUS.

```console
sudo vim /etc/nginx/sites-available/hotspot.example.com
```

Copy the following lines and paste it into the server block file

```bash
server {
	# Redirect all HTTP traffic to HTTPS since daloRADIUS requires an HTTPS connection
	listen 10.10.10.1:80 default_server; # Change this to match your HotSpot IP address
	server_name hotspot.example.com; # Change this to your domain name
	return 301 https://$server_name$request_uri;
}

server {
	listen 10.10.10.1:443 ssl default_server; # Change this to match your HotSpot IP address
        server_name hotspot.example.com;  # Change this to your domain name

        # Self signed certs generated by the ssl-cert package
        # Don't use them in a production server!
        include snippets/snakeoil.conf;

	# Replace your signed ssl certificate 
	# ssl_certificate /etc/ssl/certs/<public_key_of_ssl_certificate_here>.pem;
	# ssl_certificate_key /etc/ssl/private/<private_key_of_ssl_certificate_here>.key;


	root /var/www/hotspot.example.com; # Change this to match the folder of your hotspot app
	index hotspotlogin.php index.php index.phtml index.html index.htm;

	location / {
		# First attempt to serve request as file, then
		# as directory, then fall back to displaying a 404.
		try_files $uri $uri/ /index.php?$args /hotspotlogin.php?$args $uri/ =404;
	}
    
	location /daloradius {
		alias /var/www/hotspot.example.com/daloradius;
	}

	location ~ ^/daloradius(.+\.php)$ {
		alias /var/www/hotspot.example.com/daloradius;
		fastcgi_index index.php;
		fastcgi_param SCRIPT_FILENAME /var/www/hotspot.example.com/daloradius$1;
		include fastcgi_params;
		fastcgi_pass unix:/run/php/php7.1-fpm.sock; # check the php-fpm.conf configuration regarding the path of listen directive
	}

	location ~ ^/daloradius/(.*\.(eot|otf|woff|ttf|css|js|jpg|jpeg|gif|png|ico|zip|tgz|gz|rar|bz2|xls|tar|bmp))$ {
		alias /var/www/hotspot.example.com/daloradius/$1;
		expires 30d;
		log_not_found off;
		access_log off;
	}

	location ~ \.php$ {
		include snippets/fastcgi-php.conf;
		fastcgi_pass unix:/run/php/php7.1-fpm.sock; # check the php-fpm.conf configuration listen directive
	}
}
```

That is all we need for a basic configuration. Save and close the file to exit.

### Enable the server block and restart nginx

Now that we have our server block file, we need to enable them. We can do this by creating symbolic links from these files to the sites-enabled directory, which Nginx reads from during startup.

We can create these links by typing:

```console
sudo ln -s /etc/nginx/sites-available/hotspot.example.com /etc/nginx/sites-enabled/
```

Next, test to make sure that there are no syntax errors in any of your Nginx files:

```console
sudo nginx -t
```
If no problems were found, restart Nginx to enable your changes:

```console
sudo systemctl restart nginx
```


### Modify configuration in CoovaChilli and in the captive portal login page


Edit /etc/chilli/config.

```console
sudo vi /etc/chilli/config
```

```bash
#   Use HS_UAMFORMAT to define the actual captive portal url.
HS_UAMFORMAT=https://\$HS_UAMLISTEN/hotspotlogin.php
```


Edit /var/www/hotspot.example.com/hotspotlogin.php

```console
sudo vi /var/www/hotspot.example.com/hotspotlogin.php
```

```bash
# Shared secret used to encrypt challenge with. Prevents dictionary attacks.
# You should change this to your own shared secret.
$uamsecret = "uamtesting123"; # Change this to match the coovachilli config directive HS_UAMSECRET
```

### Restart the captive portal

Let’s now start the hostapd, nginx and CoovaChilli. And try accessing captive portal from our web browser.

```console
sudo systemctl stop hostapd
sudo systemctl stop nginx
sudo systemctl stop chilli

sudo systemctl start chilli
sudo systemctl start nginx
sudo systemctl start hostapd
```


## Testing your captive portal

Using a wireless client like smartphone or laptop, click the Wi-Fi icon in your laptop's Menu bar, or open the Settings app and tap Wi-Fi on an iPad or Android phone, and choose the Wi-Fi hotspot.

If your followed the above steps, then you will be redirected to the captive portal page. Use the access credential of test user created earlier.

```bash
username: usertest
password: passwd
```

That should be it. You should now be able to browse the internet on your laptop or smartphone.

##  Bonus: Dnsmasq

Dnsmasq is a small, open-source application that’s designed to provide DNS and, optionally, Dynamic Host Configuration Protocol (DHCP), addressing to a small network. It also supports IPv4 and IPv6 static and dynamic DHCP.

Use debian or ubuntu default package installation routines.

```console
sudo apt-get install dnsmasq
```

That’s it. Dnsmasq should now be running.

Add configuration for resolving domain name for **hotspot.example.com** to an ip address **10.10.10.1** and define nameservers

```console
sudo vi /etc/dnsmasq.conf
```

Add the following line:

```bash
no-resolv
server=208.67.222.222
server=208.67.220.220
server=8.8.8.8
server=8.8.4.4

address=/hotspot.example.com/10.10.10.1
```

Restart the dnsmasq service to apply the changes

```console
sudo service dnsmasq restart
```

### Restart the captive portal
Start the apps in the folowing sequence:
```console
sudo systemctl stop dnsmasq
sudo systemctl stop nginx
sudo systemctl stop chilli
sudo systemctl stop hostapd

sudo systemctl start hostapd
sudo systemctl start chilli
sudo systemctl start nginx
sudo systemctl start dnsmasq
```

You may  now reboot the system and make sure CoovaChilli started up fine.