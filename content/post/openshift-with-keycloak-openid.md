+++
title = "Openshift Keycloak Openid"
date = 2017-09-14T18:28:15+08:00
draft = false
tags = ["blog", "uptoknow", "openshift", "keycloak"]
categories = ["k8s","openshift","pass","keycloak"]
+++

Keycloak is an powerful tools that can be used as openid provider and SAML auth provider, and here I will record how to integrate openshift with keycloak.

First, download the keycloak from [here](https://mojo.redhat.com/external-link.jspa?url=http%3A%2F%2Fwww.keycloak.org%2Fdownloads.html)

Then, you neec setup the keycloak with https enabled, as openshift ask the openid provider to support https, see the openid spec [here](http://openid.net/specs/openid-connect-core-1_0.html#AuthorizationEndpoint)
   We will setup a standalone cluster and update the binding address, and we need generate the keystore for serve the https
the standalone config file is at: standalone/configuration/standalone.xml.

* You need update the binding address to your host:

```
<interfaces>
  <interface name="management">
    <inet-address value="${jboss.bind.address.management:10.66.140.184}"/>
  </interface>
  <interface name="public">
  <inet-address value="${jboss.bind.address:10.66.140.184}"/>
  </interface>
</interfaces>
```

* Generate the keystore, [ref: HTTPS/SSL Setup | Keycloak Documentation](https://mojo.redhat.com/external-link.jspa?url=https%3A%2F%2Fkeycloak.gitbooks.io%2Fdocumentation%2Fserver_installation%2Ftopics%2Fnetwork%2Fhttps.html)

```
keytool -genkey -alias 10.66.146.84 -keyalg RSA -keystore keycloak.jks -validity 10950
Enter keystore password:
Re-enter new password:
What is your first and last name?
[Unknown]: Haoran Wang
What is the name of your organizational unit?
[Unknown]: RedHat
What is the name of your organization?
[Unknown]: OpenShift
What is the name of your City or Locality?
[Unknown]: BeiJing
What is the name of your State or Province?
[Unknown]: China
What is the two-letter country code for this unit?
[Unknown]: CN
Is CN=Haoran Wang, OU=RedHat, O=OpenShift, L=BeiJing, ST=China, C=CN correct?
[no]: yes
Enter key password for <10.66.146.84>
(RETURN if same as keystore password):
Re-enter new password:
```

* Generate the CSR:

```
keytool -certreq -alias 10.66.146.84 -keystore keycloak.jks > keycloak.careq
cat keycloak.careq
-----BEGIN NEW CERTIFICATE REQUEST-----
MIIC3zCCAccCAQAwajELMAkGA1UEBhMCQ04xDjAMBgNVBAgTBUNoaW5hMRAwDgYD
VQQHEwdCZWlKaW5nMRIwEAYDVQQKEwlPcGVuU2hpZnQxDzANBgNVBAsTBlJlZEhh
dDEUMBIGA1UEAxMLSGFvcmFuIFdhbmcwggEiMA0GCSqGSIb3DQEBAQUAA4IBDwAw
ggEKAoIBAQC/xFVWfd6hbNGCekOcqT5NyffmuPDP80jRNGYyrGgAYCi/on6p8QOJ
qqeagLHLRAUYLrjNLJThGPlWiHlksV8WSRSP5s8Qtrg4JiZSIjMhYKMWAegIb/9d
32BYXZSRO1GaFvaq4zj4XVt8122bX00XX6P2IglrID2nuEyaB+G+3jfMzUEIVXAp
YJRII+eG0zN5Xt57Q+dSzTUYaWgw86EUpmcBGY/b8kp+PRhbFADzKYG/efgSP/Pp
606NkX3COuIL0Z1sI/kp+GwnRLbACwCex5PeuhGArVhxcczUhYBQwDp1QLChJrfM
ukDAxmhM8eG4QlZN+GWoWEGqFDQedEbtAgMBAAGgMDAuBgkqhkiG9w0BCQ4xITAf
MB0GA1UdDgQWBBQO6RWqUO/AH/XwyQs9AC8g8PIpWTANBgkqhkiG9w0BAQsFAAOC
AQEAWc8sQAigKoSp6h/Er/JY/kQGHbHyOuxmspgN4GJZUw8aGs1xejSDRmklx2G5
U2TNTUqEfItrHaMDsOCE8AaX2oNTFbQwf2lQclJW4AJITv0EH0kLhotixqOlK+PK
+xREBNKRWU7gK+6LLzHGJq5KIcx9w1l5JWob71sXYc6rU8BzAfxve+EHjguYB+3k
E3fGd/4iXz8wuvpK5o0ULy7zE1cc/vNthJ68ocqTEWCCN6risBCRc1GEwS2o5nnN
tZvsvf1kXnrY5B/ezB0II+WN9VGqYhGYgvB2lEQtbggcJfo1Ki0ntWUkcti5TSIT
B9B3utbCzTXc5Lv4kgT298Sx4g==
-----END NEW CERTIFICATE REQUEST-----
```

* Generate a private CA and import to keystore:
```
openssl genrsa -out ca.key 2048
openssl req -x509 -new -nodes -key ca.key -subj "/CN=$10.66.146.84" -days 10000 -out ca.crt
keytool -import -keystore keycloak.jks -file ca.crt -alias root
```

* Sign the CSR and import the crt:
```
echo subjectAltName = IP:10.66.146.84 > extfile.cnf
openssl x509 -req -days 365 -in keycloak.careq -CA ca.crt -CAkey ca.key -CAcreateserial   -out server.crt -extfile extfile.cnf
keytool -import -alias 10.66.146.84 -keystore keycloak.jks -file server.crt
```

* Update the config file to use this keystore
```
#In the standalone or domain configuration file, search for the security-realms element and
add(remember update the keystore-password):
<security-realm name="UndertowRealm">
 <server-identities>
 <ssl>
 <keystore path="keycloak.jks" relative-to="jboss.server.config.dir" keystore-password="secret" />
 </ssl>
 </server-identities>
</security-realm>
```
```
#Find the element server name="default-server"
(it?s a child element of subsystem xmlns="urn:jboss:domain:undertow:3.0") and add:
<subsystem xmlns="urn:jboss:domain:undertow:3.0">
  <buffer-cache name="default"/>
    <server name="default-server">
    <https-listener name="https" socket-binding="https" security-realm="UndertowRealm"/>
</subsystem>
```
* Startup the keycloak
```
./bin/standalone.sh
```

Then you can follow the guild to create some openid clients and users, groups
[Server Administration | Keycloak Documentation ](https://mojo.redhat.com/external-link.jspa?url=https%3A%2F%2Fkeycloak.gitbooks.io%2Fdocumentation%2Fserver_admin%2Findex.html)

Next update the openshift master config

```
identityProviders:
- challenge: true
login: true
mappingMethod: claim
name: keycloak_openid
provider:
apiVersion: v1
kind: OpenIDIdentityProvider
clientID: openshift
clientSecret: 8922b2d1-0ab3-48c8-899b-56cb2c15f966
ca: /root/ca.crt
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
authorize: https://${keycloak_server_ip}:8443/auth/realms/openshift/protocol/openid-connect/auth
token: https://${keycloak_server_ip}:8443/auth/realms/openshift/protocol/openid-connect/token
```

Now you can open the openshift webconsole, you will see it will redirect you to the keycloak login page!
