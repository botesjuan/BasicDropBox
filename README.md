# Basic Drop-box Assessment Tool

>Objective of this project is to create a simple device that is connect to internal network of scope to be assessed, and have a Wireless access point established to bridge from outside to the inside network.
>The Raspberry PI ethernet network is physically connected to the scope internal network, and the wireless adapter is setup as WiFi access point.

----  

# Kali Linux WiFi Access Point Setup  

>To set up your Raspberry Pi 4 (running Kali Linux) as a wireless access point and use SSH (via PuTTY) to connect from your laptop, follow these steps:

>Running Kali Linux on Raspberry Pi 4, the system does not use `dhcpcd.conf` and you've successfully configured the `eth0` interface using the `/etc/network/interfaces` file, I'll update the wireless access point (AP) setup to align with this configuration.  

### Updated Steps for Setting up a Wireless Access Point (AP) on Kali Linux

#### Step 1: Install Required Packages
First, install the necessary packages:

```bash
sudo apt update
sudo apt install hostapd dnsmasq
```

Enable the `hostapd` service:

```bash
sudo systemctl unmask hostapd
sudo systemctl enable hostapd
```

#### Step 2: Configure Static IP for WLAN0 in `/etc/network/interfaces`

Since you're using `/etc/network/interfaces`, you'll configure the static IP for the wireless interface `wlan0` directly in that file.

1. Open the `/etc/network/interfaces` file:

   ```bash
   sudo nano /etc/network/interfaces
   ```

2. Add the following configuration for `wlan0` to assign a static IP:

   ```bash
   auto wlan0
   iface wlan0 inet static
       address 192.168.4.1
       netmask 255.255.255.0
   ```

   This ensures `wlan0` uses a static IP (192.168.4.1) for your wireless access point.

3. Save and close the file.

#### Step 3: Configure `dnsmasq` for DHCP

1. Backup the default configuration file:

   ```bash
   sudo mv /etc/dnsmasq.conf /etc/dnsmasq.conf.orig
   ```

2. Create a new configuration file:

   ```bash
   sudo nano /etc/dnsmasq.conf
   ```

3. Add the following lines to serve IP addresses for devices connecting to the access point:

   ```bash
   interface=wlan0
   dhcp-range=192.168.4.2,192.168.4.20,255.255.255.0,24h
   ```

   This configures `dnsmasq` to serve DHCP addresses from 192.168.4.2 to 192.168.4.20 for clients connecting to `wlan0`.

4. Save and close the file.

#### Step 4: Configure `hostapd`

1. Create a `hostapd` configuration file:

   ```bash
   sudo nano /etc/hostapd/hostapd.conf
   ```

2. Add the following content:

   ```bash
   interface=wlan0
   driver=nl80211
   ssid=MyAccessPoint
   hw_mode=g
   channel=7
   wmm_enabled=0
   macaddr_acl=0
   auth_algs=1
   ignore_broadcast_ssid=0
   wpa=2
   wpa_passphrase=YourStrongPassword
   wpa_key_mgmt=WPA-PSK
   wpa_pairwise=TKIP
   rsn_pairwise=CCMP
   ```

   Replace `MyAccessPoint` with the SSID of your choice, and `YourStrongPassword` with a strong password for the AP.

3. Now tell the system to use this configuration by editing the `hostapd` default configuration:

   ```bash
   sudo nano /etc/default/hostapd
   ```

4. Modify the line `#DAEMON_CONF=""` to:

   ```bash
   DAEMON_CONF="/etc/hostapd/hostapd.conf"
   ```

5. Save and close the file.

#### Step 5: Enable IP Forwarding and Configure NAT

To allow devices connecting to your access point to access the internet, you'll need to enable IP forwarding and set up NAT.

1. Enable IP forwarding by editing `/etc/sysctl.conf`:

   ```bash
   sudo nano /etc/sysctl.conf
   ```

2. Uncomment the following line:

   ```bash
   net.ipv4.ip_forward=1
   ```

3. Apply the changes:

   ```bash
   sudo sysctl -p
   ```

4. Configure `iptables` for NAT:

   ```bash
   sudo iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
   sudo iptables -A FORWARD -i eth0 -o wlan0 -m state --state RELATED,ESTABLISHED -j ACCEPT
   sudo iptables -A FORWARD -i wlan0 -o eth0 -j ACCEPT
   ```

5. Save the `iptables` rules so they persist after reboot:

   ```bash
   sudo sh -c "iptables-save > /etc/iptables.ipv4.nat"
   ```

6. Ensure the rules are applied on boot by editing `/etc/rc.local`:

   ```bash
   sudo nano /etc/rc.local
   ```

   Add the following line before `exit 0`:

   ```bash
   iptables-restore < /etc/iptables.ipv4.nat
   ```

7. Save and close the file.

#### Step 6: Start Services

1. Start `hostapd` and `dnsmasq` services:

   ```bash
   sudo systemctl start hostapd
   sudo systemctl start dnsmasq
   ```

2. Ensure they start on boot:

   ```bash
   sudo systemctl enable hostapd
   sudo systemctl enable dnsmasq
   ```

#### Step 7: Connect and SSH into the Raspberry Pi

1. From your laptop, connect to the wireless access point you created (`MyAccessPoint`) using the password you set.

2. Once connected, use PuTTY (or any other SSH client) to connect to the Raspberry Pi's static IP address (`192.168.4.1`):

   - **Host Name (or IP address):** `192.168.4.1`
   - **Port:** `22`
   - **Connection Type:** SSH

This should establish a wireless SSH connection to your Raspberry Pi.

Let me know if you need further assistance!


sudo apt update
sudo apt upgrade -y
sudo apt install ffmpeg v4l-utils python3-pip python3-dev libssl-dev libcurl4-openssl-dev libjpeg-dev zlib1g-dev libffi-dev


