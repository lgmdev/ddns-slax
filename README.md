# DHCP Cloudflare DDNS for Junos SRX

This SLAX operational script checks the router's public IPv4 address using
ipify and keeps a Cloudflare A record in sync. It uses TLS certificate
verification and updates only when the address differs. The record name comes
from Junos configuration: `system host-name` plus `system domain-name`. If no
domain name is configured, it appends `oio.dev`.

## Requirements

- A Cloudflare API token limited to the target zone, with **Zone / DNS / Read**
  and **Zone / DNS / Edit** permissions.
- Junos SLAX support for the libslax `curl` and `xutil` extensions.
- The Junos trusted CA bundle installed at
  `/var/db/certs/common/curl/curl-ca-bundle.crt` and readable by the curl
  extension runtime.

## Install

Copy the operational script from the repository root:

```sh
scp slax/ddns-cloudflare.slax lukasz@szn-srx:/var/db/scripts/op/
```

Create the token file on the SRX as root. The op script reads the file as the
account invoking `op ddns-cloudflare`, so make that account the owner. Replace
`lukasz` below if a different account runs the script. Paste the token after
starting `cat`, then press Ctrl-D; this avoids putting it in shell history.

```sh
start shell user root
umask 077
cat > /var/db/scripts/cf-api-token
```

Paste only the API token, then press Ctrl-D. Set owner and permissions:

```sh
chown lukasz:wheel /var/db/scripts/cf-api-token
chmod 600 /var/db/scripts/cf-api-token
ls -ld /var/db /var/db/scripts
ls -l /var/db/scripts/cf-api-token
exit
```

Verify as the same account that runs the op script that the token file is
readable. Keep mode `600`; do not make the API token world-readable. If using
the event-script copy, also verify that its runtime account can read the file.

Junos manages its default trusted-CA bundle through PKI; the managed bundle is
not guaranteed to be exposed as
`/var/db/certs/common/trusted-ca-cert/trusted_ca_certs.pem` on every device.
First check the Junos PKI bundle and connectivity to Juniper's CA CDN from
operational mode:

```text
request security pki ca-certificate ca-profile-group default-trusted-ca-certs download check-server
request security pki ca-certificate ca-profile-group default-trusted-ca-certs download status
```

If the status says the bundle is unavailable, request a download:

```text
request security pki ca-certificate ca-profile-group default-trusted-ca-certs download
request security pki ca-certificate ca-profile-group default-trusted-ca-certs download status
```

The script's libcurl needs a PEM file at the path below, separately from the
PKI-managed store. Check whether this SRX has the source PEM:

```sh
start shell user root
ls -la /var/db/certs/common
ls -l /var/db/certs/common/trusted-ca-cert/trusted_ca_certs.pem
```

If that file exists, install a regular copy at the path used by libcurl. Remove
any old symlink at the destination before copying:

```sh
start shell user root
mkdir -p /var/db/certs/common/curl
rm -f /var/db/certs/common/curl/curl-ca-bundle.crt
cp /var/db/certs/common/trusted-ca-cert/trusted_ca_certs.pem \
   /var/db/certs/common/curl/curl-ca-bundle.crt
chown root:wheel /var/db/certs/common/curl/curl-ca-bundle.crt
chmod 644 /var/db/certs/common/curl/curl-ca-bundle.crt
chmod 755 /var/db/certs/common/curl
chmod 755 /var/db/certs/common
ls -ld /var/db/certs/common /var/db/certs/common/curl
ls -l /var/db/certs/common/curl/curl-ca-bundle.crt
exit
```

If the source PEM does not exist but the Junos PKI status reports an available
bundle, securely copy the PEM bundle from a trusted, working SRX that has it, or
obtain a PEM CA bundle through your approved certificate-management process.
Do not fetch the CA bundle over an unverified HTTPS connection.

The `common` directory must allow traversal for the curl runtime. In particular,
if the runtime belongs to group `wheel`, that group must have directory execute
permission; “other” execute permission alone will not be used when the group
matches.

Verify TLS from the shell:

```sh
start shell user root
curl --cacert /var/db/certs/common/curl/curl-ca-bundle.crt \
     -Iv https://api.cloudflare.com/
exit
```

The output should include `SSL certificate verify ok`. IPv6 connection failures
are harmless if curl successfully falls back to IPv4.

### Temporary Insecure TLS Fallback

Use this only to diagnose CA-bundle problems. It disables certificate and
hostname verification, so HTTPS connections can be intercepted. Restore
verified TLS as soon as the CA bundle is accessible.

In `slax/ddns-cloudflare.slax`, add this line inside each of the three curl
option blocks: `$ipify-options`, `$get-options`, and `$write-options`:

```slax
<insecure>;
```

Copy the modified script to the op-script directory, and also to the
event-script directory if the 15-minute check is enabled. Remove all three
`<insecure>;` lines and redeploy both copies after resolving the CA issue.

Enable the operational script:

```text
configure
set system scripts op file ddns-cloudflare.slax
commit
```

## Run Manually

The resulting FQDN is printed as `Cloudflare DNS name`. For example:

- `system host-name szn-srx` and `system domain-name oio.dev` gives
  `szn-srx.oio.dev`.
- `system host-name szn-srx` with no `system domain-name` also gives
  `szn-srx.oio.dev`.
- A configured `system host-name` that already contains a dot is used as-is.

To use a different domain, configure it on the SRX. For example:

```text
configure
set system host-name wilcza-srx
set system domain-name oio.dev
commit
```

Make sure the resulting FQDN belongs to the Cloudflare zone configured in the
script.

Run the script from operational mode:

```text
op ddns-cloudflare
```

A successful unchanged-address run looks like:

```text
Public IPv4: 95.160.144.100
Cloudflare GET HTTP status: 200
IP unchanged. No Cloudflare update required.
```

If the public address changed, the script updates the existing DNS record with
PUT, or creates it with POST if the lookup found no record. A successful update
looks like:

```text
Public IPv4: 95.160.144.100
Cloudflare GET HTTP status: 200
Cloudflare PUT HTTP status: 200
SUCCESS: Cloudflare DNS updated.
```

## Run Every 15 Minutes

Copy the same SLAX script into the event-script directory:

```sh
scp slax/ddns-cloudflare.slax lukasz@szn-srx:/var/db/scripts/event/
```

Configure the recurring event and policy:

```text
configure
set event-options generate-event cloudflare-ddns-15min time-interval 900
set event-options policy cloudflare-ddns-check events cloudflare-ddns-15min
set event-options policy cloudflare-ddns-check then event-script ddns-cloudflare.slax
set event-options event-script file ddns-cloudflare.slax
commit
```

The timer generates an event every 900 seconds; the policy runs the event
script for each generated event. The manual op command remains available too.

To stop the periodic check:

```text
configure
delete event-options policy cloudflare-ddns-check
delete event-options generate-event cloudflare-ddns-15min
commit
```

## Troubleshooting

- `error setting certificate verify locations`: confirm the CA bundle is a
  regular file at the exact path above, mode `644`, and that each parent
  directory permits traversal by the curl runtime. Test with the shell
  `curl --cacert` command before rerunning the script.
- `Public IP lookup transport error`: check DNS, IPv4 connectivity, and TLS
  access to `api4.ipify.org` from the SRX.
- Cloudflare `401` or `403`: check the API token and its zone DNS Read/Edit
  permissions.
- Cloudflare GET returns `success: false`: inspect the API response printed by
  the script for the specific zone or query error.
