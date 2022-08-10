## Filters
- tls.handshake.certificate -> packets containing certificates (useful in WPA enterprise)
- wlan.fc.type_subtype == 0x08 -> beacon frames
- wlan.ssid == "XYZ" -> specify ESSID
- wlan.bssid == 00:01:20:43:21:12 -> filter by BSSID
- wlan.fc.type == X -> X represents frame types: 0 (management), 1 (control), 2 (data), and 3 (extension)
- wlan.fc.subtype == X -> X represents frame subtypes
- wlan.fc.type_subtype in {0x0 0x1 0xb} -> EAPoL frames
- wlan.addr == xx.xx.xx.xx.xx.xx -> search for a  certain client MAC address

More examples: https://www.wifi-professionals.com/2019/03/wireshark-display-filters

### Tshark
```bash
# show packets in a file
sudo tshark -r wpa-eap-tls.pcap 

# show captured packets applying a filter for packets containing certificates exchanged during handshaek
sudo tshark -r wpa-eap-tls.pcap -Y "tls.handshake.certificate" 

# show all data (-x)
sudo tshark -r wpa-eap-tls.pcap -Y "tls.handshake.certificate" -x

# show all fields in capture files (the ones filtered with -Y)
tshark -r b64.pcap -Y "tls.handshake.certificate" -T pdml

# show a specific field (in this case, the certificate)
tshark -r b64.pcap -Y "tls.handshake.certificate" -T fields -e "tls.handshake.certificate" 

# full plaintext dump of packet (the same that you can see on wireshark)
tshark -nr b64.pcap -2 -R "ssl.handshake.certificate" -V

# in JSON format, easier to read:
tshark -nr b64.pcap -2 -R "ssl.handshake.certificate" -T json -V
```



### Tips
- To transfer a capture file you can transfer it via scp, or encode it to base64 ( `` base64 wpa-eap-tls.pcap`` ) , copy the base64 displayed in screen (careful with large files, could result in data loss if the terminal doesn't contain many buffer lines) to a local file and decode it locally ( `` cat b64.txt | base64 -d > b64.pcap`` )

