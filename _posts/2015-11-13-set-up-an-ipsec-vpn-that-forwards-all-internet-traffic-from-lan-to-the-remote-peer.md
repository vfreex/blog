---
layout: post
title:  "Set up an IPSec VPN that forwards all Internet traffic from LAN to the remote peer"
date:   2015-11-13 19:32:00 +0800
categories: technology networking
tags: Linux VPN IPSec OpenWrt
---
My [PS4][] has trouble connecting to PSN service these days. I suspect my ISP has blocked all traffic to oversea PSN servers.
Fortunately there is an [OpenWrt][] router in my room, so that I can get through this kind of block by setting up a VPN connection to forward all traffic from my PS4 to my oversea VPS. Here is the expected connection solution:

`PS4 <---> OpenWrt Router <---> VPN tunnel <---> VPS <----> Internet`

I have two [VLAN][]s behind my OpenWRT router.
VLAN1 brings my laptop, smart phone, and other devices together, while VLAN2 is for PS4.

VLAN1 has IP address range `192.168.0.0/24` and VLAN2 has `192.168.1.0/24`. My aim is to forward all Internet traffic from `192.168.1.0/24` to the VPN tunnel but exclude all LAN traffic. Upon the successful establishment of my IPSec tunnel, devices in VLAN1 and VLAN2 can also get access to each other.

## Install strongSwan ##
I use [strongSwan][], an open source IPsec-based VPN software, to set up my IPSec tunnel. The first step is to install strongSwan on both VPS and OpenWrt router.

{% highlight bash %}
yum install strongswan # for Fedora, RHEL/CentOS with EPEL
apt-get install strongswan # for Debian/Ubuntu
opkg update && opkg install strongswan-full # for OpenWrt
{% endhighlight %}

## Certificate Authority ##
For security reasons, I choose RSA authentication with X.509 certificates for my VPN tunnel.
Both peers of strongSwan instances will identify themselves by corresponding X.509 certificates.

Start by creating a self singed root CA. Create a private key on your PC:

{% highlight bash %}
# create a 4096-bit RSA key named 'rootCA.key'
openssl genrsa 4096 > rootCA.key
# be care of the access permission
chmod 600 rootCA.key
{% endhighlight %}

Then generate a self singed root CA certificate:

{% highlight bash %}
# generate a self singed root CA certificate 'rootCA.crt', which will expire automatically after 3650 days.
openssl req -new -x509 -key rootCA.key -out rootCA.crt -days 3650 -subj '/C=US/O=My Company/CN=My VPN Root Certificate Authority'
{% endhighlight %}

You can view the details of your root CA certificate with the following command:

{% highlight bash %}
openssl x509 -noout -text -in rootCA.crt
{% endhighlight %}

Example output:

