## Connecting to an Access Point

### Change between monitor and manage mode

#### Monitor mode
##### airmon-ng
``airmon-ng start wlan0``

##### Manually
``` bash
# Use the following command to set interface in monitor mode.
iw dev <interface> set monitor none

# If this gives you device busy error, then do the following:
ifconfig <interface> down
iw dev <interface> set monitor none
ifconfig <interface> up
```

#### Managed mode
<font color=red>Needed for connecting to networks!!!</font>
##### airmon-ng
``sudo airmon-ng stop wlan0mon``

##### Manually
```bash
ifconfig mon0 down
ifconfig mon0 mode managed
ifconfig mon0 up
```

We can also disconnect and reconnect the adapter. With ``iwconfig`` we can see the mode of the interface.

To connect to a network we need to reestart NetworkManager, if we killed it previously with ``airmon-ng check kill``
``sudo service NetworkManager start``

If the network uses mac filtering we cannot connect. It can be blacklist or whitelist. If it's blacklist we can use any non blacklisted MAC. If it's whitelisted we need to use the MAC of a connected client.
A symptom of MAC filtering is that the network is OPEN or we have a password and still can't connect

<font color=red>sometimes changed macs don't stay when trying to connect to the network </font>



### wpa_supplicant -> Client to connect to wifi networks

<font color=red>IMPORTANT, indent what's insde braces or it will fail (parsing error). If tabs fail, add a couple of whitespaces instead</font>

<font color=red>IMPORTANT, if wpa_supplicant is connected via an interface to a network it cannot connect to another, search ps aux for wpa_supplicant processes and kill them before connecting to another network or with a different configuration</font>

several interfaces of wpa_supplicant can be run in parallel for different interfaces with different configurations

- scan_ssid -> send probe requests
#### Config file for open network
```
network={
  ssid="<ESSID>"
  scan_ssid=1
}
```

alternative:
```
network={
  ssid="<ESSID>"
  scan_ssid=1
  mode=0
  auth_alg=OPEN
  key_mgmt=NONE
}
```
#### Config file for WEP network
<font color=red>If we have a hex key, dont use quotation marks " and don't use : to separate bytes (the next two examples are equivalent, one with ASCII key and the other with hex key)</font>
```
network={
  ssid="<ESSID>"
  key_mgmt=NONE
  wep_key0="34567"
  wep_tx_keyidx=0
}
```

```
network={
  ssid="<ESSID>"
  key_mgmt=NONE
  wep_key0=0304050607
  wep_tx_keyidx=0
}
```

alternative
```
network={
  ssid="<ESSID>"
  scan_ssid=1
  mode=0
  auth_alg=OPEN
  key_mgmt=NONE
  wep_key0=0304050607
}
```
#### Config file for WPA-PSK network
Valid for WPA-PSK and WPA2-PSK
```
network={
  ssid="<ESSID>"
  scan_ssid=1
  psk="<passphrase>"
  key_mgmt=WPA-PSK
}
```

alternative
```
network={
  ssid="<ESSID>"
  mode=0
  scan_ssid=1
  auth_alg=OPEN
  key_mgmt=WPA-PSK
  proto=WPA
  pairwise=TKIP
  group=TKIP
  psk="<passphrase>"
  
}
```
#### Specific config file for WPA2-PSK
but WPA2-PSK (only) can be specified like this also:

wpa_supplicant will automatically choose between TKIP and CCMP based on availability, but it is possible to force one or the other by adding _pairwise=CCMP_ or _pairwise=TKIP_ to the configuration if necessary.
```
network={
  ssid="<ESSID>"
  key_mgmt=WPA_PSK
  psk="<passphrase>"
  proto=RSN
  pairwise=CCMP
  group=CCMP
}

# less specific, can work better
network={
  ssid="<ESSID>"
  key_mgmt=WPA_PSK
  psk="<passphrase>"
  proto=RSN
}

# or maybe this is necessary, due to retrocompatibility with old devices
network={
  ssid="<ESSID>"
  key_mgmt=WPA_PSK
  psk="<passphrase>"
  proto=WPA
  pairwise=CCMP
  group=CCMP
}
```
- RSN -> Robust Secure Network (this sets pairwise and group to CCMP, although it can be specified explicitely so that we are not downgraded in any case). Maybe specifying  pairwise and/or group fails, don't specify them first

