# Creating a Private Root Certificate Authority
In order to better understand how certificate authorities work, I set one up in my lab.  I did some research and ultimately based a lot of my configuration on information from a couple of sources.(1)(2)

## Research
My first attempt heavily followed this guide (1) and used 4096 bit keys for the Root CA Certificate and Intermediate Certificate.  This seemed to work well for webservers.  I then started using certificates for an iPXE server in the lab and ran into issues. After some digging, I realized iPXE was complaining about the size of the key on the intermediate certificate.  This led me to do more reasearch and to the conclusion that 2048 bit keys for the root and intermediate certificates seem to be best practices at this point.

## Process
The whole process relies on OpenSSL, so you will need a recent copy of that installed to create the certificates. If this were a real production environment, you would need to take precautions to never allow any access to the root CA private key and passphrase, because that would allow anyone to sign their own intermediate certificate and then sign any arbitrary ceritificates with that. Since this is only a lab environment we won't be taking all those precautions.

### Setup directories and files
Two files must be created to keep track of certificates, **index.txt** and **serial**.  The index keeps a record of certificates issued, and the serial file keeps track of the serial number of the next certificate issued.

```
> mkdir ca
> cd ca
> touch index.txt
> echo "1000" > serial
> mkdir cert crl key newcerts
> chmod 700 key
```
### Create openssl.cnf
```
[ ca ]
# `man ca`
default_ca = CA_default

[ CA_default ]
# Directory and file locations.
dir               = /root/ca
certs             = $dir/cert
crl_dir           = $dir/crl
new_certs_dir     = $dir/newcerts
database          = $dir/index.txt
serial            = $dir/serial
RANDFILE          = $dir/key/.rand

# The root key and root certificate.
private_key       = $dir/key/ca.key.pem
certificate       = $dir/cert/ca.cert.pem

# For certificate revocation lists.
crlnumber         = $dir/crlnumber
crl               = $dir/crl/ca.crl.pem
crl_extensions    = crl_ext
default_crl_days  = 30

# SHA-1 is deprecated, so use SHA-2 instead.
default_md        = sha256

name_opt          = ca_default
cert_opt          = ca_default
default_days      = 375
preserve          = no
policy            = policy_strict

[ policy_strict ]
# The root CA should only sign intermediate certificates that match.
# See the POLICY FORMAT section of `man ca`.
countryName             = match
stateOrProvinceName     = match
organizationName        = match
organizationalUnitName  = optional
commonName              = supplied
emailAddress            = optional

[ policy_loose ]
# Allow the intermediate CA to sign a more diverse range of certificates.
# See the POLICY FORMAT section of the `ca` man page.
countryName             = optional
stateOrProvinceName     = optional
localityName            = optional
organizationName        = optional
organizationalUnitName  = optional
commonName              = supplied
emailAddress            = optional

[ req ]
# Options for the `req` tool (`man req`).
default_bits        = 2048
distinguished_name  = req_distinguished_name
string_mask         = utf8only

# SHA-1 is deprecated, so use SHA-2 instead.
default_md          = sha256

# Extension to add when the -x509 option is used.
x509_extensions     = v3_ca

[ req_distinguished_name ]
# See <https://en.wikipedia.org/wiki/Certificate_signing_request>.
countryName                     = Country Name (2 letter code)
stateOrProvinceName             = State or Province Name
localityName                    = Locality Name
0.organizationName              = Organization Name
organizationalUnitName          = Organizational Unit Name
commonName                      = Common Name
emailAddress                    = Email Address

# Optionally, specify some defaults.
countryName_default             = US
stateOrProvinceName_default     =
localityName_default            =
0.organizationName_default      =
organizationalUnitName_default  =
emailAddress_default            =

[ v3_ca ]
# Extensions for a typical CA (`man x509v3_config`).
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid:always,issuer
basicConstraints = critical, CA:true
keyUsage = critical, digitalSignature, cRLSign, keyCertSign

[ v3_intermediate_ca ]
# Extensions for a typical intermediate CA (`man x509v3_config`).
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid:always,issuer
basicConstraints = critical, CA:true, pathlen:0
keyUsage = critical, digitalSignature, cRLSign, keyCertSign

[ usr_cert ]
# Extensions for client certificates (`man x509v3_config`).
basicConstraints = CA:FALSE
nsCertType = client, email
nsComment = "OpenSSL Generated Client Certificate"
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid,issuer
keyUsage = critical, nonRepudiation, digitalSignature, keyEncipherment
extendedKeyUsage = clientAuth, emailProtection

[ server_cert ]
# Extensions for server certificates (`man x509v3_config`).
basicConstraints = CA:FALSE
nsCertType = server
nsComment = "OpenSSL Generated Server Certificate"
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid,issuer:always
keyUsage = critical, digitalSignature, keyEncipherment
extendedKeyUsage = serverAuth

[ crl_ext ]
# Extension for CRLs (`man x509v3_config`).
authorityKeyIdentifier=keyid:always

[ ocsp ]
# Extension for OCSP signing certificates (`man ocsp`).
basicConstraints = CA:FALSE
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid,issuer
keyUsage = critical, digitalSignature
extendedKeyUsage = critical, OCSPSigning
```

