# Season 2, Episode 1
In this episode we look at how to correctly host your HTML files, and reverse proxy the ws/ (Websocket) connections back to the Asterisk Service. It's all done on a single local instance so we are using a self signed certificate.

### Raspberry Pi Imager v 1.6.1
Raspberry Pi OS

**Make sure you write the ssh file to the root of the boot directory**

### Log into Raspberry Pi
Usging Putty (Windows) or Console (Mac/Linux), enter the following:
```
ssh pi@raspberrypi.local
```

The default passord for raspberry pi is: raspberry
```
$ sudo su
# apt-get update

# apt-get install ntp git samba
```

### Install Samba
```
# smbpasswd -a pi
# nano /etc/samba/smb.conf

[InnovateAsterisk]
path = /
browseable = yes
writeable = yes
read only = no
create mask = 0755
directory mask = 0755
guest ok = no
security = user
write list = pi
force user = root

# service smbd restart
# exit
```

### Asterisk 18
```
$ cd ~
$ wget http://downloads.asterisk.org/pub/telephony/asterisk/asterisk-18-current.tar.gz
$ git clone https://github.com/InnovateAsterisk/asterisk-opus.git
$ tar -xvf asterisk-[tab]
$ cd aster[tab]
$ sudo su
# contrib/scripts/install_prereq install
# apt-get install libopus-dev libopusfile-dev
# cp ../asterisk-opus/include/asterisk/* include/asterisk
# cp ../asterisk-opus/codecs/* codecs
# cp ../asterisk-opus/res/* res
# ./configure --with-pjproject-bundled
# make menuselect
# make && make install && make config
# exit
```

### Creating the OpenSSL Certificates
```
$ mkdir ~/ca && mkdir ~/certs && mkdir ~/csr
```

### Create Root CA Key
```
$ openssl genrsa -des3 -out ~/ca/RaspberryPi-Root-CA.key 4096
```
password: password

### Create Root Certificate Authority Certificate
```
$ openssl req -x509 -new -nodes -key ~/ca/RaspberryPi-Root-CA.key -sha256 -days 3650 -out ~/ca/RaspberryPi-Root-CA.crt
```
password: password

When asked complete fields like below:
```
Country Name (2 letter code) [AU]: GB
State or Province Name (full name) [Some-State]: None
Locality Name (eg, city) []: None
Organization Name (eg, company) [Internet Widgits Pty Ltd]: Innovate Asterisk
Organizational Unit Name (eg, section) []: www.innovateasterisk.com
Common Name (e.g. server FQDN or YOUR name) []: Raspberry Pi Root CA
Email Address []: conradjdw@gmail.com
```

### Generate Certificate Signing Request & Private Key
```
$ openssl req -new -sha256 -nodes -out ~/csr/raspberrypi.csr -newkey rsa:2048 -keyout ~/certs/raspberrypi.key
```

When asked complete fields like below:
```
Country Name (2 letter code) [AU]:GB
State or Province Name (full name) [Some-State]:Some State               
Locality Name (eg, city) []:Some City
Organization Name (eg, company) [Internet Widgits Pty Ltd]:My Private Company
Organizational Unit Name (eg, section) []:HQ
Common Name (e.g. server FQDN or YOUR name) []:raspberrypi.local
Email Address []:

Please enter the following 'extra' attributes
to be sent with your certificate request
A challenge password []:
An optional company name []:
```

### Generate Certificate
```
$ nano ~/csr/openssl-v3.cnf

authorityKeyIdentifier=keyid,issuer
basicConstraints=CA:FALSE
keyUsage = digitalSignature, nonRepudiation, keyEncipherment, dataEncipherment
subjectAltName = @alt_names

[alt_names]
DNS.1 = raspberrypi.local
```

### Generate Certificate
```
$ openssl x509 -req -in ~/csr/raspberrypi.csr -CA ~/ca/RaspberryPi-Root-CA.crt -CAkey ~/ca/RaspberryPi-Root-CA.key -CAcreateserial -out ~/certs/raspberrypi.crt -days 365 -sha256 -extfile ~/csr/openssl-v3.cnf
```

### PEM Combo Certificate
```
$ cat ~/certs/raspberrypi.crt ~/certs/raspberrypi.key > ~/certs/raspberrypi.pem
```

