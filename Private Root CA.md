# Creating a Private Root Certificate Authority
In order to better understand how certificate authorities work, I set one up in my lab.  I did some research and ultimately based a lot of my configuration on information from a couple of sources.(1)(2)

## Research
My first attempt heavily followed this guide (1) and used 4096 bit keys for the Root CA Certificate and Intermediate Certificate.  This seemed to work well for webservers.  I then started using certificates for an iPXE server in the lab and ran into issues. After some digging, I realized iPXE was complaining about the size of the key on the intermediate certificate.  This led me to do more reasearch and to the conclusion that 2048 bit keys for the root and intermediate certificates seem to be best practices at this point.

## Process
The whole process relies on OpenSSL, so you will need a recent copy of that installed to create the certificates. If this were a real production environment, you would need to take precautions to never allow any access to the root CA private key and passphrase, because that would allow anyone to sign their own intermediate certificate and then sign any arbitrary ceritificates with that. Since this is only a lab environment we won't be taking all those precautions.

### Setup directories and files
Two files must be created to keep track of certificates, **index.txt** and **serial**.  The index keeps a record of certificates issued, and the serial file keeps track of the serial number of the next certificate issued.

```
~> mkdir ca
~> touch index.txt
~> echo "1000" > serial
~> mkdir cert crl key
```

## Citations
1) [OpenSSL Certificate Authority](https://jamielinux.com/docs/openssl-certificate-authority/index.html)
2) [A Microsoft PKI Quick Guide](http://techgenix.com/microsoft-pki-quick-guide-part2-design/)
