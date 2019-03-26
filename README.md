<p align="center">
  <img width="600" src="https://raw.githubusercontent.com/android10/RaspberryPi-Wireguard/master/wireguard-logo.png">
</p>

WireGuard is an interesting new VPN protocol that has the potential to bring major change to the VPN industry. 
In comparison to existing VPN protocols, such as OpenVPN and IPSec, WireGuard may offer faster speeds and better reliability with new and improved encryption standards. 
This repository aims to help with the installation of Wireguard, tested on a Raspberry Pi 3 B.


## 0. This is how our solution looks like:

<p align="center">
  <img width="500" src="https://raw.githubusercontent.com/android10/RaspberryPi-Wireguard/master/portable_vpn_raspberry_pi.png">
</p>


## 1. Wireguard installation (Tested on Raspberry Pi 3 B and above)

```console
pi:~ $ sudo apt-get update
pi:~ $ sudo apt-get upgrade 
pi:~ $ sudo apt-get install raspberrypi-kernel-headers
pi:~ $ echo "deb http://deb.debian.org/debian/ unstable main" | sudo tee --append /etc/apt/sources.list.d/unstable.list
pi:~ $ sudo apt-get install dirmngr 
pi:~ $ sudo apt-key adv --keyserver   keyserver.ubuntu.com --recv-keys 8B48AD6246925553 
pi:~ $ printf 'Package: *\nPin: release a=unstable\nPin-Priority: 150\n' | sudo tee --append /etc/apt/preferences.d/limit-unstable
pi:~ $ sudo apt-get update
pi:~ $ sudo apt-get install wireguard 
pi:~ $ sudo reboot
```
**Enable ipv4 forwarding then reboot to make changes active:**

```console
pi:~ $ sudo perl -pi -e 's/#{1,}?net.ipv4.ip_forward ?= ?(0|1)/net.ipv4.ip_forward = 1/g' /etc/sysctl.conf 
pi:~ $ sudo reboot
```

Open ```systctl.conf``` file and make sure ```net.ipv4.ip_forward = 1```:

```console
pi:~ $ sudo nano /etc/sysctl.conf
net.ipv4.ip_forward = 1
```


## 2. Generate private and public keys for server
  
```console
pi:~ $ mkdir wgkeys
pi:~ $ cd wgkeys  
pi:~/wgkeys $ wg genkey > server_private.key  
Warning: writing to world accessible file.
Consider setting the umask to 077 and trying again.
pi:~/wgkeys $ wg pubkey > server_public.key < server_private.key
pi:~/wgkeys $ ls
server_private.key server_public.key
```

With `cat` command we can view the content of the generated file.

```console
pi:~/wgkeys $ cat server_public.key 
Aj2HHAutB2U0O56jJBdkZ/xgb4pnmUPJ0IriuACLLmI=
```


## 3. Generate private and public keys for a client
  
```console
pi:~/wgkeys $ wg genkey > android10_pixel2_private.key
Warning: writing to world accessible file.
Consider setting the umask to 077 and trying again.
pi:~/wgkeys $ wg pubkey > android10_pixel2_public.key < android10_pixel2_private.key
pi:~/wgkeys $ ls
android10_pixel2_private.key android10_pixel2_public.key server_private.key server_public.key
```


## 4. Setup Wireguard interface on server

```console
pi:~ $ sudo vim /etc/wireguard/wg0.conf
```

```console
[Interface]
Address = 10.200.200.1/24
SaveConfig = true
PostUp = iptables -A FORWARD -i %i -j ACCEPT; iptables -A FORWARD -o %i -j ACCEPT; iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
PostDown = iptables -D FORWARD -i %i -j ACCEPT; iptables -D FORWARD -o %i -j ACCEPT; iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE
ListenPort = 51820
PrivateKey = <server_private.key>

[Peer]
#android10-xps
PublicKey = <android10_xps_public.key>
AllowedIPs = 10.200.200.2/32

[Peer]
#android10-pixel2
PublicKey = <android10_pixel2_public.key>
AllowedIPs = 10.200.200.3/32

[Peer]
#android10-gpd
PublicKey = <android10_gpd_public.key>
AllowedIPs = 10.200.200.4/32
```


## 5. Start Wireguard

Start Wireguard with `wg-quick` command.

