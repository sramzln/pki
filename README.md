# Public Key Infratructure with OpenSSL


[Summary](#summary)
- [Introduction](#introduction)
- [Create infrastructure files and folders](#create-infrastructure-files-and-folders)
- [Generate root-ca and sub-ca public keys](#generate-root-ca-and-sub-ca-public-keys)
- [Generate root-ca public certificate](#create-root-ca-public-certificate)
- [Create a certificate request for sub-ca](#generate-a-certificate-request-for-sub-ca)
- [Generate sub-ca public certificate from csr](#Generate-sub-ca-public-certificate-from-csr)
    - [Verify the certificate request](#verify-the-certificate-request)
- [Create a private key for a server](#create-a-private-key-for-a-server)
- [Create a certificate request for a server from a config file](#create-a-certificate-request-for-a-server-from-a-config-file)
- [Sign a certificate request for a server with a config file](#sign-a-certificate-request-for-a-server-with-a-config-file)
    - [Verify the signed certificate](#verify-the-signed-certificate)
- [Add root-ca certificate to linux](#add-root-ca-certificate-to-linux)

- [Add SSL certificate to Nginx](#add-ssl-certificate-to-nginx)

## Introduction
Originally, the goal was to create a solid **Public Key Infrastructure** for my homelab. I tried to concatenate many documentations and best practices. I don't advise using these configurations in production. At least not for the moment. Comments are welcome.

I assume that these configuration files are based on this infrastructure:
```Bash
# tree /root
.
└── ca
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
    │   │   └── server.crt
    │   ├── crl
    │   ├── csr
    │   │   └── server.csr
    │   ├── csr.conf
    │   ├── newcerts
    │   └── private
    │       └── server.key
    └── sub-ca
        ├── certs
        │   └── sub-ca.crt
        ├── crl
        ├── csr
        │   └── sub-ca.csr
        ├── index
        ├── newcerts
        ├── private
        │   └── sub-ca.key
        ├── serial
        └── sub-ca.conf

```
## Create infrastructure files and folders
```Bash
# root/
mkdir -p ca/{root-ca,sub-ca,server}/{private,certs,newcerts,crl,csr}

# Modify private folders permissions
chmod -v 400 ca/{root-ca,sub-ca,server}/private

# Generate index files
# This is a file containing a unique identifier which will be
# auto-incremented as certificates are generated.
touch ca/{root-ca,sub-ca}/index
openssl rand -hex 16 > ca/root-ca/serial
openssl rand -hex 16 > ca/sub-ca/serial
```
## Generate root-ca and sub-ca public keys
```Bash
# root/ca/
openssl genrsa -aes256 -out root-ca/private/ca.key 4096
openssl genrsa -aes256 -out sub-ca/private/sub-ca.key 4096
```

## Generate root-ca public certificate
```Bash
# ca/root-ca/
# Common name : RootCA
openssl req -new -x509 -config root-ca.conf -key private/ca.key -days 7300 -extensions v3_ca -out certs/ca.crt
```

## Create a certificate request for sub-ca
```Bash
# ca/sub-ca/
# Common name : SubCA
openssl req -config sub-ca.conf  -new -key private/sub-ca.key -sha256 -out csr/sub-ca.csr
```

## Sign sub-ca public certificate from csr
```Bash
# ca/root-ca/
openssl ca -config root-ca.conf -extensions v3_intermediate_ca -days 3650 -notext -in ../sub-ca/csr/sub-ca.csr -out ../sub-ca/certs/sub-ca.crt
```

## Create a private key for a server
```Bash
# ca/server/
openssl genrsa -aes256 -out private/server.key 2048
```

## Create a certificate request for a server from a config file
```Bash
# ca/server/
openssl req -new -key private/server.key -out csr/sever.csr -config csr.conf
```

### Verify the certificate request
```Bash
# ca/server/
openssl req  -noout -text -in certs/server.csr
```

## Sign a certificate request for a server with a config file
```Bash
# ca/sub-ca/
openssl ca -config sub-ca.conf -days 365 -notext -in ../server/csr/server.csr \
-out ../server/certs/server.crt -extfile ../server/csr.conf -extensions v3_ext -sha256
```
> :bulb: **Variant:** There are many ways to sign a certificate. I used the sub-ca conf. file to point on my intermediate authority. But you can precise directly the CA to use as below.

```Bash
# ca/sub-ca/
openssl x509 -req -in ../server/csr/server.csr  -CA certs/sub-ca.crt -CAkey private/sub-ca.key \
-out ../server/certs/server.crt -days 365 -extensions v3_ext -extfile ../server/csr.conf -sha256
```

### Verify the signed certificate
```Bash
# ca/server/
openssl x509  -noout -text -in certs/server.crt
```

## Add root-ca certificate to linux
```Bash
# Solution 1°
sudo trust anchor --store ca.crt
# Solution 2°
# Copy your root certificate to this location and execute
sudo cp ca.crt /etc/ca-certificates/trust-source/anchors/
sudo update-ca-trust
```

## Add SSL certificate to Nginx
A standard Nginx SSL configuration is something like this:
```bash
server {
        listen       443 ssl;
        # You can add multiple servers separated by a comma. 
        server_name  your.domain.name;

        ssl_certificate      /path/to/bundle.crt # (or server);
        ssl_certificate_key  /path/to/server.key;

        ssl_session_cache    shared:SSL:1m;
        ssl_session_timeout  5m;

        ssl_ciphers  HIGH:!aNULL:!MD5;
        ssl_prefer_server_ciphers  on;

        location / {
            root   html;
            index  index.html index.htm;
        }
    }

```
 To have a trusted certificate chain we create a bundle with our server and sub-ca authority certificate:
 `cat server.crt sub-ca.crt > bundle.crt`