{% highlight bash %}
Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number: 12291557605863188796 (0xaa946284e1b9e93c)
    Signature Algorithm: sha256WithRSAEncryption
        Issuer: C=US, O=My Company, CN=My VPN Root Certificate Authority
        Validity
            Not Before: Nov 13 04:29:33 2015 GMT
            Not After : Nov 10 04:29:33 2025 GMT
        Subject: C=US, O=My Company, CN=My VPN Root Certificate Authority
        Subject Public Key Info:
            Public Key Algorithm: rsaEncryption
                Public-Key: (4096 bit)
                Modulus:
                    00:ad:e7:1f:7f:8c:06:96:76:ae:1c:48:97:6a:33:
                    bc:4e:cd:5b:3c:bc:a2:fd:b8:c0:2f:4c:fc:da:23:
                    ac:16:7b:9a:b5:43:d8:2a:68:57:54:48:22:c2:e2:
                    bb:c8:e0:ed:f7:16:b1:5d:32:48:bf:93:d2:a4:2c:
                    65:e0:d7:ee:6f:d8:c1:64:58:d5:58:2f:e0:47:94:
                    39:5d:8e:c6:66:aa:a9:21:44:a2:e6:b8:ca:0d:a3:
                    5d:9f:dd:81:98:e7:d8:77:0d:00:f2:cf:20:21:60:
                    1b:99:80:76:31:68:2a:72:ad:87:db:85:27:5c:69:
                    8b:06:ce:72:70:c6:c2:e7:a9:38:14:ad:b3:cb:24:
                    45:68:d3:b1:6b:2d:07:73:2c:5a:ba:16:42:77:1e:
                    41:4a:81:3b:76:03:cf:b7:c9:3d:c9:9c:b3:19:7d:
                    ee:aa:45:47:84:65:19:e7:bf:71:a3:a3:9b:b4:bd:
                    0e:51:2f:74:9b:2c:4e:b3:98:a3:91:ae:8e:d5:e1:
                    1b:c1:4a:10:58:2c:08:12:ec:38:4f:32:af:74:18:
                    4d:45:be:ea:f8:2f:cb:07:be:fd:dd:ed:fe:c0:bd:
                    13:a6:2b:0a:4e:4c:b9:f7:89:af:61:6b:bf:9f:42:
                    10:85:5f:bf:cc:68:cd:8b:82:eb:c6:14:bd:c4:18:
                    5e:3b:77:f1:4a:21:92:d2:c4:76:27:35:28:72:8e:
                    f0:c7:37:9f:fb:7b:b4:b2:90:17:7a:7e:dd:3a:eb:
                    a3:73:00:d9:86:db:38:06:5c:b8:08:a1:91:59:f7:
                    71:a2:ee:48:11:4a:89:21:32:6d:6e:d4:1d:58:68:
                    2b:82:8a:fd:36:2e:2d:d3:c9:aa:4e:b4:e5:c7:c4:
                    1a:64:f3:2b:ff:86:21:d5:91:77:e9:67:33:b7:d7:
                    77:97:c2:ce:23:66:7d:e2:16:05:0d:37:1c:95:8f:
                    ef:87:4a:01:e8:ad:91:00:8b:ad:39:cc:e5:f7:73:
                    60:c5:76:61:65:a9:27:db:30:33:b8:fd:24:69:93:
                    99:4d:e8:d5:e2:c9:14:da:6c:01:1c:c0:c1:c5:75:
                    69:35:6f:84:73:4d:d3:f2:9a:33:a6:b9:98:f5:d0:
                    3a:cf:d1:f3:43:56:59:98:49:d9:c2:b3:39:3d:49:
                    0c:4c:90:36:0e:dd:98:1e:55:a0:4a:28:93:c1:2a:
                    a0:e6:0e:e4:e9:14:0d:73:10:b2:09:80:fb:75:ff:
                    a2:b7:df:e6:7f:50:1f:a7:6c:25:ec:b8:3a:5e:03:
                    72:8a:67:cf:5e:97:90:ea:01:77:4c:70:7a:29:fc:
                    92:4f:ac:d3:23:4a:e1:2e:b2:18:21:67:4c:5f:77:
                    87:fa:9f
                Exponent: 65537 (0x10001)
        X509v3 extensions:
            X509v3 Subject Key Identifier:
                23:85:28:FE:44:D5:C3:20:99:27:0D:F0:1B:D2:E4:19:2F:E7:86:D2
            X509v3 Authority Key Identifier:
                keyid:23:85:28:FE:44:D5:C3:20:99:27:0D:F0:1B:D2:E4:19:2F:E7:86:D2
            X509v3 Basic Constraints:
                CA:TRUE
    Signature Algorithm: sha256WithRSAEncryption
         37:04:ce:97:fc:b8:ea:d8:97:9a:5f:a6:c5:1e:c8:cf:60:76:
         f3:88:ed:3c:c6:7e:9c:cd:a3:24:c3:69:13:82:7d:3f:77:6e:
         f2:f1:d9:af:08:9d:2e:09:d0:16:f2:14:27:d8:b2:0b:bc:b7:
         57:4a:bf:8d:ec:fb:91:fb:b9:95:23:86:02:07:94:91:fd:28:
         c2:ee:03:0b:e9:bf:62:ca:d2:e5:e6:24:96:1f:86:a2:b0:d8:
         80:a3:8b:cd:df:c6:b5:ff:42:e3:0d:72:23:4f:2b:e8:0e:54:
         0e:50:19:85:ab:26:33:0d:25:eb:a7:bf:82:7c:f3:77:a8:d0:
         dd:d4:6f:22:fb:c7:c6:e6:19:03:76:fd:29:ac:89:cc:1b:ff:
         dd:08:f8:f9:12:a8:f0:b9:3d:c5:fa:aa:83:59:d8:a7:5c:e6:
         e3:b4:61:33:42:7b:ca:87:27:18:82:1b:3c:9b:00:51:a4:5e:
         5f:c7:cb:5a:f2:2d:33:41:0b:6a:38:9f:4e:88:be:75:36:af:
         23:aa:99:26:fb:c8:24:54:03:09:00:54:96:2a:74:8c:c7:11:
         11:03:aa:81:31:fd:36:10:6e:43:80:d1:23:67:d0:93:a2:a9:
         76:8c:05:47:1f:ac:48:2b:7e:be:e2:b6:b2:f0:07:80:2d:90:
         06:7a:ee:0e:61:8c:f7:79:c9:1b:28:a6:10:8d:1d:69:11:2b:
         88:ee:e6:c0:7a:eb:f0:10:ad:1a:a5:7f:0e:a1:89:73:45:8d:
         8c:29:45:24:a7:1e:ed:9c:2e:13:04:aa:7b:53:5f:01:08:d7:
         cc:f2:6a:88:ac:95:a6:a5:3c:5a:9f:79:8a:b2:b4:26:79:57:
         26:c7:93:72:d3:fa:9a:99:7f:33:4a:c1:04:35:86:4f:ec:73:
         02:6e:dd:f5:e7:7e:ec:b5:fd:b5:74:cb:b9:16:ff:75:01:a1:
         b2:0d:01:85:04:0e:52:9c:de:42:40:9d:89:cf:38:a1:63:23:
         8f:37:63:d6:49:88:50:47:ab:4a:4c:79:de:1d:aa:64:25:0b:
         c4:6f:00:ec:80:9b:4c:04:07:10:0f:ce:62:23:aa:7d:d1:1a:
         62:f9:80:34:19:c0:4a:f0:4a:a1:a2:8c:d7:3d:df:d6:f3:c8:
         d9:c8:7e:14:87:52:c0:09:a2:c0:3b:d2:b3:a1:a4:3d:5b:6a:
         f9:f3:09:e7:1a:90:66:3a:d2:26:69:0b:5f:45:2d:13:8a:ae:
         43:a4:f5:e9:6f:09:6e:e1:59:51:e0:93:7a:53:73:11:75:2f:
         ad:ef:10:5d:b1:c9:4d:2a:9c:65:bb:82:0d:61:d4:63:e9:83:
         ce:bb:b4:e0:fa:40:49:3e
{% endhighlight %}

