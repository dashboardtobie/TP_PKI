# NOM : Tobie Datchoua

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

- Vérifier le key usage et le extended key usage
```
┌─[dashboard@parrot]─[~/dashCA]
└──╼ $openssl x509 -in server.crt -text -noout | grep -A 1 "Key Usage"
            X509v3 Key Usage: 
                Digital Signature, Key Encipherment
            X509v3 Extended Key Usage: 
                TLS Web Server Authentication, TLS Web Client Authentication
```

## 3 - Mise en place d'une CRL

- Révocation d'un certificat
```
┌─[✗]─[dashboard@parrot]─[~/dashCA]
└──╼ $openssl ca -config openssl.cnf -revoke newcerts/1000.pem
Using configuration from openssl.cnf
Enter pass phrase for ./private/cakey.pem:
Revoking Certificate 1000.
Database updated
```

- Génération d'une CRL
```
┌─[dashboard@parrot]─[~/dashCA]
└──╼ $openssl ca -config openssl.cnf -gencrl -out crl.pem
Using configuration from openssl.cnf
Enter pass phrase for ./private/cakey.pem:
```

- Vérifier que le certificat révoqué est bien présent dans la CRL

```
┌─[dashboard@parrot]─[~/dashCA]
└──╼ $openssl crl -in crl.pem -text -noout
Certificate Revocation List (CRL):
        Version 2 (0x1)
        Signature Algorithm: sha256WithRSAEncryption
        Issuer: C = FR, ST = IDF, L = Paris, O = ACME Corp, OU = ACME Corp, CN = ACME Corp Root CA, emailAddress = bdx@bdx.com
        Last Update: Apr  8 12:22:20 2025 GMT
        Next Update: May  8 12:22:20 2025 GMT
        CRL extensions:
            X509v3 CRL Number: 
                4096
Revoked Certificates:
    Serial Number: 1000
        Revocation Date: Apr  8 12:17:51 2025 GMT
    Signature Algorithm: sha256WithRSAEncryption
    Signature Value:
        2e:2c:69:fe:b2:58:4c:44:e6:52:05:f1:03:df:60:45:3f:e2:
        d8:89:20:5b:ba:08:b3:bf:c7:4f:d1:ea:65:98:1b:b1:4e:75:
        4d:27:3f:db:c9:1d:69:61:52:a3:36:e2:29:4f:bc:32:03:e2:
        ac:10:6c:7e:04:57:15:a0:6b:5f:a3:27:de:ac:2d:a9:04:1f:
        c3:0f:8f:60:92:45:c6:f6:8c:5b:d4:fb:0a:d5:71:3a:44:63:
        33:69:8d:96:17:9f:17:d6:4e:e4:cb:a7:bf:98:6d:98:ec:f1:
        54:f3:46:95:54:98:e0:b7:30:8a:15:30:eb:f6:d1:c4:0e:3f:
        94:55:46:84:0d:ff:0d:d5:0c:c5:94:c9:51:bf:9d:d6:b7:ed:
        93:d5:81:22:e4:9b:8a:3f:47:3e:4f:f8:e8:e3:af:0d:d4:b0:
        f2:cf:ec:3f:85:c2:e3:ee:4e:71:ee:9e:97:13:3f:81:dc:14:
        e5:6d:f8:1c:26:74:81:d1:b4:b8:a5:7f:b5:db:2b:67:3a:77:
        fe:6d:8d:6b:70:29:9f:36:0b:6e:99:3f:3b:1d:80:43:2f:ce:
        19:cb:72:3d:fd:cc:e9:09:d9:aa:2e:ee:2c:fc:3d:2d:8c:df:
        de:d3:58:d9:4d:c9:10:c1:61:cc:a1:b3:33:2c:38:b5:8f:85:
        ea:14:8a:6c:7a:94:ae:e3:ea:53:9c:ec:52:67:f8:2f:c3:8e:
        0a:87:86:dc:35:53:4b:37:fe:34:cf:ff:f3:a3:b2:9e:fc:5c:
        30:85:e0:60:9a:b8:8f:73:c5:fb:75:e7:5e:cc:6e:f6:9d:7f:
        7d:82:13:fd:3a:07:a6:fb:f1:4b:11:2c:de:06:09:07:9c:9d:
        1f:1b:b5:cd:13:04:ed:e5:ca:f2:bf:8f:52:2a:34:84:50:9b:
        79:a7:47:09:bf:72:96:67:52:95:99:2b:da:68:60:74:f9:53:
        0a:f0:2e:a6:e6:e1:e5:13:9a:a4:19:7d:6a:af:85:d1:1b:41:
        ed:3d:23:8c:39:5d:76:f6:1e:ac:ad:1f:76:f3:b0:f5:64:57:
        38:13:6c:31:44:55:ac:a2:e7:0a:6e:0f:90:23:8a:cd:be:5c:
        d3:2f:a1:53:8f:c9:d9:c6:e4:96:8c:d4:24:f3:a6:dd:93:65:
        d4:2c:29:ee:b1:99:c3:54:81:8d:26:ec:7a:9f:35:b6:07:7e:
        8a:9c:1e:7c:54:4f:de:00:a0:0a:1d:3b:3a:bb:ef:4a:3b:64:
        d3:a3:51:e1:09:7e:29:0d:be:80:8a:12:43:a6:38:a8:82:ea:
        dd:6a:78:b6:76:1b:34:7f:d0:ce:77:0f:59:83:4d:38:c3:5f:
        50:f8:3e:68:d9:c3:f0:35
```

