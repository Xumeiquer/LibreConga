# Reconnaissance

First things first, if I want to get into the Conga, I have to see what services are exposed from Conga. Second, I'll have to see how the application sends the pairing information to the Conga.

## Booting up the Conga Wireless AP

To start the AP service in the Conga you must have to press the `power + mode buttons` for at least 3 seconds. After 3 seconds the Conga will emit a blinky sound.

The AP's SSID matches this pattern: `Conga<Model>_XXXXXX`, where `<Model>` is the type of the Conga and `XXXXXX` is a number, potentially a random number. This AP has no password so you can connect to it easily.

NOTE: This process might differ among Conga models.

## Scanning services

Once connected I check what IP I've got assigned. Running `ifconfig` or `ip a` will be more than enough to check the wireless interface.

I've got assigned `192.168.5.2` so commonly the gateway will be `192.168.5.1`. However, using [nmap](https://nmap.org) we can find the available host in the net.

```sh
sudo nmap -sn 192.168.5.0/24
Starting Nmap 7.91 ( https://nmap.org ) at 2021-02-19 20:41 CET
Nmap scan report for 192.168.5.1
Host is up (0.0073s latency).
MAC Address: <redacted> (<redacted>)
Nmap scan report for my-host (192.168.5.2)
Host is up.
Nmap done: 256 IP addresses (2 hosts up) scanned in 15.00 seconds
```

NOTE: I redacted some bits for the sake of privacy.

Now, I already know where is the AP. It is at 192.168.5.1 as I already guessed.

The next step is to find out what services are available. Again, using nmap it is quite easy to find out the list of services.

```sh
sudo nmap -p- 192.168.5.1
Starting Nmap 7.91 ( https://nmap.org ) at 2021-02-19 20:45 CET
Nmap scan report for 192.168.5.1
Host is up (0.0034s latency).
Not shown: 65530 closed ports
PORT     STATE SERVICE
22/tcp   open  ssh
53/tcp   open  domain
6000/tcp open  X11
6008/tcp open  X11:8
8111/tcp open  skynetflow
MAC Address: <redacted> (<redacted>)

Nmap done: 1 IP address (1 host up) scanned in 23.77 seconds
```

NOTE: I redacted some bits for the sake of privacy.

Well, I dig a little bit more into the ports and I got the following output.

```sh
sudo nmap -sV -p22,53,6000,6008,8111 192.168.5.1
Starting Nmap 7.91 ( https://nmap.org ) at 2021-02-19 20:55 CET
Nmap scan report for 192.168.5.1
Host is up (0.33s latency).

PORT     STATE SERVICE     VERSION
22/tcp   open  ssh         Dropbear sshd 2015.71 (protocol 2.0)
53/tcp   open  domain      dnsmasq 2.75
6000/tcp open  X11?
6008/tcp open  X11:8?
8111/tcp open  skynetflow?
```

Port 22 is quite interesting but I need a user and password. Checking for vulnerabilities I can see Dropbear version 2015.71 has [3 CVE](https://www.cvedetails.com/vulnerability-list/vendor_id-15806/product_id-33536/version_id-191913/Dropbear-Ssh-Project-Dropbear-Ssh-2015.71.html) and 2 of them remote and quite interesting.

I was unable to exploit the vulnerabilities regarding Dropbear. I didn't find an exploit for CVE-2017-9078 and I found an exploit for CVE-2016-3116, it could be due to the exploit didn't work or the most probable it is due to my lack of knowledge in the pentesting area.

On the other hand, port 53 is also open so let's try to see whether it translate domain names.

Using the short options.

```sh
dig +short localhost @192.168.5.1
127.0.0.1
```

Using the long options.

```sh
dig localhost @192.168.5.1

; <<>> DiG 9.10.6 <<>> localhost @192.168.5.1
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 11050
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 1280
;; QUESTION SECTION:
;localhost.			IN	A

;; ANSWER SECTION:
localhost.		0	IN	A	127.0.0.1

;; Query time: 1 msec
;; SERVER: 192.168.5.1#53(192.168.5.1)
;; WHEN: Fri Feb 19 21:35:29 CET 2021
;; MSG SIZE  rcvd: 54
```

Lastly, I was unable to get more information about ports 6000, 6008, and 8111.
