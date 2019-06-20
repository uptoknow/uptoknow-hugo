+++
title = "How we manage certs for openshift dedicated v4"
date = 2019-06-20T14:35:56+08:00
draft = false
tags = ["blog", "uptoknow"]
categories = []
+++

### Overview
In OpenShift Dedicated V4, we use Let's Encrypt to sign the certs for OpenShift controle plane and Ingress router, we designed [certman-operator](https://github.com/openshift/certman-operator) to manage the certs deployed inside the cluster, for more details you can see [Using Kubernetes Operators to Manage Let?s Encrypt SSL/TLS Certificates for Red Hat OpenShift Dedicated](https://blog.openshift.com/using-kubernetes-operators-to-manage-lets-encrypt-ssl-tls-certificates-for-red-hat-openshift-dedicated/), here I how the certs is applied to target cluster.

### How we deploy/update the certs to clusters
We manage v4 dedicated clusters using [hive](https://github.com/openshift/hive), it has a custom resource called `clusterdeployment` which defines everything about install a cluster, also includes the which certificates to use to secure the api.

* when we create a cluster from cloud.openshift.com, a `clusterDeployment` object created, and with configuration settings about which secret is used to store the signed certs.
* certman-operator see new `clusterDeployment` created, then it try to request a signed certs following the ACME protocol.
* after that, the certs is stored inside the secret that defined in clusterDeployment
* hive operator will create a [syncset](https://github.com/openshift/hive/blob/master/docs/syncset.md) that copy that secret to the target created cluster
* openshift v4 have a apiserver operator, and have an CRD called `apiservers.config.openshift.io`, the custom resource named `cluser` have the configuration settigs about which secret to use to apply to the cluster, you can run `oc get apiserver cluster -o yaml` to see the configurations.
* the apiserver operator is responsible for apply the certs to cluster,and when the secret is udpated, it will trigger a redeploy of the certs.
* when the certs are expiring, cetman-opeator will renew the certs, and update the secret in the cluster running hive.
* and the sysnset controller will copy the new certs to the target cluster, then the apiserver operator will apply the change.







