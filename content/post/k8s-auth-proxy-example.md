+++
title = "K8s Auth Proxy Example"
date = 2017-08-12T11:23:48+08:00
draft = true
tags = ["blog", "uptoknow"]
categories = ["k8s","pass"]
+++

K8S support different kind of auth type, one of it’s auth type is [Authenticating Proxy](https://kubernetes.io/docs/admin/authentication/#authenticating-proxy), this allow user to use it’s auth provider to do the authentication, after pass the auth, sent the use related info(username, group, extra info) using http request headers to k8s api server, the headers can be defined using:

```
--requestheader-username-headers
--requestheader-group-headers
--requestheader-extra-headers-prefix
```

New people to this area are not very familiar how to setup a auth proxy and integrated with k8s, so I wish this blog can help you guys. Here, I will using nginx as the proxy, using htpasswd to do the auth, and if passed the auth, sent the using info to the request header to apiserver.Steps:

1.Setup a local cluster using hack/local-cluster-up.sh
This will startup a cluster with the request-header related options setup, and generate the request-header-ca.crt for validate the client key/crt from the proxy server, also the auth client key/crt are genarated, see the files here:

```
request-header-ca.crt
request-header-ca.key
client-auth-proxy.key
client-auth-proxy.crt
```

2.Setup the nginx

```
 location / {
           auth_basic "basic auth";
          auth_basic_user_file /etc/nginx/conf.d/nginx.htpasswd;     #using htpassword
          proxy_pass     https://localhost:6443;         #if auth succeed, redirect to apiserver
          proxy_set_header X-Remote-User $remote_user;     #set the username using this header
          proxy_set_header X-Remote-Group system:masters;       # htpassword didn’t have the group info, and by default, authenticator will map the user to system:basic user group which have no priviledge.

          proxy_ssl_certificate     /var/run/kubernetes/client-auth-proxy.crt;   #This is the client crt
          proxy_ssl_certificate_key     /var/run/kubernetes/client-auth-proxy.key;  # This is the server crt
          proxy_ssl_trusted_certificate /var/run/kubernetes/server-ca.crt;   # This is the ca that signed the crt apiserver used
          proxy_ssl_verify       on;
          proxy_ssl_session_reuse on;
          proxy_ssl_protocols           TLSv1 TLSv1.1 TLSv1.2;
        }

```

3.Create a user using htpasswd

```
 htpasswd -c -b /etc/nginx/conf.d/nginx.htpasswd admin admin
```

4.Open the browser and open http://localhost , input admin/admin, this will allow you access the apiserver now.

---

If you need kubectl using the proxy, we need enable the https of the proxy, cause currently, if you are using basic auth, you must enable the https, or the kubectl will not wrap the auth header for you.

* Create the signing ca, server crt/key follow the guide:
https://kubernetes.io/docs/admin/authentication/#openssl

* Update the nginx config:

```
 listen       443;
        ssl  on;
        ssl_certificate /var/run/kubernetes/nginx/server.crt;
        ssl_certificate_key  /var/run/kubernetes/nginx/server.key;

```

* Create a new admin.kubeconfig

```
apiVersion: v1
clusters:
- cluster:
    certificate-authority: /var/run/kubernetes/nginx/ca.crt
    server: https://localhost
  name: local-up-cluster
contexts:
- context:
    cluster: local-up-cluster
    user: local-up-cluster
  name: local-up-cluster
current-context: local-up-cluster
kind: Config
preferences: {}
users:
- name: local-up-cluster
  user:
    password: admin
    username: admin
```

And now you should be using nginx as the auth proxy against the htpasswd backend to do the authentication with k8s, we hard code the group info for the user, instead we can use the nginx ldap plugin with openldap to get the groups info and set the group http header or write your own auth server to do this.