Create an X.509 v3 extension file `ext.cnf` containing the following content
in order to issue X.509 v3 certificates instead of v1 ones to end clients:

{% highlight bash %}
[ usr_cert ]
# Extensions for server certificates (`man x509v3_config`).
basicConstraints = CA:FALSE
nsCertType = server
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid,issuer:always
keyUsage = critical, digitalSignature, keyEncipherment
extendedKeyUsage = serverAuth, 1.3.6.1.5.5.8.2.2
{% endhighlight %}

Having the CA certificate, you are able to issue client certificates to your VPS and OpenWrt router later.

## Issue a Certificate For VPS ##

On your VPS, create a private key:

{% highlight bash %}
# create a 2048-bit RSA key named 'vpsHost.key'
openssl genrsa 2048 > vpsHost.key
# be care of the access permission
chmod 600 vpsHost.key
{% endhighlight %}

Then create a certificate request file:

{% highlight bash %}
# create a certificate request file 'vpsHost.csr'
openssl req -new -key vpsHost.key -out vpsHost.csr -subj '/C=US/O=My Company/CN=vpsHost'
{% endhighlight %}

Copy your certificate request file `vpsHost.csr` from VPS to your PC.

On your PC, use the following command to issue a client certificate to your VPS:

{% highlight bash %}
# issue a client certificate 'vpsHost.crt' to your VPS from your CA. The client certificate will expire automatically after 730 days.
openssl x509 -req -CA rootCA.crt -CAkey rootCA.key -CAcreateserial -in vpsHost.csr -out vpsHost.crt -days 730 -sha256 -extfile ext.cnf -extensions usr_cert
{% endhighlight %}

You can view the certificate details with the following command:

{% highlight bash %}
openssl x509 -noout -text -in vpsHost.crt
{% endhighlight %}

Example output:

