# Creating a Private Intermediate Certificate Signing CA
Once the root certificate has been created, you can use it to sign an intermediate certificate that will be used to sign server certificates.  This is done as a preventative measure to secure the root certificate from compromise.  The root certificate's private key can be used once to sign the intermediate cert, and then be stored offline. The intermediate CA can then sign all the server certs. If the intermediate CA's private key were to be compromised, the root CA can revoke the intermediate CA's cert and issue a new one.

## Research
As mentioned in guid on creating a root CA, my first attempt heavily followed this guide (1) and used 4096 bit keys for the Root CA Certificate and Intermediate Certificate. After some bad experiencesI came to the conclusion that 2048 bit keys for the root and intermediate certificates seem to be best practices at this point, so this guide will continue down that path.

## Process
