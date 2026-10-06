# Install slax on the srx box 

## on the srx 
```start shell

echo 'YOUR_CLOUDFLARE_API_TOKEN' > /var/db/scripts/cf-api-token
chmod 600 /var/db/scripts/cf-api-token
exit
```

## copy slax script 
scp dhcp-cloudflare.slax lukasz@szn-srx:/var/db/scripts/op/

## configure as operationa script
configure
set system scripts op file dhcp-cloudflare.slax
commit

## Setup CA path for CURL in junos (as root user)
```
start shell user root 

mkdir -p /var/db/certs/common/curl

ln -s ../trusted-ca-cert/trusted_ca_certs.pem \
      /var/db/certs/common/curl/curl-ca-bundle.crt
```
# test curl 
curl -Iv https://api.cloudflare.com/

## Test 
op dhcp-cloudflare

## Test result You should get approximately:
```DHCP interface: ge-0/0/7.20
DHCP state: RENEWING(LOCAL_CLIENT_STATE_RENEWING)
DHCP address: 95.160.144.100

Cloudflare HTTP status: 200
Cloudflare address: 95.160.144.100
IP unchanged. No Cloudflare update required.

Cloudflare address: 95.160.140.20
IP changed!
Old: 95.160.140.20
New: 95.160.144.100
SUCCESS: Cloudflare DNS updated.
srx-szn.oio.dev -> 95.160.144.100
```

## You can override the interface/hostname
op dhcp-cloudflare interface ge-0/0/7.20 hostname srx-szn.oio.dev

# Add the 15-minute automatic check

## instal script int event DB on SRX
/var/db/scripts/event/dhcp-cloudflare.slax

```
scp dhcp-cloudflare.slax lukasz@szn-srx:/var/db/scripts/event/
```
## configure event in SRX
```set event-options generate-event cloudflare-ddns-15min time-interval 900

set event-options policy cloudflare-ddns-check events cloudflare-ddns-15min
set event-options policy cloudflare-ddns-check then event-script dhcp-cloudflare.slax

set event-options event-script file dhcp-cloudflare.slax
commit 
```
# ddns-slax
