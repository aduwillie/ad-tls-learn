# Transport Layer Security

We use SSL (Secure Sockets Layer) encryption to protect our web facing resources. This is not based on the SSL protocol since it has been deprecated since version 3. The protocol that provides the backbone is called TLS (Transport Layer Security). The protocol runs between TCP and end user protocols like HTTP for web traffic. It can also run on iMap or POP3 for email. Depending on the service, TLS can identify itself by working through a unique port. Check the table below:

| Protocol | Unencrypted Port | Encrypted Port |
|----------|------------------|----------------|
| HTTP     | 80               | 443            |
| LDAP     | 389              | 636            |
| TELNET   | 23               | 992            |
| IMAP     | 143              | 993            |
| IRC      | 6667             | 994            |
| POP3     | 110              | 995            |

## Certificate

A certificate is used to test the authenticity of a particulat encryption key and that it is indeed a match with the key protecting the data you are trying to access. A certificate can the a combination of resorce-specific metadata and an encryption key.

### Cryptography

- Symmetric 
- Assymetric

It is possible to manage these keys in-house. In some cases, the authenticity of these public keys is entrusted to a mutually trusted third party aka `certificate authority`. This PKI (Public Key Infrastructure) is responsible for securing data. To protect against network data noise, a relatively small message digest, a hash, is often generated to send alongside data that allows a recipient to compare the data he received to what was initially sent. Various flavours lke MD5 or SHA1 will often be incorporated into encryption operations. 

```
md5sum dummy.txt
```

### A Cypher Suite

A group of configuration choices that can serve to define a wide range of encryption and transfer behaviour. This includes the data compression, key exchange algorithm eg RSA, the bulk cipher eg. 3DES_EDE_CBC, and a hash eg. SHA.

```
TLS_ECDHE_RSA_WITH_3DES_EDE_CBC_SHA
```

Certificates (eg. X.509) can be encoded in a number of formats:

- `.DER` Distinguished Encoding Rules
- `.PEM` Privacy-enhanced Electronic Mail (base64 encoding)

The encoding can be further divided into various PKCS (Public Key Cryptography Standards).

## SSL Lifecycle

We begin with OpenSSL program in our web server that will create the Public Key Infrastructure (PKI). What this means is that OpenSSL will generate a private and public key pair. The private key is secured on the web server whereas the public key is sent to the certificate authority (CA). The keys would be used to create a properly formatted Certificate Signing Request (CSR) package. This would contain the public key along with the profile information about the certificate owner (metadata describing the website itself). 

Generating a certificate key combination using our CA's own certificate allows us to establish a strong chain of trust where the certificate that the CA will send us will be used to tell a visiting browser that this site is part of a safely encrypted infrastructure. The CA's certificate (Intermediate Certificate, IC), derives its authority from the fact that it was signed by the root certificate of the authority, which is actually signed by itself. 

The root certificate itself is preinstalled in browsers by all major browser providers who thereby, express their ongoing trust in the authencity of those CAs. So when a browser asks for your sites public key, it uses it to confirm that our cert is compatible with the copy of the root cert that's in the browser.

You should prefer certificates from trusted CAs instead of self generated ones.

## OpenSSL CLI

### Installation

```
sudo apt-get install openssl
sudo apt-cache search libssl // get the latest library
sudo apt-get install libssl1.0.0 // derive from the preferred library
```

### Commands

```
openssl -h
```

#### Generate a CSR

```
openssl -req \              
  -nodes \                  // will not encrypt the output key
  -days 10 \                // valid time for cert
  -newkey rsa:2048 \        // using RSA key of 20148 bits
  - keyout keyfile.pem \    // name for our .pem file
  -out certfile.pem         // name of the output file we are after
```

The above command outputs 2 files.
- `keyfile.pem` with the key
- `cerfile.pem` containing the certificate request

To verify the request, run the commands below:

```
openssl req \
  -in certfile.pem
  -noout                    // suppresses the certificate from being printed on screen
  -verify                   // verify our certificate request
  -key keyfile.pem
```

To review the metadata passed with the request:

```
openssl req \
  -in certfile.pem
  -noout
  -text                     // output the values passed to the request
```

If there's a mistake, you can just delete and create them again.

#### Generating self signed requests

Creating a private CA could be helpful when creating apps used internally. There might be some complains on initial access but dealing with it isn't much of a hassle.

To create a directory to host CA related files