```console
pi:~/wgkeys $ sudo wg-quick up wg0 
[#] ip link add wg0 type wireguard
[#] wg setconf wg0 /dev/fd/63
[#] ip address add 10.200.200.1/24 dev wg0
[#] ip link set mtu 1420 up dev wg0
[#] iptables -A FORWARD -i wg0 -j ACCEPT; iptables -A FORWARD -o wg0 -j ACCEPT; iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
```

Use `sudo wg` command to check if it is working:

```console
pi:~/wgkeys $ sudo wg 
interface: wg0
  public key: <server_public.key>
  private key: (hidden)
  listening port: 51820

peer: <android10_xps_private.key>
  allowed ips: 10.200.200.2/32

peer: <android10_pixel2_private.key>
  allowed ips: 10.200.200.3/32

peer: <android10_gpd_private.key>
  allowed ips: 10.200.200.4/32
```

**You can launch automatically at startup:**

```console
pi:~/wgkeys $ sudo systemctl enable wg-quick@wg0  
Created symlink /etc/systemd/system/multi-user.target.wants/wg-quick@wg0.service → /lib/systemd/system/wg-quick@.service.
```


## 6. Setup clients

You will need to install wireguard on clients as well. 
**IMPORTANT:** Wireguard does not have separate apps for server and client, just differences in the configuration file. 

**Installing Wireguard Client Tools.**

 - Arch Linux -> `sudo pacman -S wireguard-tools linux-headers`.
 - On Debian based distros -> `sudo apt-get install wireguard`.
 - Other platforms -> [Wireguard Website](https://www.wireguard.com/install/). 
 - Android -> [Google Play](https://play.google.com/store/apps/details?id=com.wireguard.android).
 - iOS -> [App Store](https://itunes.apple.com/us/app/wireguard/id1441195209?ls=1&mt=8).

**My Android Pixel 2 client EXAMPLE.**

After intalling the Android Client from the link above, here is the Example configuration we should use (same applies for other clients you want to setup up):

```wireguard-android10-pixel2.conf```

```console
[Interface]
Address = 10.200.200.3/24
PrivateKey = <android10_pixel2_private.key>
DNS = 8.8.8.8

[Peer]
PublicKey = <server_public.key>
AllowedIPs = 0.0.0.0/0, ::/0
Endpoint = my.ddns.address.com:51820
```

**IMPORTANT:** Using the catch-all **AllowedIPs = 0.0.0.0/0, ::/0** will forward all **IPv4 (0.0.0.0/0)** and **IPv6 (::/0)** traffic over the **VPN**.

**Screenshots for the Android Application**

<p align="center">
  <img width="500" src="https://raw.githubusercontent.com/android10/RaspberryPi-Wireguard/master/android_setup01.png">
</p>

<p align="center">
  <img width="500" src="https://raw.githubusercontent.com/android10/RaspberryPi-Wireguard/master/android_setup02.png">
</p>

**Arch Linux:**
- Configure Wireguard -> [Wireguard Arch Linux Wiki](https://wiki.archlinux.org/index.php/WireGuard). 

**Other clients:**
- Configure Wireguard -> [External Website](https://www.stavros.io/posts/how-to-configure-wireguard/). 


## Port Forwarding 

You need to forward one port in your router:
  - **Type:** UDP. 
  - **Port:** 51820.


## Useful Commands:

```console
# See WireGuard current state:
pi:~ $ sudo wg 

# Bring up the interface:
pi:~ $ sudo wg-quick down wg0

# Close the interface:
pi:~ $ sudo wg-quick down wg0

# Configure the wg0 interface:
pi:~ $ sudo vim /etc/wireguard/wg0.conf

# Check current service status:
pi:~ $ systemctl status wg-quick@wg0.service
```


## Resources:

### WireGuard website:
 - https://www.wireguard.com
 - https://www.wireguard.com/install/
 - https://www.wireguard.com/talks/eindhoven2018-slides.pdf

### Arch Linux wiki:
 - https://wiki.archlinux.org/index.php/WireGuard
 - https://wiki.archlinux.org/index.php/WireGuard#Server_2



## Credits
 - https://github.com/adrianmihalko/raspberrypiwireguard
 - https://emanuelduss.ch/2018/09/wireguard-vpn-road-warrior-setup/
 - https://www.ckn.io/blog/2017/12/28/wireguard-vpn-portable-raspberry-pi-setup/