{% highlight bash %}
Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number: 15485463545527761144 (0xd6e76b0c957178f8)
    Signature Algorithm: sha256WithRSAEncryption
        Issuer: C=US, O=My Company, CN=My VPN Root Certificate Authority
        Validity
            Not Before: Nov 13 05:19:06 2015 GMT
            Not After : Nov 12 05:19:06 2017 GMT
        Subject: C=US, O=My Company, CN=vpsHost
        Subject Public Key Info:
            Public Key Algorithm: rsaEncryption
                Public-Key: (2048 bit)
                Modulus:
                    00:b1:0e:85:7f:44:8a:91:f3:5f:f8:4d:41:68:86:
                    22:ce:d6:e1:a9:2a:75:09:f4:16:27:05:32:9d:fd:
                    da:a5:f7:24:ce:ac:29:9d:91:42:ef:7e:77:3d:09:
                    2f:f2:9e:82:a4:6c:fc:12:4b:39:01:73:fe:09:6d:
                    ea:9c:22:bf:ee:e6:ee:70:ea:00:c3:bf:92:c2:5f:
                    49:ae:f7:cf:90:26:d4:89:62:b5:87:e8:4c:57:1d:
                    d9:a2:f1:35:f4:2b:58:38:7e:d3:9f:03:fa:58:e3:
                    03:61:d6:2c:dc:6e:07:e2:4e:de:bd:0b:9c:97:2e:
                    0e:52:31:4b:22:8b:d8:a2:82:1d:0e:5d:f1:f4:01:
                    32:49:dc:2e:e8:e4:d9:03:8e:d5:f3:8e:3f:99:a7:
                    f2:b1:02:a9:e2:0c:0c:ff:77:72:6f:b9:8f:18:01:
                    42:0e:f2:75:92:6d:db:d0:fb:12:77:a8:6a:95:4c:
                    99:f4:a7:1b:30:67:34:4e:93:02:ab:1d:10:3e:b5:
                    68:75:d3:9d:a8:f6:42:90:87:96:2b:4f:c5:d6:81:
                    1b:ec:c0:32:be:af:59:64:04:25:ac:37:dc:fa:8b:
                    c8:35:5d:37:f3:fb:88:18:3a:a9:da:71:22:f0:fc:
                    ab:dd:c1:73:ab:b0:6b:46:d1:a3:9d:b4:4a:c8:46:
                    f7:b9
                Exponent: 65537 (0x10001)
        X509v3 extensions:
            X509v3 Basic Constraints:
                CA:FALSE
            Netscape Cert Type:
                SSL Server
            X509v3 Subject Key Identifier:
                79:E5:96:AB:4F:74:A1:8E:C6:EB:98:E9:A7:A7:52:8F:6B:6F:63:2A
            X509v3 Authority Key Identifier:
                keyid:23:85:28:FE:44:D5:C3:20:99:27:0D:F0:1B:D2:E4:19:2F:E7:86:D2
                DirName:/C=US/O=My Company/CN=My VPN Root Certificate Authority
                serial:AA:94:62:84:E1:B9:E9:3C

            X509v3 Key Usage: critical
                Digital Signature, Key Encipherment
            X509v3 Extended Key Usage:
                TLS Web Server Authentication, 1.3.6.1.5.5.8.2.2
    Signature Algorithm: sha256WithRSAEncryption
         4c:11:42:4c:af:df:4c:4e:99:16:41:4f:1f:5b:88:96:5c:ed:
         d3:b2:80:05:46:ab:12:1a:06:a8:f7:ce:6d:e3:e5:14:6c:88:
         3f:c4:e6:6e:28:87:1f:5b:51:dd:e5:15:66:2d:5a:49:2c:67:
         90:3b:1b:c6:6b:64:e4:56:0b:71:ab:95:3f:2c:a0:8e:12:bb:
         d5:89:27:cd:30:54:c6:94:bc:0f:7c:a9:b5:d3:06:0f:71:1f:
         5c:d5:c3:6a:0e:7e:db:68:cd:38:f2:b6:73:85:03:04:54:95:
         53:6c:8a:01:e8:87:19:46:f4:bd:9f:f4:cf:70:1a:fa:2d:fd:
         9c:01:c2:31:0e:43:5e:3b:b0:09:e8:9e:45:40:a5:bf:ce:ec:
         7a:1c:00:2f:56:28:c2:74:ce:60:ae:14:ba:dc:97:55:82:bc:
         08:2a:45:78:cb:f3:6f:aa:64:5e:0a:1c:86:09:50:29:50:fe:
         88:63:2d:23:16:62:35:c8:7a:69:a5:f7:70:ca:07:ef:a1:ca:
         cf:7b:32:d9:5b:61:e1:31:4f:c8:c1:22:b7:67:e9:d2:5a:60:
         ab:a0:0d:08:9f:1f:bd:a7:7f:b8:7f:fd:5d:c6:d0:dc:23:fc:
         8f:0c:14:fc:48:d8:3b:0a:cf:43:91:7c:7d:54:3f:77:41:11:
         de:a4:74:09:15:30:de:9a:77:a7:2f:99:e8:f8:8d:ff:81:fb:
         9a:51:bd:01:53:f5:ac:e0:e0:ea:65:69:0d:08:7f:c6:0b:f1:
         16:3e:fe:b3:06:de:24:dc:34:02:0c:d9:97:f4:60:5f:a4:95:
         07:3a:a5:c9:cb:1f:15:ee:fc:5b:60:04:be:d9:78:6a:03:63:
         aa:2b:8c:9f:6d:d4:80:93:05:8e:29:c7:2b:dd:90:14:78:a6:
         b8:c1:a7:ae:4c:a9:89:31:e5:eb:21:71:01:78:b9:22:e1:54:
         83:7d:fe:7e:40:83:42:98:c5:31:47:35:6d:71:8a:43:e7:1c:
         3b:f1:56:f6:49:be:6e:b6:e2:65:d8:ea:95:eb:ac:59:91:9f:
         8e:58:ed:79:14:50:d6:bf:cb:86:46:a0:6d:73:27:d7:65:7c:
         6f:73:7f:d0:45:84:da:f9:05:07:79:6a:d4:77:22:eb:d5:0f:
         44:c8:82:32:f6:b7:e5:89:1c:42:d7:af:b7:8c:0f:58:32:97:
         85:d2:45:78:28:32:cc:43:b0:67:17:0e:62:2a:cb:1f:8e:8c:
         c1:7b:ac:5b:84:a8:9b:96:dc:6d:d3:48:59:ff:2f:50:05:23:
         3b:4f:ec:c7:33:d8:61:d1:72:73:85:df:1c:67:01:ae:d5:29:
         be:e8:99:19:5a:cd:8d:fc
{% endhighlight %}

Copy your certificate files `rootCA.crt` and `vpsHost.crt` from your PC to your VPS.

## Issue a Certificate For OpenWrt Router ##

On your OpenWrt Router, create a private key:

{% highlight bash %}
# make sure you have installed openssl on your router
opkg update && opkg install openssl-util
# create a 2048-bit RSA key named 'openwrtHost.key'
openssl genrsa 2048 > openwrtHost.key
# be care of the access permission
chmod 600 openwrtHost.key
{% endhighlight %}

Then create a certificate request file:

{% highlight bash %}
# create a certificate request file 'openwrtHost.csr'
openssl req -new -key openwrtHost.key -out openwrtHost.csr -subj '/C=US/O=My Company/CN=openwrtHost'
{% endhighlight %}

Copy your certificate request file `openwrtHost.csr` from OpenWrt router to your PC.

On your PC, use the following command to issue a client certificate to your OpenWrt router:

{% highlight bash %}
# issue a client certificate 'openwrtHost.crt' to your VPS from your CA. The client certificate will expire automatically after 730 days.
openssl x509 -req -CA rootCA.crt -CAkey rootCA.key -CAcreateserial -in openwrtHost.csr -out openwrtHost.crt -days 730 -sha256 -extfile ext.cnf -extensions usr_cert
{% endhighlight %}

You can view the certificate details with the following command:

{% highlight bash %}
openssl x509 -noout -text -in openwrtHost.crt
{% endhighlight %}

Example output:

{% highlight bash %}
Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number: 15485463545527761145 (0xd6e76b0c957178f9)
    Signature Algorithm: sha256WithRSAEncryption
        Issuer: C=US, O=My Company, CN=My VPN Root Certificate Authority
        Validity
            Not Before: Nov 13 05:27:01 2015 GMT
            Not After : Nov 12 05:27:01 2017 GMT
        Subject: C=US, O=My Company, CN=openwrtHost
        Subject Public Key Info:
            Public Key Algorithm: rsaEncryption
                Public-Key: (2048 bit)
                Modulus:
                    00:cc:cd:49:e7:ab:d9:91:ed:05:6a:ae:56:bd:80:
                    bc:cf:8c:c5:db:ac:d7:2d:16:3c:ce:0d:f8:40:2d:
                    bd:60:fb:0b:37:d7:a4:b7:fe:a5:87:43:ea:c0:44:
                    d4:2e:b2:ae:53:09:7c:be:f0:d4:ec:38:c6:5d:54:
                    b8:d6:96:dd:3d:54:af:7f:94:a3:77:d5:1d:0d:0e:
                    71:54:59:c3:a3:19:ae:ca:10:de:cf:53:4a:7a:c1:
                    ca:c0:ff:00:8c:77:f3:15:2a:84:22:56:7d:58:80:
                    ea:60:51:82:ec:18:45:19:c6:7b:0a:b1:32:19:f4:
                    32:ad:2f:c9:dd:18:c1:ba:f6:c5:db:bf:0d:5d:91:
                    16:69:57:1f:03:f6:d7:87:be:15:58:3e:1a:3d:d6:
                    c6:80:0e:cf:97:ee:3a:12:fc:39:e2:40:d8:5f:6b:
                    6e:eb:6a:79:3a:4e:d6:ee:e3:8e:62:d7:ad:23:f7:
                    95:59:b0:01:a8:df:76:a7:51:df:39:80:87:fc:dc:
                    d5:cc:07:b1:8d:0a:2e:fa:8c:63:04:e3:01:0b:a7:
                    2f:aa:67:12:4e:18:d2:00:e6:e8:15:f0:cc:0a:d2:
                    7d:a7:e0:a3:3f:cb:8b:d2:dc:70:24:c7:d8:bd:e9:
                    04:d9:32:3e:1e:7f:ca:c4:9d:10:6d:1e:4d:0a:0e:
                    61:c5
                Exponent: 65537 (0x10001)
        X509v3 extensions:
            X509v3 Basic Constraints:
                CA:FALSE
            Netscape Cert Type:
                SSL Server
            X509v3 Subject Key Identifier:
                9D:4E:FA:66:23:C0:4D:06:DF:D4:36:31:CB:59:2D:F8:52:28:06:2D
            X509v3 Authority Key Identifier:
                keyid:23:85:28:FE:44:D5:C3:20:99:27:0D:F0:1B:D2:E4:19:2F:E7:86:D2
                DirName:/C=US/O=My Company/CN=My VPN Root Certificate Authority
                serial:AA:94:62:84:E1:B9:E9:3C

            X509v3 Key Usage: critical
                Digital Signature, Key Encipherment
            X509v3 Extended Key Usage:
                TLS Web Server Authentication, 1.3.6.1.5.5.8.2.2
    Signature Algorithm: sha256WithRSAEncryption
         43:05:ea:f4:be:8e:7b:9b:cb:8a:d9:22:3d:e3:b6:46:a4:1d:
         f2:2e:a6:75:88:1e:ec:66:78:53:b4:0c:80:5e:f3:7f:69:43:
         8f:ba:24:33:13:47:97:ab:ad:8e:0c:c8:05:ab:a2:3b:cd:92:
         12:72:7f:65:ac:fe:21:23:bf:87:3e:b3:88:12:b7:a7:69:df:
         eb:49:87:d7:3c:01:36:a1:41:8a:71:a5:f9:dd:e3:5f:41:35:
         bc:c9:04:56:20:dc:4c:59:e7:57:4f:a6:dd:5e:51:2f:47:7e:
         08:c5:36:92:74:37:31:13:82:9d:be:96:17:c9:58:45:be:a0:
         04:17:4c:59:c5:5f:bb:e6:50:86:05:08:f9:ee:b7:3d:9a:a8:
         c6:95:37:55:08:30:17:db:9f:41:e3:01:4e:33:7d:cd:ff:ca:
         bd:40:f6:7b:72:d0:59:6f:9e:40:24:20:cf:ed:e4:f4:3f:73:
         15:0c:45:01:fa:74:a3:53:79:a8:ae:38:9e:77:ac:f2:a7:12:
         7f:22:7b:a9:7f:03:9b:8d:48:9f:5d:7a:c6:c9:eb:df:00:9e:
         77:eb:4b:78:56:69:8f:67:7d:7f:ff:a5:da:93:64:5c:83:de:
         5e:53:0d:c1:1c:2e:0d:eb:ea:25:66:b3:56:20:e4:f7:06:11:
         87:2b:74:29:9d:71:8c:c9:67:cf:44:d9:0b:1c:d1:fd:4b:cb:
         0c:25:37:ee:a5:57:2d:5a:7c:cd:27:17:c6:20:8b:0f:19:4e:
         5c:d0:b9:e1:0c:f0:db:69:41:d9:24:e4:09:b6:a1:22:3a:95:
         6f:c2:c6:cb:c1:24:a0:75:7c:13:c3:52:16:4e:2b:f4:0a:c4:
         ce:6e:f7:39:ec:95:e0:4c:96:48:67:8b:29:70:09:06:ea:d1:
         a7:c7:19:99:d1:91:7d:76:59:ba:7f:06:83:d5:f0:b0:86:a4:
         d0:2a:f6:c8:20:55:60:69:02:4b:43:07:64:d9:75:24:ce:fa:
         3e:d4:47:c7:53:eb:93:0f:44:4c:18:b1:48:21:4e:de:07:37:
         a5:0b:70:17:cd:bb:9a:2a:98:6a:58:e3:86:b6:aa:06:87:2f:
         f6:e0:02:6f:ff:a9:8d:10:5c:df:1a:e6:dd:26:2c:48:08:96:
         54:c9:ce:f0:eb:5c:8e:f5:6f:d7:ef:c5:0a:8d:35:e0:a1:16:
         6f:34:9c:19:a1:70:72:c8:e5:ae:95:52:90:3a:00:ee:c2:cf:
         86:82:94:04:ec:ae:de:76:99:6d:2e:10:24:22:f0:46:c1:02:
         d6:cc:43:6c:8c:17:cf:8e:90:24:ce:c0:fd:41:cb:e7:46:45:
         bb:df:89:ce:10:96:4c:3d
{% endhighlight %}