alternative:
```
network={
  ssid="<ESSID>"
  scan_ssid=1
  mode=0
  auth_alg=OPEN
  key_mgmt=WPA_PSK
  psk="<passphrase>"
  proto=RSN
  pairwise=CCMP
  group=CCMP
}
```
  
#### WPA-Enterprise
##### PEAP-MSCHAPv2 authentication
```
network={
  ssid="<ESSID>"
  scan_ssid=1
  key_mgmt=WPA-EAP
  eap=PEAP
  identity="bob"
  password="hello"
  phase1="peaplabel=0"
  phase2="auth=MSCHAPV2"
}
```

##### PEAP-GTC WPA Supplicant Configuration
```
  network={
  ssid="<ESSID>"
  scan_ssid=1
  key_mgmt=WPA-EAP
  eap=PEAP
  identity="bob"
  password="hello"
  phase1="peaplabel=0"
  phase2="auth=GTC"
}
```


##### TTLS-PAP WPA Supplicant Configuration
```
network={
  ssid="<ESSID>"
  scan_ssid=1
  key_mgmt=WPA-EAP
  eap=TTLS
  identity="bob"
  anonymous_identity="anon"
  password="hello"
  phase2="auth=PAP"
}
```

##### TTLS-CHAP WPA Supplicant Configuration
```
network={
  ssid="<ESSID>"
  scan_ssid=1
  key_mgmt=WPA-EAP
  eap=TTLS
  identity="bob"
  anonymous_identity="anon"
  password="hello"
  phase2="auth=CHAP"
}
```

##### TTLS-MSCHAPv2 WPA Supplicant Configuration
```
network={
  ssid="<ESSID>"
  scan_ssid=1
  key_mgmt=WPA-EAP
  eap=TTLS
  identity="bob"
  anonymous_identity="anon"
  password="hello"
  phase2="auth=MSCHAPV2"
}
```


Tool to generate configuration files: ``wpa_passphrase``. Mandatory parameter: ESSID. Optional parameter: passphrase

#### Connect to a network with wpa_supplicant and config file
- -i -> interface used to connect
- -c -> config file
- -B -> run wpa_supplicant in the background
```bash
sudo wpa_supplicant -i wlan0 -c wifi-client.conf

# sometimes the driver that wpa_supplican uses is specified (different from the driver used for the wifi interface)
sudo wpa_supplicant -Dnl80211 -i wlan0 -c wifi-client.conf

# request an ip by dhcp, once we are connected to an AP
dhclient -v wlan0
```

### Manual connection
```
sudo /sbin/ifconfig wlan0 up
sudo /sbin/iwlist wlan0 scan
sudo /sbin/iwconfig wlan0 essid "NetworkName"
sudo /sbin/iwconfig wlan0 key network_key
sudo /sbin/iwconfig wlan0 enc on
```

To get an IP after connecting to the AP:  `dhclient -v wlan0`

alternative method:

```
sudo iwconfig wlan0 essid <SSID> key s:<KEY>
sudo dhclient -v wlan0
```


### Change MAC address
#### Manually:
```bash
ifconfig wlan0 down
ifconfig wlan0 hw ether <new MAC>
ifconfig wlan0 up
```

#### Mac-changer:
```bash
# for specified mac
sudo macchanger -m <valid MAC> wlan0

# for random mac
sudo macchanger -r wlan0
```



### Change Wifi band
for 5 GHz
``airodump-ng --band a wlan0mon``
For both 5 and 2.4 GHz:
``airodump-ng --band abg wlan0mon``


