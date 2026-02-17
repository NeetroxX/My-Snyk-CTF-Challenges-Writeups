# <center>Voidstep Writeup</center>

**Date:** 25 December 2025  
**Prepared by:** Neetrox  
**Machine Author:** Neetrox  
**Difficulty:** Easy

---

## Scenario

You're a SOC analyst at Sfax-Tech. Your team lead rushes in: "A client server triggered multiple alerts. The system isolated it and saved the traffic." She hands you a USB with a PCAP file. Find out what happened. Time is critical.

---

## Artifacts Provided

**File:** Net-Traffic.PCAP  
**SHA256 Hash:** `3f57bbe369f78c92d79b22c31c6c7d93d30fabf8d95409c28d6db98828d06bb1`

---

## Initial Analysis

To begin the analysis, the password-protected ZIP file was unlocked using the password `hacktheblue`

---

## Questions & Answers

| Question | Answer |
|----------|--------|
| Q1: How many decoy hosts are randomized in this reconnaissance evasion technique? | 11 |
| Q2: What is the real attacker IP address? | 192.168.1.23 |
| Q3: How many open ports did the attacker find? | 4 |
| Q4: What web enumeration tool did the attacker use? | gobuster |
| Q5: What is the first endpoint discovered by the attacker? | /about |
| Q6: What was the first file extension tested during the enumeration? | html |
| Q7: What is the full vulnerable request parameter used in the attack? | file |
| Q8: What is the username discovered by the attacker? | zoro |
| Q9: What SSH-related file did the attacker try to access, enter the full path? | /home/zoro/.ssh/authorized_keys |
| Q10: When were the first brute force attempts occurred by the attacker to get the right password? | 09:02:47,19 |

---

## Detailed Writeup

### Q1: Decoy Hosts Detection

**Wireshark Filter Used:**
```
tcp.flags.syn==1 && tcp.flags.ack==1 && ip.src == 192.168.1.27
```

This filter shows SYN-ACK packets from the target host `192.168.1.27`.

![Decoy hosts analysis](assets/1.png)

**Analysis:**
- 11 decoy hosts were identified
- All packets share the same timestamp (09:01:02)
- Same source (192.168.1.27) contacts many different destination IPs

**Evidence:**  
Destinations identified: `192.168.1.23`, `37.17.134.155`, `17.185.172.74`, `92.168.10.2`, `111.67.234.66`, `164.226.167.106`, `119.43.115.176`, `190.206.212.93`, `219.26.47.118`, `35.51.129.206`, `49.16.108.198`

**Conclusion:**  
This is likely a decoy scan using `nmap --randomize-hosts -D RND:10`. The attacker hides the real scan by mixing many fake targets.

**Answer: 11**

---

### Q2: Real Attacker IP Address

Navigate to: **Statistics → Conversations → IPv4 tab**

This shows all communication pairs between IP addresses with traffic statistics.

![IP conversation statistics](assets/2.png)

**Traffic Volume Analysis:**
- `192.168.1.23` ↔ `192.168.1.27`: 235,790 packets, 26 MB
- `192.168.1.23` ↔ `172.19.0.2`: 463,476 packets, 52 MB

**Key Observation:**  
This IP address (`192.168.1.23`) shows massive traffic volume and is also included in the decoy scan from Question 1.

**Answer: 192.168.1.23**

---

### Q3: Open Ports Discovery

**Wireshark Filter Used:**
```
tcp.flags.syn==1 && tcp.flags.ack==1 && ip.src == 192.168.1.27
```

![Open ports identification](assets/3.png)

**TCP Handshake Analysis:**
- The attacker sends SYN packets to initiate a TCP handshake
- If a port is open, the target responds with a SYN-ACK packet

**Open Ports Found:**
- Port 22 (SSH)
- Port 5000
- Port 6789
- Port 8000

**Answer: 4**

---

### Q4: Web Enumeration Tool

**Wireshark Filter Used:**
```
http.request && ip.src == 192.168.1.23
```

![User-Agent string showing gobuster](assets/4.png)

**User-Agent String:**
```
User-Agent: gobuster/3.8
```

The User-Agent string is clearly visible in the HTTP request headers.

**Answer: gobuster**

---

### Q5: First Endpoint Discovered

**Wireshark Filter Used:**
```
http.response.code == 200
```

This filter shows all HTTP responses with status code 200 (OK), indicating successful requests to existing endpoints.

![First endpoint discovery](assets/5.png)

**Request Details:**
- Request URI: `/about`
- Full request URI: `http://192.168.1.27:5000/about`
- Timestamp: 09:01:27

**Timeline Analysis:**  
Looking at the timestamps, `/about` appears at 09:01:27 and was the first endpoint discovered.

**Answer: /about**

---

### Q6: First File Extension Tested

**Wireshark Filter Used:**
```
http.request && ip.src==192.168.1.23
```

This filter shows HTTP requests from the attacker's source IP.

![First file extension test](assets/6.png)

**Evidence:**
- Packet #25423 at 09:01:23: `GET /.bash_history.html`

**Timeline Analysis:**  
Looking at the earliest packets, the first file extension tested was `html`.

**Answer: html**

---

### Q7: Vulnerable Request Parameter

**Wireshark Filter Used:**
```
http.request.uri contains "etc" && http.request.uri contains "passwd"
```

This filter shows HTTP requests containing both "etc" and "passwd" in the URI, indicating a Local File Inclusion (LFI) attack.

![LFI attack parameter](assets/7.png)

**Full Vulnerable Request:**
```
GET /read?file=%2Fetc%2Fpasswd HTTP/1.1
```

**Parameter Identified:** `file`

**Answer: file**

---

### Q8: Username Discovery

Follow the HTTP stream of packet number 701782 and examine the `/etc/passwd` file contents.

![Username in /etc/passwd](assets/8.png)

**User Entry Found:**
```
zoro:x:1000:1000::/home/zoro:/bin/bash
```

**User Details:**
- Username: `zoro`
- UID: 1000 (typically the first regular user account)
- GID: 1000
- Home Directory: `/home/zoro`
- Shell: `/bin/bash` (full shell access)

**Answer: zoro**

---

### Q9: SSH-Related File Access

**Wireshark Filter Used:**
```
http.response.code==200
```

This filter shows successful HTTP responses to identify what files the attacker attempted to access.

![SSH authorized_keys access](assets/9.png)

**SSH-Related File Accessed:**
```
/home/zoro/.ssh/authorized_keys
```

**Answer: /home/zoro/.ssh/authorized_keys**

---

### Q10: SSH Brute Force Timing

After retrieving the username during directory brute-force, the attacker immediately launched an SSH brute-force attack.

![SSH brute force attempts](assets/10.png)

**Connection Pattern:**
- Multiple TCP connections from `192.168.1.23` to port 22 (SSH)
- TCP handshakes (SYN, SYN-ACK, ACK) followed by SSH protocol negotiation

**Brute Force Indicators:**
- Multiple rapid SSH connection attempts
- Same source IP (`192.168.1.23`) targeting SSH service
- Typical pattern of automated password guessing

**First Brute Force Attempts:** 09:02:47.19

**Answer: 09:02:47,19**

*End of Writeup*