- Décrire comment serait distribuée cette CRL dans un environnement réel

Dans un environnement réel, la distribution de la CRL pourrait se faire de plusieurs façons :  

a - Point de distribution HTTP :  

Publiez la CRL sur un serveur web accessible  

Configurez l'extension "CRL Distribution Points" dans vos certificats pour pointer vers cette URL  

b - Point de distribution LDAP :  

Stockez la CRL dans un annuaire LDAP  

Les clients peuvent interroger cet annuaire pour obtenir la CRL à jour  

c - Utilisation d'OCSP (Online Certificate Status Protocol) :  

Alternative plus efficace aux CRLs  

Permet aux clients de vérifier le statut d'un certificat spécifique sans télécharger la CRL complète  

Nécessite la mise en place d'un répondeur OCSP  

d- Automatisation du renouvellement :  

Configurez un script cron pour régénérer automatiquement la CRL à intervalles réguliers (par exemple, toutes les 24 heures)  

Le paramètre default_crl_days = 30 dans votre fichier openssl.cnf définit la durée de validité de la CRL  

## 4 - Déploiement et test de service OCSP

- Mettre en place un service OCSP local

```
┌─[✗]─[dashboard@parrot]─[~/dashCA]
└──╼ $openssl genrsa -out ocsp_responder.key 4096
┌─[dashboard@parrot]─[~/dashCA]
└──╼ $openssl req -new -key ocsp_responder.key -out ocsp_responder.csr
┌─[dashboard@parrot]─[~/dashCA]
└──╼ $openssl ca -config openssl.cnf -extensions v3_ca -in ocsp_responder.csr -out ocsp_responder.crt -days 365
┌─[✗]─[dashboard@parrot]─[~/dashCA]
└──╼ $openssl ocsp -port 8080 -index index.txt -CA certs/cacert.pem -rsigner ocsp_responder.crt -rkey ocsp_responder.key -text
ACCEPT 0.0.0.0:8080 PID=78901
ocsp: waiting for OCSP client connections...
```

- Vérifier l'état d'un certificat
```
┌─[dashboard@parrot]─[~/dashCA]
└──╼ $openssl ocsp -CAfile certs/cacert.pem -issuer certs/cacert.pem -cert server.crt -url http://localhost:8080
server.crt: revoked
	This Update: Apr  8 12:58:58 2025 GMT
	Revocation Time: Apr  8 12:17:51 2025 GMT
```


```
┌─[dashboard@parrot]─[~/dashCA]
└──╼ nano /etc/nginx/sites-available/default
server {
    #listen 80 default_server;
    #listen [::]:80 default_server;

    # SSL configuration
    #
    listen 443 ssl default_server;
    listen [::]:443 ssl default_server;
    ssl_certificate /home/dashboard/dashCA/server.crt;
    ssl_certificate_key /home/dashboard/dashCA/server.key;
```

- Comparaison OCSP et CRL

    | Caractéristique            | CRL                                             | OCSP                                        |
    | -------------------------- | ----------------------------------------------- | ------------------------------------------- |
    | Méthode de vérification    | Téléchargement périodique de listes complètes   | Requêtes en temps réel à un serveur OCSP    |
    | Fraîcheur des info         | Dépend de la fréquence de mise à jour de la CRL | Information en temps réel                   |
    | Confidentialité            | Pas de confidentialité (liste publique)         | Peut être configuré pour la confidentialité |
    | Complexité                 | Simple à mettre en place                        | Plus complexe (serveur OCSP)                |


## 5 - Lancer un serveur HTTPS

- Utiliser un certificat pour lancer un serveur
```
┌─[dashboard@parrot]─[~/dashCA/certs]
└──╼ $sudo nano /etc/nginx/sites-enabled/default
server {

    # SSL configuration
    listen 443 ssl;
    listen [::]:443 ssl;
    ssl_certificate /home/dashboard/dashCA/server.crt;
    ssl_certificate_key /home/dashboard/dashCA/server.key;
```

