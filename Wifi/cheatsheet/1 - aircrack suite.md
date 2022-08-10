#### airmon-ng
```bash
# List interfaces
sudo airmon-ng  

# List programs that can interfere with aircrack-ng suite
sudo airmon-ng check 

# Kill processes that can interfere with aircrack-ng suite
sudo airmon-ng check kill

# Create an interface (wlan0mon) in monitor mode from an existing one (wlan0)
sudo airmon-ng start wlan0

# Stop monitor mode
sudo airmon-ng stop wlan0mon

# Start monitor mode only on channel 2 (only do this when the tool that will be used next doesn't change channels itself)
sudo airmon-ng start wlan0 2

# manually set channel
iw dev wlan0 set channel 13

# Check that we changed the channel correctly
sudo iw dev wlan0mon info

# verbose and debug mode
sudo airmon-ng --verbose
sudo airmon-ng --debug

```


Even if we don't see connected clients, send deauth packets against the different networks, with airodump listening in the specific channel (so that we don't miss reauth attempts if we are scanning other channels at that moment)


#### airodump-ng
```bash
# Specify the channel where airodump listens
airodump-ng --channel 11 --bssid <bssid>

# listen to a single bssid and write output to a file (it creates several files with different formats)
airodump-ng --channel 11 --bssid <bssid> --write <file name>

# scan both 2.4 and 5 GHz simultaneously
airodump-ng wlan0 --band abg

# load capture file in airodump
airodump-ng -r <file.cap>

# show WPS status for WPA networks
airodump-ng wlan0 --wps
```


#### aireplay-ng
```bash
# deauth a client (1000000 is a large number of packets, to keep the deauth attack working for a while):
sudo aireplay-ng --deauth 4 -a <bssid> -c <client_MAC> wlan0mon

# To background the command and don't see output
sudo aireplay-ng --deauth 4 -a <bssid> -c <client_MAC> wlan0mon &> /dev/null &

# with "jobs" we can see the jobs backgrounded with &. each has an ID
jobs

# kill all backgrounded aireplay processes.
killall aireplay-ng 

# kill only the first process in the "jobs" list:
kill %1

# To deauth every client connected to a BSSID don't specify a client <MAC>
aireplay-ng --deauth 4 -a <bssid> wlan0mon &> /dev/null &

# check if we can inject in visible APs
sudo aireplay-ng -9 wlan0mon 

# check if we can inject in a specific AP
sudo aireplay-ng -e <ap_name> -a <MAC> wlan0mon

# Same as above, but without expecting to receive probes
sudo aireplay-ng -e <ap_name> -a <MAC> -D wlan0mon

# if we have two wifi cards, wlan0mon and wlan1mon, card-to-card test, to make sure they can inject. if it says (5/7 error, still can be used to attack an AP)
sudo aireplay-ng -9 -i wlan1mon wlan0mon

```

<font color="red">Sometimes aireplay-ng says that he can't find the BSSID, that's because it's not using the appropriate channel. For that, run airodump-ng in the appropriate channel before aireplay-ng, or set the channel with 
"iw dev wlan0 set channel 13"</font>


#### airdecap-ng
```bash
# Keep the packets targeted to a specific <BSSID> and remove the rest from a cap file (creates a new file)
airdecap-ng -b <MAC> file.pcap

# decrypt saved traffic with a passphrase (check that the passphrase works, we may capture failed logins)
airdecap-ng -b <bssid> -e <essid> -p <pass> file.pcap
```


#### aircrack-ng
```bash
#benchmark (dice k/s, que es el numero de palabras por segundo que puede crackear)
aircrack-ng -S  

# DON'T use a dictionary for WEP files!!!!
aircrack-ng wep.cap

# crack a handshake saved in a cap file:
aircrack-ng -w <path to wordlist> -e <ESSID> -b <ap bssid> file.pcap
aircrack-ng -w /usr/share/john/password.lst -e <ESSID> -b <ap bssid> file.cap

#crack using a db created with airolib (precomputed PMKs)
aircrack-ng -r wifu.sqlite wpa1-01.cap
```
<font color=red>If in a capture file an AP has hidden name but we find it in another way, we need to pass both arguments to aircrack-ng, -b and -e, so that it can match a BSSID to an ESSID</font>


#### airgraph-ng
Creates graphs of APs and stations. Colors:
- green -> WPA
- yellow -> WEP
- red -> open
- black -> desconocido el cifrado

```bash
#CAPR: client to access point relationship. Provide a csv captured by airodump
airgraph-ng -i dump.csv -o file.png -g CAPR

# CPG: client probe graph -> shows relations (connections) from clients to APs
airgraph-ng -i dump.csv -o file.png -g CPG
```


#### airolib-ng
manages password lists in SQLite (calculating pairwise master key (PMK) is slow, but it is constant for an AP. precomputing it saves time later).

```bash
# create a text file containing the ESSID of the target AP
echo wifu > essid.txt

# import the text file into an airolib-ng database
airolib-ng wifu.sqlite --import essid essid.txt

# info about database (ESSIDs and stored passwords)
airolib-ng wifu.sqlite --stats

# import a dictionary of passwords (ignores those shorter than 8 chars and larger than 63 chars, since they are not valid WPA passphrases)
airolib-ng wifu.sqlite --import passwd /usr/share/john/password.lst

# calculate the PMK corresponding to each inported password
airolib-ng wifu.sqlite --batch

#crack using a db
aircrack-ng -r wifu.sqlite wpa1-01.cap
```
