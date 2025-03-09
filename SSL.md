# SSL
Setup:
ircd listens on:
* 127.0.0.1 6679 and 6697
* ::1 6679 and ::1 6697

stunnel listens on:
* <your-external-ipv4> 6679 and 6697
* <your-external-ipv6> 6679 and 6697
.. and tunnels traffic to the local ports above

In this example IPv4 is `x.x.x.x` and IPv6 is `xxxx:xxx:xxxx::x`.

We will use apache to request and renewal the certificate. After the initial setup you can stop apache. It will be restarted automatically during the renewal process (see below).

Upgrade your ircd to 2.11.3 or higher.

```
apt-get install certbot python3-certbot-apache apache2
mkdir /var/www/irc.ircnet2.cz
```

Append to /etc/apache2/apache2.conf
```
<VirtualHost *:80>
  ServerName irc.warszawa.pl
  DocumentRoot /var/www/irc.warszawa.pl
</VirtualHost>
```

```
apachectl reload
certbot --apache
```

Request a certificate for irc.ircnet2.cz
```
certbot --apache
```

Now there should be a valid certificate at /etc/letsencrypt/live/irc.ircnet2.cz

Create bundle.pem
```
cat /etc/letsencrypt/live/irc.ircnet2.cz/fullchain.pem /etc/letsencrypt/live/irc.ircnet2.cz/privkey.pem > /etc/letsencrypt/live/irc.ircnet2.cz/bundle.pem
chmod 600 /etc/letsencrypt/live/irc.ircnet2.cz/bundle.pem
```

conf.P-Lines or ircd.conf
```
# SSL
P|127.0.0.1|||6679||T|x.x.x.x|
P|127.0.0.1|||6697||T|x.x.x.x|
P|::1|||6679||T|xxxx:xxx:xxxx::x|
P|::1|||6697||T|xxxx:xxx:xxxx::x|
```

/etc/network/if-up.d/stunnel
```
#!/bin/bash
[ "$IFACE" = "lo" ] || exit 0
ip rule add from 127.0.0.1/8 iif lo table 123
ip route add local 0.0.0.0/0 dev lo table 123
ip -6 rule add from ::1/128 iif lo table 123
ip -6 route add local ::/0 dev lo table 123
```
```
chmod +x /etc/network/if-up.d/stunnel
ifup -v -f lo
```

Install and configure stunnel:
```
apt-get install stunnel4
```

/etc/stunnel/stunnel.conf
```
pid = /var/run/stunnel.pid
output = /var/log/stunnel4/stunnel.log
foreground = no
fips = no
socket = l:TCP_NODELAY=1
socket = r:TCP_NODELAY=1

[ircd_6679]
accept  = x.x.x.x:6679
connect = 127.0.0.1:6679
cert = /etc/letsencrypt/live/irc.ircnet2.cz/bundle.pem
transparent = source

[ircd_6697]
accept  = x.x.x.x:6697
connect = 127.0.0.1:6697
cert = /etc/letsencrypt/live/irc.ircnet2.cz/bundle.pem
transparent = source

[ircd_6679_ipv6]
accept = xxxx:xxx:xxxx::x:6679
connect = ::1:6679
cert = /etc/letsencrypt/live/irc.ircnet2.cz/bundle.pem
transparent = source

[ircd_6697_ipv6]
accept = xxxx:xxx:xxxx::x:6697
connect = ::1:6697
cert = /etc/letsencrypt/live/irc.ircnet2.cz/bundle.pem
transparent = source
```

```
systemctl start stunnel4.service
```

Test each port
```
openssl s_client -4 -connect irc.ircnet2.cz:6679
openssl s_client -4 -connect irc.ircnet2.cz:6697
openssl s_client -6 -connect irc.ircnet2.cz:6679
openssl s_client -6 -connect irc.ircnet2.cz:6697
```
You should get:
```
:irc.ircnet2.cz 020 * :Please wait while we process your connection.
```

## Renew certificates
/etc/letsencrypt/renewal-hooks/pre/before-renewal.sh
```
#!/bin/bash
systemctl start apache2
```

/etc/letsencrypt/renewal-hooks/post/after-renewal.sh
```
#!/bin/bash
# If you started apache only for certificate renewal, you can stop it now.
systemctl stop apache2
cat /etc/letsencrypt/live/irc.ircnet2.cz/fullchain.pem /etc/letsencrypt/live/irc.ircnet2.cz/privkey.pem > /etc/letsencrypt/live/irc.ircnet2.cz/bundle.pem
kill -1 `cat /var/run/stunnel.pid`
```

```
chmod +x /etc/letsencrypt/renewal-hooks/pre/before-renewal.sh /etc/letsencrypt/renewal-hooks/post/after-renewal.sh
```

Crontab for root:
```
@daily certbot renew --apache > /dev/null 2>&1
```

## Link over SSL
Append to stunnel.conf:
```
[hub_contempt_chat]
client = yes
accept = 127.0.0.1:6696
connect = <hub-ip>:6697
local = x.x.x.x
```

Restart stunnel
Test connection:
```
$ nc 127.0.0.1 6696
:hub.contempt.chat 020 * :Please wait while we process your connection.
```

Change C/N Lines to connect to 127.0.0.1 6696
```
c|127.0.0.1|xxxxxxxxxxx|hub.contempt.chat|6696|1000||
N|127.0.0.1|xxxxxxxxxxx|hub.contempt.chat||1000|

```

## Start stunnel automatically
The default systemd script did not start stunnel properly after reboot. This script checks every 5 minutes if stunnel is running:
```
#!/bin/bash
ps -p `cat /var/run/stunnel.pid`
if [ $? = 1 ]
  then /usr/bin/stunnel4 /etc/stunnel/stunnel.conf
fi
```
Please check if the path of the PID file and of the stunnel binary are correct.
And it to crontab:

```
*/5 * * * * /root/cronjob_start_stunnel4.sh > /dev/null 2>&1
```

## Self-signed certificates
If the IP address of the server is not public, you can also use a self-signed certificate instead:
```
openssl req -new -x509 -days 10950 -nodes -out ircd.pem -keyout ircd.pem
```
and it it your stunnel.conf
```
cert = /etc/stunnel/ircd.pem
```
