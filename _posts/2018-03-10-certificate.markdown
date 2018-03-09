---
layout: post
title:  "Basic guide to understanding certificates"
date:   2018-03-10 00:57:10 +0800
categories: certificate ssl https
---
This is just a basic guide to understand how certificates used in HTTPS work.

## Background

There are 2 cases to see why you need certificates.

**Case 1**

Sending a secure message from **a** to **b**:

```
@a
stext = a.sign('hello', b.PUB);

@b
b.open(stext, b.PRI);
```

On internet, to check whether a public key is valid, we can ask from Certification Authority (CA).


**Case 2**

To make people believe that a JAR is really from Apache:

```
@Apache
signed_jar = apache.sign(a_jar, apache.PRI)

@MyPC
check(signed_jar, apache.PUB)
```


## Concepts

Some key points:

* Digital Certificate = Public Key (to let everybody open its produced messsages) + Information about the certificate owner.
* A Digital Certificate is signed with the Certificate Authority(CA)'s private key.
* Browsers keep list of CA's public keys, so that the browsers can verify host certificates.
* Two-way SSL: Not only server that display its certificate to client, the client must do the same as well.
* In Java, private keys and trusted CA certificates are stored in a keystore.


## How

To use SSL, a server needs:

* Its private key
* Its digital certificate (which contains matching corresponding public key)
* A trusted CA's certificate

### Generating a pair of public and private key:

