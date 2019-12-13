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
create openssl.cnf
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


## Citations
1) [OpenSSL Certificate Authority](https://jamielinux.com/docs/openssl-certificate-authority/index.html)
2) [A Microsoft PKI Quick Guide](http://techgenix.com/microsoft-pki-quick-guide-part2-design/)
