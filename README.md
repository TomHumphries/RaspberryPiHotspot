# RaspberryPiHotSpot
This turns a Raspberry Pi into a WiFi hotspot that is not connected to the internet, hosting a single website.

When connecting to the Raspberry Pi it should redirect you to the website hosted on it.

This is done by mimicking a signin step, which opens a browser window on Windows, Android, and Apple devices when connecting to the WiFi.


**This guide continues on from the steps in the guide "[Setting up a Raspberry Pi as a Wireless Access Point](https://www.raspberrypi.org/documentation/configuration/wireless/access-point.md)" from the RaspberryPi.org website.**


# Setting up a Raspberry Pi as an open Wireless Access Point with a Captive Portal

Before proceeding, please ensure your Raspberry Pi is [up to date](../../raspbian/updating.md) and rebooted.

## Setting up a Raspberry Pi as an open access point in a standalone network (NAT)


The Raspberry Pi can be used as a wireless access point, running a standalone network. This can be done using the inbuilt wireless features of the Raspberry Pi 3 or Raspberry Pi Zero W, or by using a suitable USB wireless dongle that supports access points.

Note that this documentation was tested on a Raspberry Pi Zero W, and it is possible that some USB dongles may need slight changes to their settings. If you are having trouble with a USB wireless dongle, please check the forums.

To add a Raspberry Pi-based access point to an existing network, see [this section](#internet-sharing).

In order to work as an access point, the Raspberry Pi will need to have access point software installed, along with DHCP server software to provide connecting devices with a network address.

To create an access point, we'll need DNSMasq and HostAPD. Install all the required software in one go with this command:

```
sudo apt install dnsmasq hostapd
```

Since the configuration files are not ready yet, turn the new software off as follows:

```
sudo systemctl stop dnsmasq
sudo systemctl stop hostapd
```

### Configuring a static IP

We are configuring a standalone network to act as a server, so the Raspberry Pi needs to have a static IP address assigned to the wireless port. This documentation assumes that we are using the standard 192.168.x.x IP addresses for our wireless network, so we will assign the server the IP address 192.168.4.1. It is also assumed that the wireless device being used is `wlan0`.


To configure the static IP address, edit the dhcpcd configuration file with:

```
sudo nano /etc/dhcpcd.conf
```

Go to the end of the file and edit it so that it looks like the following:

```
interface wlan0
    static ip_address=192.168.4.1/24
    nohook wpa_supplicant
```

Now restart the dhcpcd daemon and set up the new `wlan0` configuration:

```
sudo service dhcpcd restart
```

### Configuring the DHCP server (dnsmasq)

The DHCP service is provided by dnsmasq. By default, the configuration file contains a lot of information that is not needed, and it is easier to start from scratch. Rename this configuration file, and edit a new one:

```
sudo mv /etc/dnsmasq.conf /etc/dnsmasq.conf.orig
sudo nano /etc/dnsmasq.conf
```

Type or copy the following information into the dnsmasq configuration file and save it:

```
interface=wlan0      # Use the require wireless interface - usually wlan0
dhcp-range=192.168.4.2,192.168.4.255,255.255.255.0,15m
address=/#/192.168.4.1 # Redirect all domains (the #) to the address 192.168.4.1 (the server on the (Pi)
```

So for `wlan0`, we are going to provide IP addresses between 192.168.4.2 and 192.168.4.20, with a lease time of 24 hours. If you are providing DHCP services for other network devices (e.g. eth0), you could add more sections with the appropriate interface header, with the range of addresses you intend to provide to that interface.

There are many more options for dnsmasq; see the [dnsmasq documentation](http://www.thekelleys.org.uk/dnsmasq/doc.html) for more details.

Reload `dnsmasq` to use the updated configuration:
```
sudo systemctl reload dnsmasq
```

<a name="hostapd-config"></a>
### Configuring the access point host software (hostapd)

You need to edit the hostapd configuration file, located at /etc/hostapd/hostapd.conf, to add the various parameters for your wireless network. After initial install, this will be a new/empty file.

```
sudo nano /etc/hostapd/hostapd.conf
```

Add the information below to the configuration file. This configuration assumes we are using channel 7, with a network name of NameOfNetwork, and a password AardvarkBadgerHedgehog. Note that the name and password should **not** have quotes around them. The passphrase should be between 8 and 64 characters in length.

To use the 5 GHz band, you can change the operations mode from hw_mode=g to hw_mode=a. Possible values for hw_mode are:
 - a = IEEE 802.11a (5 GHz)
 - b = IEEE 802.11b (2.4 GHz)
 - g = IEEE 802.11g (2.4 GHz)
 - ad = IEEE 802.11ad (60 GHz) (Not available on the Raspberry Pi)

```
interface=wlan0
driver=nl80211
ssid=Pi WiFi
channel=7
```

We now need to tell the system where to find this configuration file.

```
sudo nano /etc/default/hostapd
```

Find the line with #DAEMON_CONF, and replace it with this:

```
DAEMON_CONF="/etc/hostapd/hostapd.conf"
```

### Start it up

Now enable and start `hostapd`:

```
sudo systemctl unmask hostapd
sudo systemctl enable hostapd
sudo systemctl start hostapd
```

Do a quick check of their status to ensure they are active and running:

```
sudo systemctl status hostapd
sudo systemctl status dnsmasq
```

### Add routing and masquerade

Edit /etc/sysctl.conf and uncomment this line:

```
net.ipv4.ip_forward=1
```

Add a masquerade for outbound traffic on eth0:

```
sudo iptables -t nat -A  POSTROUTING -o eth0 -j MASQUERADE
```

(more information about masquerade in iptables https://askubuntu.com/a/466451)

Add redirect for all inbound http web traffic (port 80) to our Node.js server on port 3000.

```
sudo iptables -t nat -I PREROUTING -p tcp --dport 80 -j DNAT --to-destination 192.168.4.1:3000
```

Save the iptables rules.

```
sudo sh -c "iptables-save > /etc/iptables.ipv4.nat"
```

Edit /etc/rc.local 

```
sudo nano /etc/rc.local
```

and add this just above "exit 0" to install the iptables rules on boot.

```
iptables-restore < /etc/iptables.ipv4.nat
```

Reboot and ensure it still functions.

Using a wireless device, search for networks. The network SSID you specified in the hostapd configuration should now be present, and it should be accessible with the specified password.

If SSH is enabled on the Raspberry Pi access point, it should be possible to connect to it from another Linux box (or a system with SSH connectivity present) as follows, assuming the `pi` account is present:

```
ssh pi@192.168.4.1
```

By this point, the Raspberry Pi is acting as an access point, and other devices can associate with it. Associated devices can access the Raspberry Pi access point via its IP address for operations such as `rsync`, `scp`, or `ssh`.

# Configuring the Node.js server to start on boot
Create service that starts the Node.js server automatically whenever the Pi boots up. 

Create a directory for the Node.js files:

```
sudo mkdir -p /Node/PiWiFi
```

Change to the directory:

```
cd Node/PiWiFi
```

Create a package.json file for our Node.js server and accept all the default options:

```
npm init
```

Install express as a dependency:

```
npm install express
```

Create the Node.js file:

```
sudo nano app.js
```

And write / paste the code:

```
const express = require('express');
const app = express();
const port = 3000;

var hostName = 'pi.wifi';

app.use((req, res, next) => {
    if (req.get('host') != hostName) {
        return res.redirect(`http://${hostName}`);
    }
    next();
})

