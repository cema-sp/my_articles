# Certificates Authentication

## Authentication methods

* Symmetric (pre-shared keys)
* Asymmetric

### Pre-shared keys

Same key used on both sides for encryption and decryption.

## SSL/TLS

Operates on top of TCP and uses x.509 certificates.  

### x.509

x.509 - standard for a public key infrastructure (PKI) 
and Privilege Management Infrastructure (PMI). 
X.509 specifies, amongst other things, standard formats for public key certificates, 
certificate revocation lists, attribute certificates, 
and a certification path validation algorithm.  

x.509 defines **Certificate Chain** - 
a sequence of certificates signing, starting with root certificate 
and ending with server's one.  

### Authentication

**Both sides have**:

1. Public (shared) key for data encryption on other side
2. Private key for data decryption

**Authentication (digital signature)**:

1. **A** sends message (m.b. long random)
2. **B** encrypts it with Private key and returns to A
3. **A** decrypts it with Public key and matches against initial

## HTTPS

1. TCP Connection
2. Start Handshake
2. ClientHello - Protocol agreement (version, etc.), rand, cipher suite, etc.
3. ServerHello - -//-
3. Server sends its certificate (containing server's Public key)
1. Has the Digital Certificate been trusted by client or issued/signed by a Trusted CA?

  * CA issues (creates) certificate and sign it with its (CA's) Private key 
  * CA's Public key is stored in clients Trusted Certificates store

2. Is the Certificate Expired – checks both the start and end dates
3. Has the Certificate been revoked?

  * Certificate Revocation List (CRL). 
    This is basically a signed list that the CA publishes on a website 
    that can be read by authentication servers. 
    The file is periodically downloaded 
    and stored locally on the authentication server, 
    and when a certificate is being authenticated 
    the server examines the CRL to see if the client’s cert was revoked already. 
  * Online Certificate Status Protocol (OCSP). 
    This is the preferred method for revocation checks in most environments today, 
    because it provides near real-time updates. 
    OCSP allows the authentication server to send a real-time request 
    (like a http web request) to the  service running on the CA 
    or another device and checking the status of the certificate 
    right then & there. 

4. Has the client provided proof of possession 
  (hostname in certificate matches one in browser)?
5. Accept server certificate
6. Client compute symmetric key (AES, DES, BlowFish, etc.), 
  encrypts it with server's Public key and send it to server
7. Client send message with MAC, server matches and responds with MAC too
8. Client matches response
9. Handshake is finished now
6. Exchange data encrypted with symmetric key
7. "close_notify" for connection termination

**MAC (Message authentication code)** - 
short piece of information used to authenticate a message and 
to provide integrity and authenticity assurances on the message.  

![MAC](https://en.wikipedia.org/wiki/File:MAC.svg)

## STARTTLS

**STARTTLS** - 
an extension to plain text communication protocols, 
which offers a way to upgrade a plain text connection 
to an encrypted (TLS or SSL) connection 
instead of using a separate port for encrypted communication.  

## OpenVPN

1. Create Diffie-Hellman key for safe keys exchange (dh.pem)
2. Create CA with certificate (ca.key, ca.crt)
3. Create server key, certificate and certificate signing request 
  (server.key, server.crt, server.csr)
4. Sign server certificate with CA
5. Create client key, certificate and certificate signing request 
  (client.key, client.crt, client.csr)
6. Sign client certificate with CA
7. Provide certificates and keys to server and client

## Links

* [StackExchange](http://security.stackexchange.com/questions/20803/how-does-ssl-tls-work)
* [SSL/TLS - Wiki](https://en.wikipedia.org/wiki/Transport_Layer_Security)
* [x.509 - Wiki](https://en.wikipedia.org/wiki/X.509)
* [STARTTLS - Wiki](https://en.wikipedia.org/wiki/STARTTLS)
* [MAC - Wiki](https://en.wikipedia.org/wiki/Message_authentication_code)