Refer to ["Steps to create a CSR (certificate signing request) using keytool 
and get it signed from an external CA (Certificate Authority - Thawte)"]
(https://blogs.oracle.com/blogbypuneeth/entry/steps_to_create_a_csr) for details.

1. Create Digital Cert and Private Key:

   ```sh
   keytool -genkeypair -alias mydomain -keyalg RSA \
		   -keysize 2048 -validity 365 \
		   -keystore identity.jks
   ```
   The first key is for keystore password e.g. "ksp123"
   The second key is for private key e.g. "priv123"

   When asked: "What is your first and last name?", please answer with your host/domain name e.g.: "michsan.web.id" or "*.michsan.web.id".
   
2. Create a CSR 
   to be passed to a Certification Authority (CA) like Thawte, or _Let's encrypt_.

   ```sh
   keytool -certreq -alias mydomain -file mydomain.csr \
           -keystore identity.jks
   ```

   For non-Java applications, each key will be in a separate file. 
   This is how to generate the private key and CSR:
   
   ```sh
   openssl req -new -newkey rsa:2048 -keyout server.key -out mydomain.csr
   ```

   Common name should be your host/domain name e.g.: "michsan.web.id" or "*.michsan.web.id".
   
3. Go to a CA
   For [Let's encrypt](https://letsencrypt.org) can bring the CSR to https://zerossl.com/free-ssl

4. Save each certificate emailed by CA
   * SSL Cert -> signedcert.pem
   * CA Intermediate Cert -> intermediate.pem
   * CA Root Cert -> root.pem

   In "Let's Encrypt", the result should be separated into 2 files
   (in order): signedcert.pem and intermediate.pem

5. Import them in **CORRECT** order

   ```sh
   keytool -importcert -alias inter -file intermediate.pem \
           -keystore identity.jks
   keytool -importcert -alias root -file root.pem \
           -keystore identity.jks

   keytool -importcert -alias mydomain -file signedcert.pem \
           -keystore identity.jks
   ```
   Notice that the alias for the signed cert (the last command) **should be the same** with the one you used previously.

6. Check if they're in already

   ```sh
   keytool -list -v -keystore identity.jks
   ```
   
   Check for the following in the detailed output:
   
   * Alias name: mydomain
   * Entry type: PrivateKeyEntry
   * Certificate chain length: **3**

7. Check if it is valid
   
   ```sh
   java -cp /software/bea/wls/10.30/wlserver_10.3/server/lib/weblogic.jar \
        utils.ValidateCertChain -jks mydomain identity.jks
   ```

8. Create trust keystore
	This is required by Weblogic server.

   ```sh
   keytool -import -file intermediate.pem -alias inter -keystore trust.jks
   keytool -import -file root.pem -alias root -keystore trust.jks
   ```

Note: You **can only get a valid certificate** from CA if your domain is accessible from CA (internet)!


### Setting up Apache with HTTPS

In Apache, it is usually found at `<VirtualHost>` of the `apache2.conf`
* `SSLCertificateFile` = Your server digital certificate (this should be 
    the one that has been signed by CA e.g. signedcert.pem)
* `SSLCertificateKeyFile` = Your server private key (e.g. server.key)
* `SSLCertificateChainFile` = The CA intermediate certificate (e.g. intermediate.pem)


### Other reading

Acting as a CA (usually this is for intranet) - [Setting Up and Authorizing the Internal Certificate Authority](https://blog.secureideas.com/2013/06/ssl-certificates-setting-up-and.html)


### Exposing HTTPS connection in a Java app

In this case, a Spring boot application.

1. Create the key pair

   ```sh
   keytool -genkeypair -alias tomcat -storetype PKCS12 \
           -keyalg RSA -keysize 2048 -validity 3650 \
           -keystore identity.p12
   ```

   In this case, storetype used is `PKCS12` (default is `JKS`)

   You may want to install a valid certificate in this step also (follow my explanation above).

2. Run the app

   ```sh
   java -Dserver.port=8443 \
        -Dserver.ssl.key-store=path/to/identity.p12 \
        -Dserver.ssl.key-store-password=secret \
        -Dserver.ssl.keyStoreType=PKCS12 \
        -Dserver.ssl.keyAlias=tomcat \
        -jar hello-calculator-1.0.0.jar
   ```

3. Ensure you can access via its domain on HTTPS protocol:

   ```sh
   curl -k https://tomcat.local:8443/calc/sum?values=4,7
   ```

   (`-k` means to ignore invalid certificates)

4. Install the certificate to your system as a self-signed certificate,
   **only if** you haven't registered and installed a valid certificate for the domain (step 1).

   Once you install the self-signed certificate (read "instaling-certificate-of-custom-ca.txt"), retry without `-k`


### Configuring a Java client app to use self-signed certificate

This means to configure an app which consumes HTTPS connection with self-signed certificate. So, you don't need this if the certificate is a valid one (from a CA).

1. Import the CA first,

   ```sh
   keytool -import -v -trustcacerts \
           -file tomcat.local.crt -alias tomcat-local \
           -keystore trusts.jks
   ```
   You may want to use `JRE_HOME/lib/security/cacerts` instead of `trusts.jks` as this is the default trust store for Java (with store password is "changeit").
   
   Some other useful commands:
   
   ```sh
   # To delete
   keytool -delete -alias tomcat-local -keystore trusts.jks
   
   # To list:
   keytool -list -v -keystore trusts.jks
   ```
   
2. Run a sample java code (which prints out HTTP GET result)

   ```sh
   java -jar HttpsClient-1.0-all.jar \
	    --type=hc https://tomcat.local:8443/calc/sum?values=4,7
   ```
   
   It only works if you stored the CA in Java trust store. If you didn't, you must run your app this way:

   ```sh
   java -Djavax.net.ssl.trustStore=trusts.jks -Djavax.net.ssl.trustStorePassword=secretStore \
	    -jar HttpsClient-1.0-all.jar \
	    --type=hc https://tomcat.local:8443/calc/sum?values=4,7
   ```
   You can do this way, if you use Apache HTTP Client >= 4.4.1 (to be confirmed)

   
## References:

* http://docs.oracle.com/cd/E13222_01/wls/docs81/secmanage/ssl.html#1167001
* https://www.sslshopper.com/article-most-common-java-keytool-keystore-commands.html
* http://stackoverflow.com/questions/1828775/how-to-handle-invalid-ssl-certificates-with-apache-httpclient
