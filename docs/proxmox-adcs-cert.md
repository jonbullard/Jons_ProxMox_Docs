---
title: Signing Proxmox VE with a Windows AD CS Certificate
description: Issue a trusted SSL cert to a single Proxmox node from a single-tier Windows AD Certificate Authority.
---

# Signing Proxmox VE with a Windows AD CS Certificate

How-to for issuing a trusted SSL cert to a single Proxmox node from a single-tier
Windows AD Certificate Authority.

## Environment

- Node: `pve-0` (FQDN `pve-0.homelab.local`, IP `192.168.16.235`)
- CA: single-tier AD CS on Windows Server 2016 DC (`homelab-DC-CA`)
- Goal: trusted padlock on `https://pve-0.homelab.local:8006`

!!! note
    Adjust names, IPs, and CA host to match your own setup.

---

## Part 1: On pve-0 (generate key and CSR)

### 1. Create the OpenSSL config

Open a new CMD prompt, then connect to your Proxmox VE server via SSH.

```bash
ssh root@192.168.16.235
```
Create the OpenSSL config file.

```bash
nano /root/cert.cfg
```

Using the example provided below, copy and paste the contents from the example below into the cert.cfg file you created. Be sure to make the neccessary adjustments to match your environment needs.

```ini
[req]
distinguished_name = req_distinguished_name
req_extensions = v3_req
prompt = no

[req_distinguished_name]
C = US
ST = Georgia
L = Atlanta
O = HomeLab
OU = IT
CN = pve-0.'homelab.local'

[v3_req]
subjectAltName = @alt_names

[alt_names]
DNS.1 = pve-0.'homelab.local'
DNS.2 = pve-0
IP.1 = '192.168.16.235'
```

Once you have made the neccessary changes you will then save the changes and overwrite the file (Ctrl+X + Enter).

### 2. Generate the private key

```bash
openssl genrsa -out /etc/pve/nodes/pve-0/pveproxy-ssl.key 2048
```

### 3. Generate the CSR

```bash
openssl req -new -key /etc/pve/nodes/pve-0/pveproxy-ssl.key -out /tmp/pve-0.csr -config /root/cert.cfg
```

### 4. Print the CSR so you can copy it

```bash
cat /tmp/pve-0.csr
```

Copy the whole block, including the BEGIN and END lines.

---

## Part 2: On the Domain Controller (issue the cert)

### 5. Submit the CSR

Web enrollment (easiest if installed):

1. Browse to `http://localhost/certsrv`
2. Request a certificate > advanced certificate request
3. Paste the CSR into the Saved Request box
4. Certificate Template: Web Server
5. Submit
6. Choose Base 64 encoded, Download certificate, save as `pve-0.crt`

Or from an elevated Command Prompt:

```bat
certreq -submit -attrib "CertificateTemplate:WebServer" pve-0.csr pve-0.crt
```

!!! note
    Submit to the same CA that will issue the chain (here, the homelab-DC CA).

### 6. Export the root CA cert as Base64

```bat
certutil -ca.cert C:\root-ca-der.crt
certutil -encode C:\root-ca-der.crt C:\root-ca.pem
```

Open `C:\root-ca.pem` in Notepad. It must start with `-----BEGIN CERTIFICATE-----`.

!!! warning "Why Base64 matters"
    The root CA comes out as DER (binary) by default, and binary does not survive
    copy-paste through a text editor. Convert it to Base64 (PEM) on the DC first so
    it stays intact.

You now have two text files on the DC: `pve-0.crt` and `root-ca.pem`.

---

## Part 3: Move both certs to pve-0

Both files are Base64 text now, so copy-paste is safe.

### 7. Paste the leaf cert

```bash
nano /tmp/pve-0.crt
```

Paste, save.

### 8. Paste the root CA cert

```bash
nano /tmp/root-ca.pem
```

Paste, save.

### 9. Confirm both are valid PEM

```bash
head -1 /tmp/pve-0.crt
head -1 /tmp/root-ca.pem
```

Both must show `-----BEGIN CERTIFICATE-----`.

!!! danger "Never move binary through an editor"
    If either file shows garbled characters, it got mangled in transit. Re-export
    as Base64 and paste again. Do not move the DER/binary version through a text
    editor, the bytes get re-encoded and the cert becomes unreadable.

---

## Part 4: Assemble and apply on pve-0

### 10. Build the chained pem (leaf on top, root below)

```bash
cat /tmp/pve-0.crt /tmp/root-ca.pem > /etc/pve/nodes/pve-0/pveproxy-ssl.pem
```

### 11. Verify the SAN made it into the cert

```bash
openssl x509 -in /tmp/pve-0.crt -noout -text | grep -A1 "Subject Alternative Name"
```

You want to see `pve-0.homelab.local` (plus the short name and IP if you kept them).

### 12. Confirm the key and cert are a matched pair

```bash
openssl x509 -noout -modulus -in /etc/pve/nodes/pve-0/pveproxy-ssl.pem | openssl md5
openssl rsa  -noout -modulus -in /etc/pve/nodes/pve-0/pveproxy-ssl.key | openssl md5
```

!!! warning
    Both hashes must be identical. If they differ, the cert and key are from
    different generations. Regenerate from Part 1.

### 13. (Optional) Verify the chain validates against the root

```bash
openssl verify -CAfile /tmp/root-ca.pem /tmp/pve-0.crt
```

You want `pve-0.crt: OK`.

### 14. Restart the proxy

```bash
systemctl restart pveproxy
```

---

## Part 5: Trust and test

### 15. DNS

Confirm the DC has an A record for `pve-0` to `192.168.16.235` in the
`homelab.local` forward & lookup zone.

### 16. Test

From a domain-joined machine (already trusts the AD root CA via Group Policy),
browse to:

```text
https://pve-0.homelab.local:8006
```

Clean padlock means done.

- Name-mismatch warning: you used the wrong name. Use the FQDN.
- Untrusted warning: that machine does not have your AD root CA in its trust store.
  On a non-domain machine, import `root-ca.pem` into Trusted Root Certification
  Authorities manually.

---

## Notes

!!! tip "Certs bind to names, not ports"
    Certs bind to hostnames and IPs (SAN), never to ports. The cert covers
    `pve-0.homelab.local` on any port, including 8006.

- The cert files Proxmox reads are fixed paths:
    - `/etc/pve/nodes/pve-0/pveproxy-ssl.pem` (leaf + chain)
    - `/etc/pve/nodes/pve-0/pveproxy-ssl.key` (private key)
- On a cluster, repeat per node. Each node needs its own cert with that node's
  FQDN/IP in the SAN.
- Default cert lifetime from the Web Server template is typically 1 to 2 years.
  Set a reminder to repeat this before expiry.
