# Public Key Infratructure with OpenSSL

## Summary
[Public Key Infratructure with OpenSSL](#public-key-infratructure-with-openSSL)
- [Introduction](#introduction)
- [Generate infrastructure files and folders](#generate-infrastructure-files-and-folders)
- [Generate root-ca and sub-ca public keys](#generate-root-ca-and-sub-ca-public-keys)
- [Generate root-ca public certificate](#generate-root-ca-public-certificate)

## Introduction
Originally, the goal was to create a solid Public Key Infrastructure for my homelab. I tried to concatenate many documentations and best practices. I don't advise using these configurations in production. At least not for the moment. Comments are welcome.

I assume that these configuration files are based on this infrastructure:
```Bash
# tree /root/ca/
ca
├── root-ca
│   ├── certs
│   │   └── ca.crt
│   ├── crl
│   ├── csr
│   ├── index
│   ├── newcerts
│   ├── private
│   │   └── ca.key
│   ├── root-ca.conf
│   ├── serial
├── server
│   ├── certs
│   ├── crl
│   ├── csr
│   ├── newcerts
│   ├── private
│   └── server_template.conf
│   └── client_template.conf
├── sub-ca
│   ├── certs
│   │   └── sub-ca.crt
│   ├── crl
│   ├── csr
│   │   └── sub-ca.csr
│   ├── index
│   ├── private
│   │   └── sub-ca.key
│   ├── serial
│   └── sub-ca.conf
```
## Generate infrastructure files and folders
```Bash
# root/
mkdir -p ca/{root-ca,sub-ca,server}/{private,certs,newcerts,crl,csr}

# Modify private folders permissions
chmod -v 700 ca/{root-ca,sub-ca,server}/private

# Generate index files
# This is a file containing a unique identifier which will be auto-incremented as certificates are generated.
touch ca/{root-ca,sub-ca}/index
openssl rand -hex 16 > ca/root-ca/serial
openssl rand -hex 16 > ca/sub-ca/serial
```
## Generate root-ca and sub-ca public keys
```Bash
# root/
# Generate root-ca and sub-ca private keys
openssl genrsa -aes256 -out root-ca/private/ca.key 4096
openssl genrsa -aes256 -out sub-ca/private/sub-ca.key 4096
```

## Generate root-ca public certificate
```Bash
# root-ca/
openssl req -new -x509 -config root-ca.conf -key private/ca.key -days 7300 -extensiosns v3_ca -out certs/ca.crt
```