```
mkdir ~/my-ca
cd ~/my-ca
mkdir signedcerts           // all the certificates that our CA will sign
mkdir private               // where we keep our private key
```

OpenSSL will keep track of our certificates and keys using a very simple database made up of files called `serial` and `index.text`. OpenSSL will manage creating a trail of named backup versions. 

```
echo '01' > serial
touch index.text
```

Create a configuration file from which OpenSSL can read our environment profile values. We would call it `caconfig.cnf`. Contents could look like:

```
# Sample caconfig.cnf file.
#
# Default configuration to use when one is not provided on the command line.
#
[ ca ]
default_ca      = local_ca
#
#
# Default location of directories and files needed to generate certificates.
#
[ local_ca ]
dir             = /home/ubuntu/my-ca
certificate     = $dir/cacert.pem
database        = $dir/index.txt
new_certs_dir   = $dir/signedcerts
private_key     = $dir/private/cakey.pem
serial          = $dir/serial
#       
#
# Default expiration and encryption policies for certificates.
#
default_crl_days        = 365
default_days            = 1825
default_md              = sha1
#       
policy          = local_ca_policy
x509_extensions = local_ca_extensions
#
#
# Copy extensions specified in the certificate request
#
copy_extensions = copy
#       
#
# Default policy to use when generating server certificates.  The following
# fields must be defined in the server certificate.
#
[ local_ca_policy ]
commonName              = supplied
stateOrProvinceName     = supplied
countryName             = supplied
emailAddress            = supplied
organizationName        = supplied
organizationalUnitName  = supplied
#       
#
# x509 extensions to use when generating server certificates.
#
[ local_ca_extensions ]
basicConstraints        = CA:false
#       
#
# The default root certificate generation policy.
#
[ req ]
default_bits    = 2048
default_keyfile = /home/ubuntu/my-ca/private/cakey.pem
default_md      = sha1
#       
prompt                  = no
distinguished_name      = root_ca_distinguished_name
x509_extensions         = root_ca_extensions
#
#
# Root Certificate Authority distinguished name.  Change these fields to match
# your local environment!
#
[ root_ca_distinguished_name ]
commonName              = stuff.com
stateOrProvinceName     = Ontario
countryName             = CA
emailAddress            = info@bootstrap-it.com
organizationName        = Bootstrap IT
organizationalUnitName  = Tech
#       
[ root_ca_extensions ]
basicConstraints        = CA:true
```

Next step is to generate our CA root certificate. This would be passed to client browsers or devices. Lets start by exporting the location of our `caconfig.cnf`

```
export OPENSSL_CONF=~/my-ca/caconfig.cnf
```

To generate a new certificate and key:

```
openssl req \
  -x509                     // create an actual certificate instead of a request
  -newkey rsa:2048
  -out cacert.pem           // certificate name
  -outform PEM              // use the PEM formate
  -days 1825
```

NB: Don't forget to enter a passphrase

The command generates a public certifate called `cacert.pem` and a private key of `cakey.pem` in the private folder.

Next is to generate a self-signed certificate to go with the root certificate. There is a need to create a separate configuration in the root directory, `my-ca`. We call it `testserver.cnf`. Content could be as follows:

```
#
# testserver.cnf
#

[ req ]
prompt                  = no
distinguished_name      = server_distinguished_name
req_extensions          = v3_req

[ server_distinguished_name ]
commonName              = stuff.com
stateOrProvinceName     = Ontario
countryName             = CA
emailAddress            = info@bootstrap-it.com
organizationName        = Bootstrap IT
organizationalUnitName  = IT

[ v3_req ]
basicConstraints        = CA:FALSE
keyUsage                = nonRepudiation, digitalSignature, keyEncipherment
subjectAltName          = @alt_names

[ alt_names ]
DNS.0                   = www.stuff.com
DNS.1                   = 10.0.3.63
```

```
export OPENSSL_CONF=~/my-ca/testserver.cnf
```

To use the new file as a reference:

```
openssl req \
  -newkey rsa:2048
  -keyout basekey.pem           // certificate name
  -keyform PEM                  // use the PEM formate
  -out basereq.pem
  -ourform PEM
```

Need to poing the environment variable to the main config file

```
export OPENSSL_CONF=~/my-ca/caconfig.cnf
```

To sign the certificate:

```
openssl ca
  -in basereq.pem
  -out server_crt.pem
```

Whew!! Now we are done.