Copy your certificate files `rootCA.crt` and `openwrtHost.crt` from your PC to your OpenWrt router.

## Configure VPS peer ##

On your VPS, copy `rootCA.crt` to `/path_to_strongSwan/ipsec.d/cacerts`:

{% highlight bash %}
cp rootCA.crt /etc/strongswan/ipsec.d/cacerts # for Fedora/RHEL/CentOS
cp rootCA.crt /etc/ipsec.d/cacerts # for Debian/Ubuntu
{% endhighlight %}

Copy `vpsHost.crt` to `/path_to_strongSwan/ipsec.d/certs`:

{% highlight bash %}
cp vpsHost.crt /etc/strongswan/ipsec.d/certs # for Fedora/RHEL/CentOS
cp vpsHost.crt /etc/ipsec.d/certs # for Debian/Ubuntu
{% endhighlight %}

Copy `vpsHost.key` to `/path_to_strongSwan/ipsec.d/private`:

{% highlight bash %}
cp vpsHost.key /etc/strongswan/ipsec.d/private # for Fedora/RHEL/CentOS
cp vpsHost.key /etc/ipsec.d/private # for Debian/Ubuntu
{% endhighlight %}

Tell strongSwan where to find the private key by editing `/path_to_strongSwan/ipsec.secrets`:

{% highlight bash %}
# /etc/strongswan/ipsec.secrets - strongSwan IPsec secrets file for Fedora/RHEL/CentOS
# /etc/ipsec.secrets - strongSwan IPsec secrets file for Debian/Ubuntu

: RSA vpsHost.key
{% endhighlight %}

Configure IPSec policy by editing `/path_to_strongSwan/ipsec.conf`. Here is an example:

{% highlight bash %}
# ipsec.conf - strongSwan IPsec configuration file. See https://wiki.strongswan.org/projects/strongswan/wiki/ConnSection
# /etc/strongswan/ipsec.conf - for Fedora/RHEL/CentOS
# /etc/ipsec.conf - for Debian/Ubuntu

# basic configuration

config setup
        # strictcrlpolicy=yes
        # uniqueids = no

# Add connections here.

