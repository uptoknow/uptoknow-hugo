+++
title = "OpenShift with CoreOS Dex Openid Provider"
date = 2017-09-15T20:00:08+08:00
draft = false
tags = ["blog", "uptoknow", "openshift"]
categories = ["k8s","openshift","dex"]
+++
### Overview
Dex is an openid provider, opensourced by CoreOS,it can be integrated with k8s, and it also can be integrated with OpenShift,here I will write how to setup the openid provider with openshift.

Dex is not a user management system like ldap,saml,etc. but acts as a portal to other identity providers, it has different connectors that can be used to connect to different user
management system, when you login use user/pass through the portal page of dex, it will generate an ID token for you.Here I will introduce how openshift works with ldap + dex.
In this doc, I assume you have an LDAP server setup that manage your users.

### ID tokens
  ID tokens are an OAuth2.0 extention introduced by OpenID Connect, ID tokens are [JSON Web Tokens](https://jwt.io/), signed by issuer, inject the user claims including username, groups, etc.
You can find the standand claims [here](https://openid.net/specs/openid-connect-core-1_0.html#StandardClaims). So, when you login and get the token, openshift will know what you are by decoding the
token and get the username, groups and so on.

*Note:* Untill now, openshift have not consume the groups decoded from the ID tokens, openshift have it's own `Groups` resource that define which user belong to this group, and we persistent this to etcd.
but we are planning consume the groups info from the ID tokens.

### Setup the dex Openid Provider
Steps:

* Clone the dex code and build the binary

```
git clone https://github.com/coreos/dex
cd dex && make
```
* Generate certs for dex server

```
$openssl genrsa -out ca.key 2048
$openssl req -x509 -new -nodes -key ca.key -subj "/CN=${MASTER_IP}" -days 10000 -out ca.crt
$openssl genrsa -out server.key 2048
$openssl req -new -key server.key -subj "/CN=${MASTER_IP}" -out server.csr
$echo subjectAltName = IP:10.66.146.84 > extfile.cnf
$openssl x509 -req -days 365 -in server.csr -CA ca.crt -CAkey ca.key -CAcreateserial   -out server.crt -extfile extfile.cnf
$openssl x509  -noout -text -in ./server.crt
```
* Update dex config, replace ${dex_server_ip} with your own server ip and ${openshift_master_ip} with your own openshift master ip
and update your ldap server info corrding to your ldap setup.

```
cat examples/config-dev.yaml
# The base path of dex and the external name of the OpenID Connect service.
# This is the canonical URL that all clients MUST use to refer to dex. If a
# path is provided, dex's HTTP service will listen at a non-root URL.
issuer: https://${dex_server_ip}:5554/dex

# The storage configuration determines where dex stores its state. Supported
# options include SQL flavors and Kubernetes third party resources.
#
# See the storage document at Documentation/storage.md for further information.
storage:
 type: sqlite3
 config:
   file: examples/dex.db

# Configuration for the HTTP endpoints.
web:
 #http: 0.0.0.0:5556
 # Uncomment for HTTPS options.
 https: ${dex_server_ip}:5554
 tlsCert: /etc/dex/server.crt
 tlsKey: /etc/dex/server.key

# Uncomment this block to enable the gRPC API. This values MUST be different
# from the HTTP endpoints.
# grpc:
#   addr: 127.0.0.1:5557
#  tlsCert: examples/grpc-client/server.crt
#  tlsKey: examples/grpc-client/server.key
#  tlsClientCA: /etc/dex/client.crt

# Uncomment this block to enable configuration for the expiration time durations.
# expiry:
#   signingKeys: "6h"
#   idTokens: "24h"

# Options for controlling the logger.
logger:
 level: "debug"
 format: "text" # can also be "json"

# Uncomment this block to control which response types dex supports. For example
# the following response types enable the implicit flow for web-only clients.
# Defaults to ["code"], the code flow.
# oauth2:
#   responseTypes: ["code", "token", "id_token"]

# Instead of reading from an external storage, use this list of clients.
#
# If this option isn't chosen clients may be added through the gRPC API.
staticClients:
- id: openshift
 redirectURIs:
         - 'https://${openshift_master_ip}:8443/oauth2callback/dex_openid'
 name: 'openshift'
 secret: ZXhhbXBsZS1hcHAtc2VjcmV0

connectors:
- type: ldap
 id: ldap
 name: LDAP
 config:
   insecureNoSSL: true
   insecureSkipVerify: true
   host: ${ldap_server_ip}
   bindDN: "CN=Administrator,CN=Users,DC=ad-example,DC=com"
   bindPW: ${ldap_server_pass}
   userSearch:
     baseDN: cn=users,dc=ad-example,dc=com
     username: uid
     idAttr: distinguishedName
     emailAttr: mail
     nameAttr: cn
   groupSearch:
     baseDN: ou=groups,dc=ad-example,dc=com
     userAttr: DN
     groupAttr: member
     nameAttr: name
```

* Startup dex with https enabled

```
./dex serve examples/config.yaml
```
* Verify your dex are setup correctly
Dex write a simple to that you can use to perform the login action and gain the id token an print it in the page.
Startup it like this:

```
./bin/example-app --issuer https://${dex_server_ip}:5554/dex --issuer-root-ca /etc/dex/ca.crt --listen http://${dex_server_ip}:5555 --client-id example-app --client-secret ZXhhbXBsZS1hcHAtc2VjcmV0 --redirect-uri http://${dex_server_ip}:5555/callback
```
then you can open the [link](http://${dex_server_ip}:5555), and perform login action to check whether you have correct setup the dex with your own ldap server.

* Update openshift master config

```
 identityProviders:
 - challenge: true
   login: true
   mappingMethod: claim
   name: dex_openid
   provider:
     apiVersion: v1
     kind: OpenIDIdentityProvider
     clientID: openshift
     clientSecret: ZXhhbXBsZS1hcHAtc2VjcmV0
     ca: /root/dex/ca.crt
     claims:
       id:
       - sub
       preferredUsername:
       - name
       name:
       - name
       email:
       - email
     extraScopes:
     - email
     - profile
     urls:
       authorize: https://${dex_server_ip}:5554/dex/auth
       token: https://${dex_server_ip}:5554/dex/token
```
Finaly, restart the openshift master, then when you open your openshift login page, it will dirrect you to the dex login page, any problem, please contact [me](mailto:smarterhaoran@gmail.com).