### Create the private key
You will want to keep the private key and the passphrase in a secure location.  In this case, I used the passphrase **CHANGEME** as an example, but you will want a long, strong password in practice.
```
> openssl genrsa -aes256 -out key/ca.key.pem 2048
Generating RSA private key, 2048 bit long modulus
...............................................................+++
...+++
e is 65537 (0x10001)
Enter pass phrase for key/ca.key.pem: **CHANGEME**
Verifying - Enter pass phrase for key/ca.key.pem: **CHANGEME**
> chmod 400 key/ca.key.pem
```
### Create the root certificate
```
> openssl req -config openssl.cnf -key key/ca.key.pem -new -x509 -days 6600 -sha256 -extensions v3_ca -out cert/ca.cert.pem
Enter pass phrase for key/ca.key.pem:
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) [US]:
State or Province Name []:Kentucky
Locality Name []:Sample
Organization Name []:Sample's Own Example
Organizational Unit Name []:Sample's Own Example IT
Common Name []:Sample IT
Email Address []:sample-it@example.com
```

### Verify the root certificate
```
> openssl x509 -noout -text -in cert/ca.cert.pem
Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number: 14268503899016459974 (0xc603e8b2c926f6c6)
    Signature Algorithm: sha256WithRSAEncryption
        Issuer: C=US, ST=Kentucky, L=Sample, O=Sample's Own Example, OU=Sample's Own Example IT, CN=Sample IT/emailAddress=sample-it@example.com
        Validity
            Not Before: Dec 13 05:29:21 2019 GMT
            Not After : Jan  7 05:29:21 2038 GMT
        Subject: C=US, ST=Kentucky, L=Sample, O=Sample's Own Example, OU=Sample's Own Example IT, CN=Sample IT/emailAddress=sample-it@example.com
        Subject Public Key Info:
            Public Key Algorithm: rsaEncryption
                Public-Key: (2048 bit)
                Modulus:
                    00:a6:58:f4:5d:81:70:d8:89:15:cb:41:9b:56:be:
                    e0:b9:10:3f:e2:25:f1:dd:20:db:bc:ca:08:e2:7b:
                    c4:97:0a:31:26:ba:07:af:09:5f:bd:58:10:6f:5a:
                    2e:e7:d3:52:52:ba:a6:d2:90:dc:4d:56:70:7a:10:
                    83:a2:90:70:78:7b:6b:c0:78:5b:86:e8:42:6b:3c:
                    86:ce:1e:63:61:24:7a:75:bf:bb:34:c7:5f:cd:b8:
                    e0:78:bf:b0:b7:a5:3a:73:71:08:11:87:bc:f9:e9:
                    f2:c7:8c:2b:ea:87:c6:72:1a:a0:b6:f9:fe:7f:cc:
                    14:e6:57:0c:95:f8:a0:68:cc:61:9b:6b:61:5e:16:
                    c1:79:86:e8:b5:5d:e2:09:45:83:34:ea:d2:10:d7:
                    b4:d8:db:f1:c8:bd:ea:b6:ef:c0:17:88:8f:71:37:
                    ce:e1:28:27:e7:11:d9:95:8e:98:b9:67:94:55:ac:
                    36:25:92:b2:c5:d8:81:4f:37:ba:d2:25:65:58:7d:
                    ef:39:76:50:9f:8b:8b:9d:32:b6:44:11:15:f2:9d:
                    a3:66:b8:86:a8:f0:0c:10:99:f9:4e:4a:fd:35:1d:
                    2a:e0:f8:54:4a:fd:4a:5c:f8:54:13:f8:08:0c:ee:
                    40:a3:e0:65:f1:8b:e7:ea:cb:35:d4:f5:f8:68:f1:
                    25:8d
                Exponent: 65537 (0x10001)
        X509v3 extensions:
            X509v3 Subject Key Identifier: 
                8A:D8:30:86:7B:51:84:0F:0F:D3:29:BD:4A:28:F5:A9:74:1F:36:4C
            X509v3 Authority Key Identifier: 
                keyid:8A:D8:30:86:7B:51:84:0F:0F:D3:29:BD:4A:28:F5:A9:74:1F:36:4C

            X509v3 Basic Constraints: critical
                CA:TRUE
            X509v3 Key Usage: critical
                Digital Signature, Certificate Sign, CRL Sign
    Signature Algorithm: sha256WithRSAEncryption
         3d:45:a1:1a:01:46:a2:c6:25:a6:f7:5a:2f:7c:8b:cc:1a:43:
         f1:38:f6:5e:30:8c:8f:84:7b:7a:85:64:d6:b0:2e:43:13:d2:
         26:8c:43:78:7b:14:96:10:d1:dc:e0:e5:fc:ae:c1:3a:1f:4f:
         87:29:e1:ab:68:f9:10:bc:a6:7b:ac:4a:31:51:80:94:f0:8a:
         17:5c:d9:b6:95:49:94:9f:96:f6:cd:e2:04:c1:b3:0c:9d:d9:
         d5:bf:94:7c:e6:89:01:e0:49:ad:d0:80:7e:e0:49:55:20:73:
         16:fe:0c:1f:53:bf:6e:99:d2:aa:a6:cf:a8:c9:1f:f7:1c:58:
         13:0f:f1:d0:db:76:f1:bc:fb:be:1a:0e:1f:51:cb:48:14:35:
         b0:81:01:85:ae:6f:92:39:e2:c3:11:db:5a:0a:af:c6:a3:81:
         08:8d:bb:c1:b7:55:34:a8:f6:e8:da:7b:7c:55:f4:d5:0a:5e:
         48:bf:3f:58:93:d7:01:7f:12:55:e9:be:ca:c4:bf:89:f2:2a:
         0d:8e:bd:89:cb:97:9f:fd:cf:a4:09:e0:bb:d1:96:31:c0:0a:
         42:ee:de:c7:83:32:c0:94:cf:aa:9e:42:ca:89:74:54:1d:ca:
         ce:6a:f6:74:79:a1:e8:2a:4c:09:f5:2f:8b:12:e2:71:d4:b1:
         33:88:5a:d3
```


## Citations
1) [OpenSSL Certificate Authority](https://jamielinux.com/docs/openssl-certificate-authority/index.html)
2) [A Microsoft PKI Quick Guide](http://techgenix.com/microsoft-pki-quick-guide-part2-design/)