conn %default # default configuration
        left=%any
        leftfirewall=yes # tell strongSwan to auto configure your firewall
        leftauth=pubkey # Use X.509 pubkey authentication for local peer
        leftcert=vpnHost.crt # tell strongSwan where to find the certificate file for local peer
        leftid=@vpnHost # must match the CN part of the certificate
        keyexchange=ikev2 # use IKEv2
        esp=aes192-aes256-sha1-sha256! # Cipher suit for ESP
        ike=aes192-aes256-sha256-modp4096-modp3072! # Cipher suit for IKE
        dpdaction=clear # Related to Dead Peer Detection. Clear connections when a dead peer is detected
        dpddelay=15m # Detect dead peer every 15 minutes
        fragmentation=yes # use IKE fragmentation (proprietary IKEv1 extension or IKEv2 fragmentation as per RFC 7383).

conn openwrtHost # connection to my OpenWrt router
        leftsubnet=0.0.0.0/0 # allow traffic to any IPv4 address
        rightauth=pubkey # Use X.509 pubkey authentication for remote peer
        right=%any # Since my OpenWrt router has a dynamic public IP address, I have to allow any IP address to connect.
        rightid=@openwrtHost must match the CN part of remote peer's certificate
        rightsubnet=192.168.1.0/24  # allow traffic from IPv4 address range 192.168.1.0/24
        auto=add # enable this configuration section
{% endhighlight %}

To make your VPS as a software router, you should turn on the following kernel options:

{% highlight bash %}
# enable IPv4 routing
sysctl -w net.ipv4.ip_forward=1
# make this option persistent after a system reboot
echo 'net.ipv4.ip_forward = 1' >> /etc/sysctl.conf
# if you want to enable IPv6 routing, run following commands:
sysctl -w net.ipv6.conf.all.forwarding=1
echo 'net.ipv6.conf.all.forwarding = 1' >> /etc/sysctl.conf
{% endhighlight %}

Enable [SNAT][] to allow forwarding packets from private IP addresses to the Internet, because your clients (PS4, etc) have no public IPv4 addresses. You should configure your Linux firewall:

{% highlight bash %}
# For firewalld
# make sure your firewall is running
firewall-cmd --direct --permanent --add-rule ipv4 filter FORWARD_direct 1 -o <interface to the Internet> -j ACCEPT
firewall-cmd --direct --permanent --add-rule ipv4 nat POSTROUTING_direct 1 -o <interface to the Internet> -m policy --pol none --dir out -j MASQUERADE # masquerade non-IPSec traffic
# if you want to enable IPv6 routing, run the following command:
firewall-cmd --direct --permanent --add-rule ipv6 filter FORWARD_direct 1 -o <interface to the Internet> -j ACCEPT
# reload your firewalld rules
firewall-cmd --reload

# For iptables
# make sure your firewall is running
iptables -A FORWARD -o <interface to the Internet> -j ACCEPT
iptables -t nat -A POSTROUTING -o <interface to the Internet> -m policy --pol none --dir out -j MASQUERADE  # masquerade non-IPSec traffic
# if you want to enable IPv6 routing, run the following command:
ip6tables -A FORWARD -o <interface to the Internet> -j ACCEPT
# save rules
service iptables save
{% endhighlight %}

Don't forget to start your strongSwan service and enable it at startup:

{% highlight bash %}
systemctl enable strongswan # For Fedora 14+/RHEL 7+/CentOS 7+
service strongswan enable # For old Fedora/RHEL/CentOS
{% endhighlight %}

## Configure OpenWrt Router ##
On your OpenWrt Router, copy `rootCA.crt` to `/etc/ipsec.d/cacerts`:

{% highlight bash %}
cp rootCA.crt /etc/ipsec.d/cacerts
{% endhighlight %}

Copy `openwrtHost.crt` to `/etc/ipsec.d/certs`:

{% highlight bash %}
cp openwrtHost.crt /etc/ipsec.d/certs
{% endhighlight %}

Copy `openwrtHost.key` to `/etc/ipsec.d/private`:

{% highlight bash %}
cp openwrtHost.key /etc/ipsec.d/private
{% endhighlight %}

Tell strongSwan where to find the private key. Edit `/etc/ipsec.secrets`:

{% highlight bash %}
# /etc/ipsec.secrets - strongSwan IPsec secrets file

: RSA openwrtHost.key
{% endhighlight %}

Configure IPSec policy by editing `/etc/ipsec.conf`. Here is an example:

{% highlight bash %}
# ipsec.conf - strongSwan IPsec configuration file. See https://wiki.strongswan.org/projects/strongswan/wiki/ConnSection
# /etc/strongswan/ipsec.conf - for Fedora/RHEL/CentOS
# /etc/ipsec.conf - for Debian/Ubuntu

# basic configuration

config setup
        # strictcrlpolicy=yes
        # uniqueids = no

# Add connections here.

