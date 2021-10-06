---
title: "Fix outdated Let's Encrypt root CA certificate on Ubuntu 14"
date: 2021-10-06 9:00 -0700
categories: [Linux]
tags: [linux, certificates, letsencrypt]
excerpt: "When the initial Let's Encrypt root/CA certificate expired, it needed to be manually replaced on outdated systems. Here is one way to do this."
classes: wide
header:
  image: /assets/images/powershell_header-1280x200.png
  teaser: /assets/images/powershell-250.png
  og_image: /assets/images/powershell-250.png
---
When the initial Let's Encrypt root CA certificate expired in September 2021, it needed to be manually replaced on outdated systems. Here is how I did this via command line (Bash & SSH) on a bunch of systems that due to their setup weren't able to get the update automatically.

---

On a system that has SSH access I stash a list of remote names and/or IP adresses in a variable:

```bash
iplist="192.168.1.3
192.168.2.100
10.10.0.5"
```

---

Then I define the list  of commands I want to run through the SSH connection. The reason for me to not issue these directly in the loop was the combination of the use of an `IF` condition and using a subshell for root access via sudo. Also this shows an example how to properly nest commands when tunneling through SSH - note the use of `'\''` to escape the single quotes.

```bash
export cmdlist='echo "22b557a27055b33606b6559f37703928d3e4ad79f110b407d04986e1843543d1  /tmp/ISRG_Root_X1.crt" > /tmp/ISRG_Root_X1.sha256
curl -k --output "/tmp/ISRG_Root_X1.crt"  "https://letsencrypt.org/certs/isrgrootx1.pem.txt"
sha256sum -c /tmp/ISRG_Root_X1.sha256 && sudo bash -c '\''
  mv -f /tmp/ISRG_Root_X1.crt /usr/share/ca-certificates/mozilla/ISRG_Root_X1.crt; 
  echo "mozilla/ISRG_Root_X1.crt" | sort -u -o /etc/ca-certificates.conf -m - /etc/ca-certificates.conf; 
  sed -i '\''s|^mozilla\/DST_Root_CA_X3\.crt|!mozilla/DST_Root_CA_X3.crt|'\'' /etc/ca-certificates.conf;
  update-ca-certificates
'\''
curl -I https://letsencrypt.org/certs/isrgrootx1.pem.txt'
```

This will create a text file that has the checksum of the certificate file in it. After downloading the new root CA certificate with disabled SSL check (because that would fail as my system doesn't trust any new certificates yet which use this new CA) the check file allows me to verify that the downloaded file hasn't been tempered with.

>I made sure to get the checksum from a valid copy of the file on my local system which already had this cert added automatically.
{: .notice--danger }

Once that check succeeded the file will be moved to the right place in `/usr/share/ca-certificates/` and added to the config while the old certificate will be disabled. Then the ca-certificates database is updated and another call of `curl` is issued to verify the system is working as expected.

---

The next step then is to cycle through the list of remote systems and issue the set of commands I've defined above.

```bash
for i in $iplist ; do ssh $i "$cmdlist"; done
```

---

Sometimes I may add a `sleep 5` or so to the loop if I need to be able to interrupt the process before moving on to the next system.