### Set Permission to Key
```
$ chmod a+r ~/certs/raspberrypi.key
```

### Git This Project
```
$ git clone https://github.com/InnovateAsterisk/S2E1.git
```
Copy the config folder to asterisk folder
```
$ sudo cp ~/S2E1/config/* /etc/asterisk
```

### Configure HTTP
```
$ sudo nano /etc/asterisk/http.conf

[general]
enabled=yes ; HTTP
bindaddr=127.0.0.1
bindport=8080
tlsenable=no ; HTTPS
enablestatic=no

$ sudo service asterisk restart
```

### Apache
```
$ sudo su
# apt-get install apache2
# a2enmod ssl
# a2enmod proxy
# a2enmod proxy_http
# a2enmod proxy_wstunnel

# nano /etc/apache2/ports.conf
Listen 0.0.0.0:80
Listen 0.0.0.0:443
Listen 0.0.0.0:4431 # Asterisk Websocket

# nano /etc/apache2/sites-enabled/000-default.conf
<VirtualHost 0.0.0.0:80>
        ServerName raspberrypi.local
        Redirect / https://raspberrypi.local/
</VirtualHost>

<VirtualHost 0.0.0.0:443>
        ServerName raspberrypi.local
        DocumentRoot /var/www/html
        ErrorLog ${APACHE_LOG_DIR}/error.log

        SSLEngine on
        SSLCertificateFile /home/pi/certs/raspberrypi.crt
        SSLCertificateKeyFile /home/pi/certs/raspberrypi.key
        SSLProtocol all -SSLv2 -SSLv3 -TLSv1
        SSLCipherSuite ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECD$
        SSLHonorCipherOrder on
        SSLCompression off
        SSLOptions +StrictRequire
</VirtualHost>

<VirtualHost 0.0.0.0:4431>
        ServerName raspberrypi.local
        DocumentRoot /var/www/html
        ErrorLog ${APACHE_LOG_DIR}/error.log

        ProxyRequests off
        ProxyPreserveHost On

        SSLEngine on
        SSLCertificateFile /home/pi/certs/raspberrypi.crt
        SSLCertificateKeyFile /home/pi/certs/raspberrypi.key
        SSLProtocol all -SSLv2 -SSLv3 -TLSv1
        SSLCipherSuite ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECD$
        SSLHonorCipherOrder on
        SSLCompression off
        SSLOptions +StrictRequire

        ProxyPass /ws ws://127.0.0.1:8080/ws
        ProxyPassReverse /ws ws://127.0.0.1:8080/ws
</VirtualHost>

# service apache2 restart
```

Check with
```
netstat -tunlp
```

### Add Some Users

```
; == Users

[User1](basic_endpoint,webrtc_endpoint)
type=endpoint
callerid="One Hundred" <100>
auth=User1
aors=User1
[User1](single_aor)
type=aor
mailboxes=User1@default
[User1](userpass_auth)
type=auth
username=User1
password=1234

[User2](basic_endpoint,webrtc_endpoint)
type=endpoint
callerid="Two Hundred" <200>
auth=User2
aors=User2
[User2](single_aor)
type=aor
[User2](userpass_auth)
type=auth
username=User2
password=1234

[User3](basic_endpoint,phone_endpoint)
type=endpoint
callerid="Three Hundred" <300>
auth=User3
aors=User3
[User3](single_aor)
type=aor
[User3](userpass_auth)
type=auth
username=User3
password=1234
```

### Update the Dialplan
```
[subscriptions]
exten => 100,hint,PJSIP/User1
exten => 200,hint,PJSIP/User2
exten => 300,hint,PJSIP/User3

[from-extensions]
exten => 100,1,Dial(PJSIP/User1,30)
exten => 200,1,Dial(PJSIP/User2,30)
exten => 300,1,Dial(PJSIP/User3,30)

exten => _[*0-9].,1,NoOp(Music On Hold)
exten => _[*0-9].,n,Ringing()
exten => _[*0-9].,n,Wait(2)
exten => _[*0-9].,n,Answer()
exten => _[*0-9].,n,Wait(1)
exten => _[*0-9].,n,MusicOnHold()

exten => e,1,Hangup()
```
