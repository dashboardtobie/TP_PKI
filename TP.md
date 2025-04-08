## 1 - Création d'une autortié de certification

- Génération d'une clé privée pour la CA  
```
┌─[dashboard@parrot]─[~]
└──╼ $mkdir -p ~/dashCA/{certs,crl,newcerts,private}
┌─[dashboard@parrot]─[~]
└──╼ $cd dashCA/
┌─[dashboard@parrot]─[~/dashCA]
└──╼ $openssl genrsa -aes256 -out private/cakey.pem 4096
Enter PEM pass phrase:
Verifying - Enter PEM pass phrase:
```

- Créer un certificat autosigné par le CA
```
┌─[dashboard@parrot]─[~/dashCA]
└──╼ $openssl req -key private/cakey.pem -new -x509 -days 3650 -sha256 -extensions v3_ca -out certs/cacert.pem
```

- Créer les fichiers necessaires à la gestion des certificats  
```
┌─[dashboard@parrot]─[~/dashCA]
└──╼ $touch index.txt
┌─[dashboard@parrot]─[~/dashCA]
└──╼ $echo '1000' > serial
┌─[dashboard@parrot]─[~/dashCA]
└──╼ $echo '1000' > crlnumber
```

- Mettre en place une politque de signature et compléter le fichier `openssl.cnf`

```
┌─[dashboard@parrot]─[~/dashCA]
└──╼ $nano openssl.cnf
```

Extensions à ajouter 

```
[ v3_ca ]
basicConstraints = critical, CA:true
keyUsage = critical, digitalSignature, cRLSign, keyCertSign
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid:always,issuer
```

```
[ usr_cert ]
# Extensions pour certificats serveur
basicConstraints = CA:false
keyUsage = digitalSignature, keyEncipherment
extendedKeyUsage = serverAuth, clientAuth
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid,issuer
```

```
[ crl_ext ]
# Extensions pour CRL
authorityKeyIdentifier = keyid:always
```

### Justification des choix
- Durée de validité:  
CA: 10 ans (3650 jours) - Une durée longue mais raisonnable pour un CA racine  
Certificats émis: 1 an (365 jours) comme configuré dans le fichier openssl.cnf - Conforme aux bonnes pratiques actuelles  

- Taille de clé:  
4096 bits - Offre une sécurité renforcée  
SHA-256 comme algorithme de hachage (default_md = sha256) - Résistant aux collisions connues, contrairement à MD5 ou SHA-1  

- Extensions X.509:  
Restrictions d'usage précises pour limiter les risques  
Contraintes de base pour différencier clairement CA et certificats finaux  
Identifiants de clé pour faciliter la validation du chemin de certification

- Politique stricte:   
Vérification stricte des champs comme countryName, stateOrProvinceName, etc.  
Champs organizationalUnitName et emailAddress optionnels pour plus de flexibilité

## 2 - Génération d'un certificat serveur

- Générer une clé privée et une CSR 
```
┌─[dashboard@parrot]─[~/dashCA]
└──╼ $openssl req -new -newkey rsa:4096 -nodes -keyout server.key -out server.csr
```

- Signer le certificat avec la CA
```
┌─[dashboard@parrot]─[~/dashCA]
└──╼ $openssl ca -config openssl.cnf -in server.csr -out server.crt
Using configuration from openssl.cnf
Enter pass phrase for ./private/cakey.pem:
Check that the request matches the signature
Signature ok
The Subject's Distinguished Name is as follows
countryName           :PRINTABLE:'FR'
stateOrProvinceName   :ASN.1 12:'IDF'
localityName          :ASN.1 12:'Paris'
organizationName      :ASN.1 12:'ACME Corp'
organizationalUnitName:ASN.1 12:'IT'
commonName            :ASN.1 12:'server.acmecorp.local'
Certificate is to be certified until Apr  8 10:55:00 2026 GMT (365 days)
Sign the certificate? [y/n]:y


1 out of 1 certificate requests certified, commit? [y/n]y
Write out database with 1 new entries
Database updated
```

- Verifier le key usage et le extended key usage
```
┌─[dashboard@parrot]─[~/dashCA]
└──╼ $openssl x509 -in server.crt -text -noout | grep -A 1 "Key Usage"
            X509v3 Key Usage: 
                Digital Signature, Key Encipherment
            X509v3 Extended Key Usage: 
                TLS Web Server Authentication, TLS Web Client Authentication
```
