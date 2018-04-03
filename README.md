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

Hostapd allows your computer to function as an Access Point (AP) WPA/WPA2 Authenticator. Since debian-based systems have pre-packaged version of hostapd, a simple command will install this package

```
sudo apt-get install hostapd
sudo echo 'DAEMON_CONF="/etc/hostapd/hostapd.conf"' >> /etc/default/hostapd
sudo vim /etc/hostapd/hostapd.conf
```

Change the following parameters in hostapd.conf file:

```
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

```
sudo hostapd -d /etc/hostapd/hostapd.conf
```

If all goes well, the hostapd daemon should start and not quit.

```
sudo service hostapd restart
```

Enable the service to start automatically at boot:

```
sudo systemctl enable hostapd
```

If you had issues trying to start hostapd in Ubuntu desktop, run the following command:

```
sudo nmcli radio wifi off
sudo rfkill unblock wlan
sudo systemctl start hostapd
sudo systemctl status hostapd
```

## Freeradius

## MySQL

## CoovaChilli

## Nginx

## daloRadius
