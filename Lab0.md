> Charles Obser
# 1. Installing Wireshark on Arch Linux
1. Update the system and install the package
```bash
sudo pacman -Syu
sudo pacman -S wireshark-qt
```
2. Add user to the wireshark group to not have to launch it as super user each time
```bash
sudo usermod -aG wireshark $(whoami)
```
3. Reboot and launch wireshark
```bash
wireshark
```
# 2. Website Capture Packet
- https://www.canada.ca/en.html
![](attachements/Pasted%20image%2020260922142906.png)

- The IP is 23.49.140.216 and the port is 443
### 2.2 What is the IP address and port of your PC when initially accessing the page?
- My IP is 10.38.161.141 and my port is 54014
### 2.3 What is the process of the TCP three-way handshake?
- SYN: Client sends SYN to server.
![](attachements/Pasted%20image%2020260922143540.png)
- SYN-ACK: Server replies with SYN-ACK.
![](attachements/Pasted%20image%2020260922143643.png)
- ACK: Client sends ACK. Connection established.
![](attachements/Pasted%20image%2020260922144521.png)
# 3. Use the filter `dns` to find a DNS packet and answer the following questions:

![](attachements/Pasted%20image%2020260922150239.png)
### 3.1 What is the IP address and port of the DNS server?
- The IP of the DNS server is 168.95.1.1 and the port is 53
### 3.2 What is the domain name in this query?
- The domain name is in the query is www.canada.ca
### 3.3 Which protocol(s) does this DNS packet use? (List the protocols from Layer 2 — Link Layer — up to Layer 5 — Application Layer in the TCP/IP five-layer model.)
![](attachements/Pasted%20image%2020260922150716.png)
Layer 2 (Link Layer): Ethernet II  
Layer 3 (Network Layer): Internet Protocol Version 4 (IPv4)  
Layer 4 (Transport Layer): User Datagram Protocol (UDP)  
Layer 5 (Application Layer): Domain Name System (DNS)
# 4. Access an HTTP page and answer the following questions:
### 4.1 Which HTTP page did you access?
- httpforever.com
### 4.2 What is the IP address and port of the server hosting this page?
![](attachements/Pasted%20image%2020260922152604.png)
- The IP is 10.4.21.4.210 and the port is 80

### 4.3 What is the request method?
- This is a GET method
### 4.4 What is the response status code, and what does it mean?
![](attachements/Pasted%20image%2020260922153030.png)
The response is 200 OK, meaning that the request was successful, and the server has successfully returned the web page resource.