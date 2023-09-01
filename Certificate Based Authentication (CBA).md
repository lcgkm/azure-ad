# Certificate Based Authentication (CBA)

## Prepare PKI Certificate-Chain
### Generate Root CA
```
# 001 Create a self signed root certificate

$ openssl req -subj '/CN=CBA Test Root CA/O=CBA Test Dept/C=US' -new -newkey rsa:4096 -sha256 \
    -addext basicConstraints=critical,CA:TRUE \
    -addext keyUsage=critical,digitalSignature,keyCertSign,cRLSign \
    -addext authorityKeyIdentifier=keyid,issuer:always \
    -addext subjectKeyIdentifier=hash \
    -days 365 -nodes -x509 -keyout root.key -outform DER -out root.cer

$ openssl x509 -inform DER -in root.cer -text -noout
```
### Generate Intermediate CA
```
# 002 Create an intermediate CA certificate (signed by the root CA)
$ openssl req -subj '/CN=CBA Test Intermediate CA/O=CBA Test Dept/C=US' \
    -new -nodes -keyout intermediate.key -out intermediate.csr -days 180

$ openssl req -text -noout -verify -in intermediate.csr

$ cat > intermediate.ext <<EOL
[ intermediate_ca ]
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid, issuer:always
basicConstraints = critical, CA:true, pathlen:0
keyUsage=critical,digitalSignature,keyCertSign,cRLSign
EOL

$ openssl x509 -req -in intermediate.csr \
               -CA root.cer -CAform DER -CAkey root.key -CAcreateserial \
               -out intermediate.cer -outform DER -extensions intermediate_ca -extfile intermediate.ext

$ openssl x509 -inform DER -in intermediate.cer -text -noout
```
### Generate Entity Certificate
```
# 003 Create an entity certificate (signed by the intermediate CA)

$ openssl req -subj '/CN=CBA Test Entity Certificate/O=CBA Test Dept/C=US' \
    -new -nodes -keyout entity.key -out entity.csr -days 90
$ openssl req -text -noout -verify -in entity.csr

# 1.3.6.1.4.1.311.20.2.2 - Smart Card Logon
# 1.3.6.1.4.1.311.20.2.3 - UPN (User Principal Name)
$ cat > entity.ext <<EOL
[ entity ]
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid:always, issuer:always
basicConstraints = critical, CA:false
keyUsage = critical, nonRepudiation, digitalSignature, keyEncipherment
extendedKeyUsage = critical, clientAuth, emailProtection, msSmartcardLogin
certificatePolicies=1.2.3.4
subjectAltName= @alt_names

[alt_names]
email= xxx@xxx.com
otherName=1.3.6.1.4.1.311.20.2.3;UTF8:xxx@xxx.com
EOL

$ openssl x509 -req -in entity.csr \
               -CA intermediate.cer -CAform DER -CAkey intermediate.key -CAcreateserial \
               -out entity.cer -outform DER -extensions entity -extfile entity.ext

$ openssl x509 -inform DER -in entity.cer -text -noout
$ openssl x509 -inform der -in entity.cer -out entity.crt
$ openssl pkcs12 -inkey entity.key -in entity.crt -export -out entity.pfx
```

## Configure the certificate authorities
### CRL
```
Only one CRL Distribution Point (CDP) for a trusted CA is supported. The CDP can only be HTTP URLs. Online Certificate Status Protocol (OCSP) or Lightweight Directory Access Protocol (LDAP) URLs aren't supported.

To configure your certificate authorities in Azure Active Directory, for each certificate authority, upload the following:

The public portion of the certificate, in .cer format
The internet-facing URLs where the Certificate Revocation Lists (CRLs) reside
```

Configure certification authorities using PowerShell

https://learn.microsoft.com/en-us/azure/active-directory/authentication/how-to-certificate-based-authentication?WT.mc_id=Portal-Microsoft_AAD_IAM#configure-certification-authorities-using-powershell

Smart Card Deployment: Manually Importing User Certificates

https://support.yubico.com/hc/en-us/articles/360015668799-Smart-Card-Deployment-Manually-Importing-User-Certificates

We can generate private key and CSR file by Yubikey，and sign the CSR to generate User Certificate. And then import User Certificate to Yubikey.

https://developers.yubico.com/yubikey-piv-manager/Certificates.html

## Reference

https://learn.microsoft.com/en-us/azure/active-directory/authentication/how-to-certificate-based-authentication

https://hmaslowski.com/home/f/azure-ad-certificate-based-authentication-cba-in-public-preview

[Certificate Authorities: Certificate Uploading Failed - Microsoft Q&A](https://learn.microsoft.com/en-us/answers/questions/905204/certificate-authorities-certificate-uploading-fail)
