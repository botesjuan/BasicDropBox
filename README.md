# Basic Drop-box Assessment Tool

>Objective of this project is to create a simple device that is connect to internal network to be assessed, and have a Wireless access point established to bridge from outside to the inside network.
>This device provide a means to overcome network segmentation and spoof a valid internal MAC address in attempt to by network access control (NAC).  
>The Raspberry PI ethernet network is physically connected to the target internal network, and the wireless adapter is setup as WiFi access point.

[My other physical penetration tools, including another more expensive drop-box](https://github.com/botesjuan/Pentester-Toolbox/blob/main/README.md)  

----  

>High level setup steps:

1. Install Kali Linux on Raspberry Pi4
2. Preparing networking, NTP timezone, & changing default password of kali user.
3. Install extra Penetration Testing Tools, Services, GPIO Libraries, daemons, etc.
4. Setup Wireless access point on Kali OS.  
5. Change Ethernet MAC address to bypass Network Access Control (NAC).  
6. Wiring power direct to GPIO connectors, and wire the seven segment LED display to GPIO pin outs.
7. Run a python script at startup as service (Create authentic look of physical device disguise with 7 segement LED display)
8. SSH Connect via WiFi IP.  

----  

# Install Kali on Raspberry Pi 4  

>Download the 7z from Offsec Kali Linux for Raspberry Pi.
>Use `balenaEtcher` as Local administrator on Windows 10 workstation to burn image to 32GB SD memory card.  

----  

# Prepare Kali on Raspberry Pi  

>Change the ethernet `eth0` to automatically get DHCP IP address.
>Edit `/etc/network/interfaces`  

```
auto eth0
  iface eth0 inet dhcp

auto wlan0
  iface wlan0 inet static
  address 192.168.4.1
  netmask 255.255.255.0

auto lo
  iface lo inet loopback

```

>Change Kali user password `sudo passwd kali`

>Set the timezone where the pentest drop-box is GEO located: `sudo timedatectl set-timezone Africa/Johannesburg`  

----  

# Extra Tools & Services Install  

>Install extra Penetration Testing Tools, Services, wordlist, daemons:

```
sudo apt update && sudo apt upgrade -y
sudo apt install hostapd dnsmasq
gobuster
feroxbuster
bloodhound
rlwrap
searchsploit --update
xsltproc
jq
sudo apt -y install seclists
```

>Tunnel pivot tool [CHISEL](https://github.com/jpillora/chisel?tab=readme-ov-file#install)  

>Install raspberry pi GPIO libraries on Kali Linux:  

```
sudo apt install python3-rpi.gpio
```  

----  

# Kali Linux WiFi Access Point Setup  

>set up Raspberry Pi 4 (running Kali Linux) as a wireless access point and use SSH (via PuTTY) to connect from laptop, follow these steps:

>Running Kali Linux on Raspberry Pi 4, the system does not use `dhcpcd.conf`    
>Configured the `eth0` interface using the `/etc/network/interfaces` file  
>Wireless access point (AP) configuration:  

## Setting up a Wireless Access Point (AP) on Kali Linux  

### Step 1: Install Required Packages  

>Install the necessary packages:  

```bash
sudo apt update
sudo apt install hostapd dnsmasq
```

>Enable the `hostapd` service:  

```bash
sudo systemctl unmask hostapd
sudo systemctl enable hostapd
```

### Step 2: Configure Static IP for WLAN0  

>Use `/etc/network/interfaces`, to configure the static IP for the wireless interface `wlan0` directly in that file.  

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

>This ensures `wlan0` uses a static IP (192.168.4.1) for wireless access point.  

3. Save and close the file.  

### Step 3: Configure `dnsmasq` for DHCP  

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

>This configures `dnsmasq` to serve DHCP addresses from 192.168.4.2 to 192.168.4.20 for clients connecting to `wlan0`.

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

   Replace `MyAccessPoint` with the SSID name, and `YourStrongPassword` with a strong password for the AP.

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

To allow devices connecting to access point to access the internet, need to enable IP forwarding and set up NAT.

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

1. From laptop, connect to the wireless access point created (`MyAccessPoint`) using the password.  

2. Once connected, use PuTTY (or any other SSH client) to connect to the Raspberry Pi's static IP address (`192.168.4.1`):

   - **Host Name (or IP address):** `192.168.4.1`
   - **Port:** `22`
   - **Connection Type:** SSH

----  

# Change Hostname & MAC Address Permanent  

## Change Hostname  

>To change the hostname on Kali Linux: Edit the `/etc/hostname` file:

>Replace the current hostname with desired new hostname.

>Edit the `/etc/hosts` file:

>Find the line that says `127.0.1.1 old-hostname` (where old-hostname is current hostname).
>Change old-hostname to new hostname (e.g., new-hostname):

## Make MAC Address Persistent  

>To ensure the MAC address remains after reboot, create file: `sudo mousepad /etc/systemd/system/macspoof@.service` adding below line:
>Insert Content below in new file:

```
[Unit]
Description=Spoof MAC address for %i
Wants=network-pre.target
Before=network-pre.target
BindsTo=sys-subsystem-net-devices-%i.device
After=sys-subsystem-net-devices-%i.device

[Service]
Type=oneshot
ExecStart=/usr/bin/macchanger -m XX:XX:XX:XX:XX:XX %i
ExecStartPost=/usr/sbin/ip link set dev %i up

[Install]
WantedBy=multi-user.target
```  

>Enable the macspoof@.service  

```
sudo systemctl enable macspoof@eth0.service
```

>Start the Service  

```
sudo systemctl start macspoof@eth0.service
sudo reboot
```

>Verify the MAC Address  

```
ip addr show eth0
```

----  

# Install Python Script as Service  

>Automatically run Python program at boot on Raspberry Pi running Kali Linux, setting it up using systemd.
>Confirm prerequisite of `python3-rpi.gpio` is installed before starting python service.  

## Step 1: Create a Service File for systemd  

>Create a new service file for Python program:

```bash
sudo nano /etc/systemd/system/seven_segment.service
```

>Add the following content to the service file, modifying the path to Python script if necessary:

```ini
[Unit]
Description=Seven Segment Display Script
After=multi-user.target

[Service]
Type=idle
ExecStart=/usr/bin/python3 /path/to/x/seven_segment.py
Restart=on-failure
User=pi  # Replace 'pi' with  Raspberry Pi username if different

[Install]
WantedBy=multi-user.target
```

>ExecStart: Replace `/path/to/x/seven_segment.py` with the full path to Python script.
>User: Comment out username of kali for pi to run as root.  

## Step 2: Reload systemd and Enable the Service  

>Reload systemd to recognize the new service:  

```
sudo systemctl daemon-reload
```  
>Enable the service so it runs at boot:

```
sudo systemctl enable seven_segment.service
```  
>Start the service immediately to test if it works:

```
sudo systemctl start seven_segment.service
```
>Check the status of the service to ensure it's running without errors:

```
sudo systemctl status seven_segment.service
```
:See that the service is running. If there are any errors, check the logs for more details.

## Step 3: Reboot to Test  

>Reboot Raspberry Pi to confirm the script runs automatically at startup:  

```
sudo reboot
```

## Step 4: Debugging the Service  

>If the service doesn't start correctly, check the logs with:

```
journalctl -u seven_segment.service
```
>This will help identify any errors or issues that might be occurring.

----  

![basic-drop-box-spray.jpg](/basic-drop-box-spray.jpg)  

>The inside of the air freshner spray drop box containing raspberry pi 4 with USB power bank (battery for 5V)

![basic-drop-box-spray-inside.jpg](/basic-drop-box-spray-inside.jpg)  

----  

# Raspberry Pi Maintenance  

>After several months using a 32 GB size SDcard memory is out of disk space.  
>I ran out of space but did not want too loose all my customization and hosting data content.  

## Replacing 32gb with 128gb  

✅ Full migration process **replacing a full 32 GB Kali SD card** with a **128 GB bootable upgrade**, while keeping all custom configs and installed Kali Linux pentest tools.  

----  

## Replace Full 32GB SDCard with 128GB Bootable Kali Linux (Preserving Setup)  

* Working Raspberry Pi 4
* Full 32GB Kali Linux SD card (source)
* 128GB A1 Class 10 SD card (target)
* Ubuntu host machine
* Tools: `rsync`, `dd`, `losetup`, `blkid`, `gparted` or `fsck`  

----  

## 🔧 Step-by-Step  

1. **Remove the full 32GB SD card** from Raspberry Pi 4, insert into Ubuntu host.
2. **Unmount its partitions** using `umount` or `lsblk`.
3. **Create a full image backup** using `dd`:

   ```bash
   sudo dd if=/dev/sdX of=~/kali32.img bs=4M status=progress conv=fsync
   ```
4. **Prepare a 128GB A1 SD card** (Class 10, non-A2 for Pi compatibility).
5. **Flash official Kali Linux ARM64 image** to 128GB card using Raspberry Pi Imager.
6. **Boot test** the 128GB card in Raspberry Pi 4 to confirm it works.
7. **Insert 128GB SD card into Ubuntu**, identify and mount the `ROOTFS` partition.
8. **Mount the 32GB `.img` file's root partition** using `losetup` and offset.
9. **Use `rsync` to copy the rootfs** from image to mounted 128GB root partition:

   ```bash
   sudo rsync -aAXv ~/mnt/kali32-rootfs/ /mnt/kali128-rootfs/
   ```
10. **Edit `/mnt/cmdline.txt`** on the 128GB boot partition:

    * Append to the end of the single line:

      ```
      init=/bin/bash
      ```
11. **Run `sync` and unmount all partitions** cleanly.
12. **Eject and insert the 128GB SD into the Raspberry Pi 4.**
13. Pi **boots into root shell** (due to `init=/bin/bash`).
14. Run:

    ```bash
    blkid
    ```

    * Note the **correct UUID** of `/dev/mmcblk0p2`
15. Remount root as writable:

    ```bash
    mount -o remount,rw /
    ```
16. **Edit `/etc/fstab`** to replace old UUID with the correct one.
17. **Remove `init=/bin/bash`** from `/boot/cmdline.txt`
18. Reboot:

    ```bash
    sudo reboot
    ```

>**Original Kali build** running on a **fully bootable 128GB SD card**, with full disk space and all data/configs preserved 🎯.  

----  