- Tester son accessibilité
```
┌─[✗]─[dashboard@parrot]─[~/dashCA]
└──╼ $curl -v https://server.acmecorp.local --cacert certs/cacert.pem --resolve 'server.acmecorp.local:443:127.0.0.1'
* Added server.acmecorp.local:443:127.0.0.1 to DNS cache
* Hostname server.acmecorp.local was found in DNS cache
*   Trying 127.0.0.1:443...
* GnuTLS ciphers: NORMAL:-ARCFOUR-128:-CTYPE-ALL:+CTYPE-X509:-VERS-SSL3.0
* found 1 certificates in certs/cacert.pem
* found 423 certificates in /etc/ssl/certs
* SSL connection using TLS1.3 / ECDHE_RSA_AES_256_GCM_SHA384
*   server certificate verification OK
*   server certificate status verification SKIPPED
*   common name: server.acmecorp.local (matched)
*   server certificate expiration date OK
*   server certificate activation date OK
*   certificate public key: RSA
*   certificate version: #3
*   subject: C=FR,ST=IDF,O=ACME Corp,OU=IT,CN=server.acmecorp.local
*   start date: Tue, 08 Apr 2025 13:13:06 GMT
*   expire date: Wed, 08 Apr 2026 13:13:06 GMT
*   issuer: C=FR,ST=IDF,L=Paris,O=ACME Corp,OU=ACME Corp,CN=ACME Corp Root CA,EMAIL=bdx@bdx.com
* ALPN: server accepted http/1.1
* Connected to server.acmecorp.local (127.0.0.1) port 443
* using HTTP/1.x
> GET / HTTP/1.1
> Host: server.acmecorp.local
> User-Agent: curl/8.10.1
> Accept: */*
> 
```

## Questions 
1 - Pourquoi le navigateur indique-t-il une alerte de sécurité ?

Un navigateur affiche une alerte de sécurité lorsqu'il ne peut pas vérifier la validité du certificat d'un site web.  

2 - Quelle est la durée de validité du certificat serveur ? Où cette info est-elle visible ?

La durée par défaut est de 365 jours, configurée via le paramètre default_days = 365 dans le fichier openssl.cnf.  

3 - Quelles informations contient le certificat ?  

Le certificat contient les informations suivantes :  

* Nom de l'entité (sujet) à qui le certificat a été délivré.  

* Clé publique de l'entité.  

* Informations sur l'autorité de certification (CA) qui a émis le certificat.  

* Période de validité du certificat.  

* Numéro de série du certificat.  

* Algorithme de signature utilisé.  

* Extensions X.509.

4 - Quel est le rôle d’une CRL ? Quelles limites présente-t-elle ?  

Une liste de révocation de certificats (CRL) est une liste de certificats qui ont été révoqués par l'autorité de certification (CA) avant leur date d'expiration. Les clients (par exemple, les navigateurs web) peuvent télécharger la CRL pour vérifier si un certificat est toujours valide.

Limites d'une CRL :  

* Taille : La taille de la CRL peut augmenter avec le temps, ce qui peut entraîner des problèmes de bande passante et de performance.  

* Latence : Les clients doivent télécharger la CRL régulièrement pour obtenir les informations de révocation les plus récentes, ce qui peut introduire une latence.  

* Disponibilité : La CRL doit être disponible en permanence pour que les clients puissent vérifier les certificats.  

5 - Quelle différence avec OCSP ? Pourquoi est-il plus utilisé ?   

OCSP (Online Certificate Status Protocol) est une alternative à la CRL. Au lieu de télécharger une liste complète de certificats révoqués, les clients envoient une requête OCSP à un répondeur OCSP pour vérifier l'état d'un certificat spécifique.  

OCSP est plus utilisé car :  

* Plus efficace : OCSP est plus efficace en termes de bande passante et de performance car il ne nécessite pas le téléchargement d'une liste complète de certificats révoqués.  

* Temps réel : OCSP offre une vérification en temps réel de l'état des certificats.  

6 - Comment tester si un certificat est bien révoqué (via CRL et via OCSP) ?

* Via CRL : Générer une CRL après avoir révoqué le certificat. Ensuite, utiliser la commande openssl crl pour afficher le contenu de la CRL et vérifier que le numéro de série du certificat révoqué y figure.  

* Via OCSP : Envoyer une requête OCSP au répondeur OCSP pour vérifier l'état du certificat. Vérifier que la réponse OCSP indique que le certificat a été révoqué.  

7 - En quoi la configuration d’un fichier openssl.cnf est-elle cruciale dans un environnement professionnel ? expliquez chaque partie du fichier.

Le fichier openssl.cnf est crucial car il définit la politique de sécurité de votre infrastructure PKI. Il contrôle les aspects suivants :  

* [ca] : Section principale définissant les paramètres de l'autorité de certification.   

* [CA_default] : Définit les chemins vers les fichiers et répertoires utilisés par la CA (certificats, clés privées, CRL, etc.).  

* [policy_strict] : Définit la politique de nommage des certificats.  

* [req_distinguished_name] : Définit le nom distinctif (DN) par défaut pour les CSR.  

* [v3_ca],[usr_cert], [crl_ext] : Définissent les extensions X.509 à utiliser pour les certificats d'autorité de certification (CA), les certificats utilisateur et les CRL, respectivement. Ces sections permettent de définir les usages autorisés pour les certificats et de renforcer la sécurité de l'infrastructure PKI.  