#### Wifi bands
- Decide which ranges of freqs can be used
- Determine the channels that can be used
- Clients must support the band used by the AP to connect to it or sniff traffic
Most common bands:
- a, only 5 GHz -> seems like scanning with airodump on band a can pick up 2.4 GHz APs too
- b, g, only 2.4 GHz
- n, both 5 and 2.4 GHz
- ac, freqs lower than 6 GHz


Channel bonding: sometimes several channels are combined into one, used to avoid interferences between channels. 802.11 n - compatible networks means that they support channel bonding.

In 5 GHz there is no overlapping in frequency between adjacent channels, that increases throughput. In 2.4 GHz there is.

When a client sends the others cannot. For that is good to have low power APs, to avoid many clients connecting to the same AP and one of them takes over.


### Other commands
```bash
# devices connected by usb
sudo lsusb -vv

# physical properties of wifi interfaces (support of a card for monitor mode can be found)
iw dev
iw phy
iw list

# view regional settings. If some channel says PASSIVE-SCAN, it is listening but not sending packets
iw reg get

# change regulatory domain settings
iw reg set <country code>

# if iw phy says no IR (IR=initial radiation) that channel is not used in the configured country. We can also check if a channel can be used by checking if packets arrive when we do:
iw dev wlan0 set channel 13
aireplay-ng --test wlan0

# change channel width (for channel bonding, although management frames have always a standard width of 20 MHz)
iw dev wlan0 set channel 6 HT40+
iw dev wlan0 set channel 36 80MHz

# Scan networks without airodump
iw dev wlan0 scan
iw dev wlan0 scan |grep "SSID:"

# Connect to an open SSID
iwconfig wlan0 essid <essid>


```

### Common mistakes:
- interface
	- make sure it is up (ifconfig wlan0 up)
	- make sure it is the correct mode (iw dev)
- sniffing 
	- sniff in all frequencies: (a, b/g)
	- use proper channel width, if there is channel bonding


#### Hidden networks
Hidden networks don't advertise their name (ESSID) but they advertise their presence (BSSID). This is enough for us to not be able to connect or try to crack their pass, or try to launch attacks against it.
In linksys routers the ESSID can be disabled like this:

![[Pasted image 20220701204113.png]]
Airodump only shows the name length, but not its value
![[Pasted image 20220701204047.png]]

In windows we see "hidden network" and it asks for the name when we try to connect
![[Pasted image 20220701204326.png]]

If a network has hidden ESSID the first step is to ALWAYS try to find it. If there are clients connected we can deauth one of them and we will capture the name when he reconnects.
If there are no connected clients try a dictionary attack, trying to connect to a network using different names from a dictionary

Non probing clients are clients which are there, but don't probe networks, therefore we cannot detect their presence. We can create fake APs to see if he connects to any of them.

### others
Several APs with different BSSID can share the same ESSID (this is, a single network with several access points)

The radiotap header that wireshark shows is added by wireshark, that information is not in packets sent through the air

Types of frames (for attacks, data frames are the important ones)
- Management (advertisement, discovery, connection/disconnection)
- Control (to facilitate delivery of Management and Data frames)
- Data

WDS -> wireless distribution system: provide internet access from one wifi router to another via wifi (not by cable), so that the second one can cover a zone where the signal of the first doesn't reach

Beacon flood attack: fill the air with fake beacons so that clients see a lot of APs and the ones they want to see may fall out of the list ``mdk4 wlan0 b``


-   Some wireless drivers ignore directed deauthentication and only respond to broadcast deauthentication. We can run the same aireplay-ng deauthentication command without the -c parameter.

-   If 802.11w is in use, unencrypted deauthentication frames are ignored. The only course of action is to wait for a client to connect.
-   The device simply didn't reconnect or was already out of range of the AP.

wigle.net -> geographical location of BSSIDs

Some phones randomize their MAC until the moment they connect to a network, when they switch to the good one. If we setup a honeypot we can get their real MAC if they connect to us