app.get('/', (req, res, next) => {
    res.send('Pi WiFi - Captive Portal');
})

app.listen(port, () => {
    console.log(`Server listening on port ${port}`)
})
```

The key part here is the redirection of any requests that were not from the hostname "pi.wifi".

The redirection command returns a 302 status code. It's this 302 code that triggers the "Sign in to network" functionality, the kind you see when connecting to an open WiFi network at an airport etc.

Create the service file:

```
sudo nano piwifi.service
```

And write / paste the code:

```
[Unit]
Description=Pi WiFi Hotspot Service
After=network.target

[Service]
WorkingDirectory=/home/pi/Node/PiWiFi
ExecStart=/usr/bin/nodejs /home/pi/Node/PiWiFi/app.js
Restart=on-failure
User=root
Environment=PORT=3000

[Install]
WantedBy=multi-user.target
```

Copy the file across:
```
sudo cp piwifi.service /etc/systemd/system
```

and set it up with:

```
sudo systemctl enable piwifi.service
```

It will start automatically on boot, but you can start it now with:

```
sudo systemctl start piwifi
```

When you connect to the WiFi you should be prompted to signin to the network (tested on Windows, Android, and iPhone). Otherwise, any time you try to connect to a http site, you should be redirected to the web server at the domain **pi.wifi**, a domain that doesn't exist on the "real" internet.

If you are redirected to the site successfully, you should see the text "Pi WiFi - Captive Portal".