conn %default # default configuration
        left=%any
        leftfirewall=no # tell strongSwan not to auto configure your firewall
        leftauth=pubkey # Use X.509 pubkey authentication for local peer
        leftcert=openwrtHost.crt # tell strongSwan where to find the certificate file for local peer
        leftid=@openwrtHost # must match the CN part of the certificate
        keyexchange=ikev2 # use IKEv2
        esp=aes192-aes256-sha1-sha256! # Cipher suit for ESP
        ike=aes192-aes256-sha256-modp4096-modp3072! # Cipher suit for IKE
        dpdaction=clear # Related to Dead Peer Detection. Clear connections when a dead peer is detected
        dpddelay=15m # Detect dead peer every 15 minutes
        fragmentation=yes # use IKE fragmentation (proprietary IKEv1 extension or IKEv2 fragmentation as per RFC 7383).

conn bypass # bypass LAN, multicast, and limited broadcast traffic
        leftsubnet=0.0.0.0/0
        leftsubnet=192.168.0.0/23, 224.0.0.0/4, 240.0.0.0/4
        auto=route

conn vpnHost # connection to my OpenWrt router
        leftsubnet=192.168.1.0/24 # allow traffic from 192.168.1.0/24 (VLAN2's IP range)
        right=<IP address or domain name of your VPS>
        rightauth=pubkey # Use X.509 pubkey authentication for remote peer
        rightid=@vpnHost must match the CN part of remote peer's certificate
        rightsubnet=0.0.0.0/0 # allow traffic to any IPv4 address
        auto=route # auto start this IPSec tunnel when such traffic occurs
{% endhighlight %}

Make sure your OpenWrt router have [SHA256][] kernel module installed:

{% highlight bash %}
opkg update && opkg install kmod-crypto-sha256
{% endhighlight %}

A very important step, don't let strongSwan configure routes on your OpenWrt router.
Otherwise strongSwan will insert an additional policy-based default route that forwards all traffic from VLAN2 to your VPS, which blocks the communication from VLAN2 clients to your OpenWrt router, resulting in your clients cannot receive any traffic from your router.
Edit `/etc/strongswan.conf` on your OpenWrt router:

{% highlight bash %}
# strongswan.conf - strongSwan configuration file
#
# Refer to the strongswan.conf(5) manpage for details
#
# Configuration changes should be made in the included files

charon {
        load_modular = yes
        plugins {
                include strongswan.d/charon/*.conf
        }
        install_routes = no # add this line
}
include strongswan.d/*.conf
{% endhighlight %}

You should also disable the [farp plugin][] for strongSwan.
The farp plugin will fake [ARP][] responses for `rightsubnet` to an established tunnel.
In our VPN deployment our `rightsubnet` (`0.0.0.0/0`) covers `leftsubnet` (`192.168.1.0/24`).
If we don't disable this plugin, it will fake ARP responses for `0.0.0.0/0`, which will disturb the ARP traffic in VLAN2 unexpectedly.
To disable the plugin, edit `/etc/strongswan.d/charon/farp.conf` on your OpenWrt router:

{% highlight bash %}
# /etc/strongswan.d/charon/farp.conf
farp {
    # Whether to load the plugin. Can also be an integer to increase the
    # priority of this plugin.
    load = no # change this line
}
{% endhighlight %}

The last step, add new firewall rules to allow VPN traffic on your OpenWrt router:

{% highlight bash %}
# /etc/firewall.user
# for IPv4 IPSec traffic
iptables -A input_rule -p esp -j ACCEPT # IPSec payload - ESP
iptables -A input_rule -p ah -j ACCEPT # IPSec payload - AH
iptables -A input_rule -p udp --dport 4500 -j ACCEPT -m conntrack --ctstate NEW # for IPSec NAT traversal (NAT-T)
iptables -A input_rule -p udp --dport 500 -j ACCEPT -m conntrack --ctstate NEW # for IKE
# for IPv6 IPSec traffic
ip6tables -A input_rule -p esp -j ACCEPT # IPSec payload - ESP
ip6tables -A input_rule -p ah -j ACCEPT # IPSec payload - AH
ip6tables -A input_rule -p udp --dport 4500 -j ACCEPT -m conntrack --ctstate NEW # for IKE
ip6tables -A input_rule -p udp --dport 500 -j ACCEPT -m conntrack --ctstate NEW # for IPSec NAT traversal (NAT-T)
{% endhighlight %}

Reload your OpenWrt firewall:

{% highlight bash %}
fw3 reload
{% endhighlight %}

Start and enable your strongSwan service:

{% highlight bash %}
/etc/init.d/ipsec start
/etc/init.d/ipsec enable
{% endhighlight %}

Now all Internet traffic from VLAN2 should be forwarded to the VPS. Please enjoy it.


[PS4]: https://www.playstation.com/en-us/explore/ps4/
[OpenWrt]: https://www.openwrt.org/
[VLAN]: https://en.wikipedia.org/wiki/Virtual_LAN
[strongSwan]: https://strongswan.org/
[SNAT]: https://en.wikipedia.org/wiki/Network_address_translation
[SHA256]: https://en.wikipedia.org/wiki/SHA-2
[farp plugin]: https://wiki.strongswan.org/projects/strongswan/wiki/Farpplugin
[ARP]: https://en.wikipedia.org/wiki/Address_Resolution_Protocol
