## Default credentials
First of all, if we know the AP model, check for default credentials. The first half of the BSSID can help:
https://www.wireshark.org/tools/oui-lookup.html

## Online cracking
```bash
# hydra ssh
hydra -t 4 -l root -P /root/wordlists/100-common-passwords.txt ssh://192.105.16.4

# hydra ftp
hydra -t 3 -l root -P /root/wordlists/100-common-passwords.txt ftp://192.105.16.4

# POP3
hydra -L login.txt -P passwords.txt -V 192.168.50.128 -s 110 -t 2 pop3
```

## Offline Cracking
### Hashcat
- <font color="red">The hashcat module to crack WPA/WPA2 is 2500</font>
- <font color="red">We can pause (``p``) and resume (``r``) the hashcat execution</font>
```bash
# info about cracking hardware
hashcat -I 

# benchmark of all hash types (very slow)
hashcat -b

# benchmark a single hash type
hashcat -b -m 2500

# extract hashes from a cap file
cap2hccapx.bin file.cap output.hccapx

# crack
hashcat -m 2500 out.hccapx >wordlist>

# with -d we can choose the cracking device of the listed ones
hashcat64 -m 2500 -d 1 <pcap file> <wordlist>

# with --pot-file we can indicate another path to save the pot file

# install hashcat utilities (found in /usr/lib/hashcat-utils)
sudo apt install hashcat-utils

# convert PCAP file to HCCAPx file with a hashcat util
/usr/lib/hashcat-utils/cap2hccapx.bin wifu-01.cap output.hccapx

```

#### Hashcat modules

| type | module | example|
|------|----------|-------|
|WPA-EAPOL-PBKDF2 | -m 2500 | |
|WPA-EAPOL-PMK | -m 2501 | |
| WPA-PMKID-PBKDF2 | -m 16800 | | 
|WPA-PMKID-PMK | -m 16801 | 2582a8281bf9d4308d6f5731d0e61c61*4604ba734d4e*89acf0e761f4 |
|TTLS-CHAP | -m 4800 | ce8d3c0b4c5c9369ce426ba7d36d164e:38ddb29b0fea9243afb6fb9d6bb95bfb:2a |
|TTLS-MSCHAPv2| -m 5500 | user1::::f4ed9fe147deaed3bfb1a1744ce1908788c66d281b134a11:d98dd4b772ee831c |






### Password mutation

#### John
The rules to mutate passwords are in /etc/john/john.conf

- Rule to add 2 and 3 numbers at the end of the password:
```
$[0-9]$[0-9]
$[0-9]$[0-9]$[0-9]
```

Use --rules with john to apply rules

```bash
# See the wordlist in screen
john --wordlist=<wordlist file> --stdout

# We can see the variations that will be applied and see if password123 is generated
sudo john --wordlist=<path to wordlist> --rules --stdout |grep -i password123

# With --session the session is saved and can be restored the next time it is resumed from the last password tried 
john --wordlist=<wordlist file> --stdout --session=<session name> | aircrack-ng -w - -b <AP BSSID> <file.cap>

# We can stop it with "q" or "ctrl-c", and continue later from that point
john --restore=<session name> | aircrack-ng -w - -b <AP BSSID> <file.cap>

# We can pipe mutations to aircrack without saving to disk with "-w -" :
sudo john --wordlist=<wordlist> --rules --stdout | aircrack-ng -e wifu -w - <file.pcap>

# We can save and restore sessions also with wordlists generated on the fly with crunch 
crunch 8 8 | john --stdin --session=<session name> --stdout | aircrack-ng -w - -b <AP BSSID> <file.cap>

# and later restore with
crunch 8 8 | john --restore=<session name> | aircrack-ng -w - -b <AP BSSID> <file.cap>
```



#### Crunch
Generate new passwords, we need to say the minimum and maximum length (WPA requires passphrases between 8 and 63 chars). Crunch also allows us to specify a pattern with the _-t_ option with or without a character set. Different symbols in the pattern define the type of character to use.

-   _@_ represents lowercase characters or characters from a defined set
-   _,_ represents uppercase characters
-   _%_ represent numbers
-   _^_ represents symbols
```bash
# Create all combinations of words from 8 to 9 characters (a lot of output, not practical)
crunch 8 9

# Crate all combinations of words from 8 to 9 characters using only the characters: a,b,c,1,2 and 3:
crunch 8 9 abc123

# Create combinations of 11 chars formed by the word "Password" and 3 numbers
crunch 11 11 Password%%%

# equivalent:
crunch 11 11 0123456789 -t password@@@

# generate unique combinations from a set (in this case the min and max lengths are ignored but we need to provide them so that the program doesn't fail)
crunch 1 1 -p abcdefg1234

# Generate unique words from some words (it combines them)
crunch 1 1 -p january february march

# Generate patterns with the words we say (they are replaced in the "d")
crunch 5 5 -t ddd%% -p january february march

# If instead of the % we use @ crunch adds lowercase letters instead of numbers

# it replaces the letters "aADE" in the @@ and the words in the "d" letters
crunch 5 5 aADE -t ddd@@ -p january february march

# pipe output from crunch to aircrack:
crunch 5 5 aADE -t ddd@@ -p january february march | aircrack-ng -e wifu file.pcap -w -

```


#### Mangler
Ruby script to mutate passwords
--allow-duplicates is usually worth it because of the time we save in not checking if there are duplicates

```bash
# Mutate words of a file
rsmangler --file file.txt
cat file.txt | rsmangler --file -

# Limit the size of the generated words
rsmangler --file wordlist.txt --min 12 --max 13

# Pipe to aircrack (don't use --output, that is only to save to disk)
rsmangler --file wordlist.txt --min 12 --max 13 | aircrack-ng -e wifu file.pcap -w -
```

A set of three words generates about 6,000 passwords, four words generates about 23,000 passwords, and five words generates about 125,000 passwords. We need to take care to ensure the wordlist we begin with is a reasonable size.


 ### coWPAtty

Tool that recovers WPA pre-shared keys using both dictionary and rainbow table attacks. Although coWPAtty is not being developed anymore, it is still useful, especially when using its rainbow table attack method. Install it with ``sudo apt install cowpatty``

```bash
#Generate rainbow tables
genpmk -f <password file> -d hashes -s test

cowpatty -r wpa-01.pcap -d hashes -s test 
```

### extra WEP
Automatic tool. It sends packets to the WEP network that we are trying to attack. It may be necessary to run this command several times, sometimes it fails. 
```bash
besside-ng wlan0 -c 6 -b <BSSID>  
```

### wordlists
ftp://ftp.openwall.com/pub/wordlists/
http://www.openwall.com/mirrors/
https://github.com/danielmiessler/SecLists
http://www.outpost9.com/files/WordLists.html
http://www.vulnerabilityassessment.co.uk/passwords.htm
http://packetstormsecurity.org/Crackers/wordlists/
http://www.ai.uga.edu/ftplib/natural-language/moby/
http://www.cotse.com/tools/wordlists1.htm
http://www.cotse.com/tools/wordlists2.htm
http://wordlist.sourceforge.net/






