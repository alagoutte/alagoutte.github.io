---
layout: post
title: "Problème rencontré avec NSX Intelligence et la CNI Cilium"
date: 2025-03-06 12:55:00 -0000
tags: NSX-Intelligence NAPP Cilium Kubernetes VMware
---

Un autre souci rencontré avec NSX Intelligence, et plus précisément avec la CNI Cilium.

En effet, [la documentation de NSX Intelligence](https://techdocs.broadcom.com/us/en/vmware-cis/nsx/nsxt-dc/3-2/deployment-guide/nsx-application-platform-overview/nsx-application-platform-deployment-prerequisites.html) ne précise que les CNI suivantes :
- **Antrea** la CNI développée par VMware avec une base Open vSwitch
- **Calico** la deuxième CNI supportée par VMware Tanzu
- **Flannel** une CNI qui ne supporte pas les Network Policy 

Aucune raison que ça ne fonctionne pas sur Cilium !

## Installation de NSX Intelligence

Le début de l’installation se déroule bien (les pré-checks passent sans souci), mais l’installation de **NSX Application Platform** boucle 🔁 malgré les retries !

 Heureusement, Hubble 🛰 est là !

Pendant l’installation de NSX Platform, j’étais en train d’installer les outils complémentaires à Cilium, dont **Hubble**, qui permet d’avoir une visibilité sur les flux réseaux.

Regardons donc ce qui se passe…

![Hubble Drop](/assets/nsxi-cilium/nsxi_cilium_drop.png)

Aïe, il y a des **DROP** !

### 💡 Première découverte 

Lors de son installation, **NSX Intelligence déploie aussi des Network Policies** ! C’est top... (une grosse cinquantaine quand même !)

```bash
kubectl get networkpolicies.networking.k8s.io -n nsxi-platform

allow-all-egress                                      allow-traffic-to-all=true                                                                                                            32d
allow-all-ingress                                     allow-traffic-from-all=true                                                                                                          32d
authserver                                            app.kubernetes.io/instance=nsxi-platform,app.kubernetes.io/name=authserver                                                           32d
cluster-api                                           app.kubernetes.io/instance=nsxi-platform,app.kubernetes.io/name=cluster-api                                                          32d
cluster-api-client-egress                             cluster-api-client=true                                                                                                              32d
cluster-api-ingress                                   allow-traffic-from-cluster-api=true                                                                                                  32d
common-agent                                          app.kubernetes.io/instance=nsxi-platform,app.kubernetes.io/name=common-agent                                                         32d
contour-egress-networkpolicy                          allow-traffic-to-contour=true                                                                                                        32d
contour-ingress-networkpolicy                         allow-traffic-from-contour=true                                                                                                      32d
debezium-onprem                                       app.kubernetes.io/instance=intelligence,app.kubernetes.io/name=debezium-onprem                                                       31d
debezium-onprem-client-egress                         debezium-onprem-client=true                                                                                                          31d
default-deny-egress                                   <none>                                                                                                                               32d
default-deny-ingress                                  <none>                                                                                                                               32d
dns-networkpolicy                                     allow-traffic-to-dns=true                                                                                                            32d
druid-broker                                          app=druid,component=broker,release=nsxi-platform                                                                                     32d
druid-broker-client-egress                            druid-broker-client=true                                                                                                             32d
druid-coordinator                                     app=druid,component=coordinator,release=nsxi-platform                                                                                32d
druid-coordinator-client-egress                       druid-coordinator-client=true                                                                                                        32d
druid-historical                                      app=druid,component=historical,release=nsxi-platform                                                                                 32d
druid-historical-client-egress                        druid-historical-client=true                                                                                                         32d
druid-middle-manager                                  app=druid,component=middle-manager,release=nsxi-platform                                                                             32d
druid-middle-manager-client-egress                    druid-middleManager-client=true                                                                                                      32d
druid-router                                          app=druid,component=router,release=nsxi-platform                                                                                     32d
druid-router-client-egress                            druid-router-client=true                                                                                                             32d
egress-networkpolicy                                  allow-traffic-to-nsx=true                                                                                                            32d
[....]
```

Par contre, ça bloque.

### 👮 Vérification des Network Policies

Elles me paraissent normales...
```bash
kubectl describe networkpolicies.networking.k8s.io minio -n nsxi-platform
Name:         minio
Namespace:    nsxi-platform
Created on:   2025-01-30 11:59:45 +0100 CET
Labels:       app.kubernetes.io/instance=nsxi-platform
              app.kubernetes.io/managed-by=Helm
              app.kubernetes.io/name=minio
              app.kubernetes.io/version=1.16.0
              helm.sh/chart=minio-0.1.0
              vmware/version=4.2.0-1.0-24389993
Annotations:  meta.helm.sh/release-name: nsxi-platform
              meta.helm.sh/release-namespace: nsxi-platform
Spec:
  PodSelector:     app.kubernetes.io/instance=nsxi-platform,app.kubernetes.io/name=minio
  Allowing ingress traffic:
    To Port: https/TCP
    From:
      PodSelector: minio-client=true
  Allowing egress traffic:
    To Port: https/TCP
    To:
      PodSelector: app.kubernetes.io/component=minio
  Policy Types: Ingress, Egress

```

ou encore au format yaml
```bash
kubectl get networkpolicies.networking.k8s.io minio -n nsxi-platform -o yaml
```

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  annotations:
    meta.helm.sh/release-name: nsxi-platform
    meta.helm.sh/release-namespace: nsxi-platform
  creationTimestamp: "2025-01-30T10:59:45Z"
  generation: 1
  labels:
    app.kubernetes.io/instance: nsxi-platform
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/name: minio
    app.kubernetes.io/version: 1.16.0
    helm.sh/chart: minio-0.1.0
    vmware/version: 4.2.0-1.0-24389993
  name: minio
  namespace: nsxi-platform
  resourceVersion: "9752279"
  uid: 5b26c6d6-5a95-4535-abe5-2d371f57f43d
spec:
  egress:
  - ports:
    - port: https
      protocol: TCP
    to:
    - podSelector:
        matchLabels:
          app.kubernetes.io/component: minio
  ingress:
  - from:
    - podSelector:
        matchLabels:
          minio-client: "true"
    ports:
    - port: https
      protocol: TCP
  podSelector:
    matchLabels:
      app.kubernetes.io/instance: nsxi-platform
      app.kubernetes.io/name: minio
  policyTypes:
  - Ingress
  - Egress
```

## 🚧 Workaround

En attendant, j’applique les deux Network Policies suivantes pour autoriser tout le trafic entrant et sortant.

Policy pour l'ingress
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: alg-allow-all-ingress
  namespace: nsxi-platform
spec:
  ingress:
  - {}
  podSelector: {}
  policyTypes:
  - Ingress

```
Policy pour l'egress

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: alg-allow-all-egress
  namespace: nsxi-platform
spec:
  egress:
  - {}
  podSelector: {}
  policyTypes:
  - Egress
```

et on applique 
```bash
kubectl apply -f alg-allow-all-*.yaml
```

J'aurai pu fusionné les 2 policies mais j'ai voulu verifier si j'avais pas le souci que dans un sens ! mais ce n'est pas le cas, j'ai des flux drop en egress & ingress !

Mon deploiement NSX Application Platform est maintenant fonctionnel !

## 🚨 C'est quoi le probleme au final ?

Revenons à notre problème…

Si on regarde bien les Network Policies, on constate qu’elles utilisent des **named ports**…

```yaml
kubectl get networkpolicies.networking.k8s.io minio -n nsxi-platform -o yaml
[...]
spec:
  egress:
  - ports:
    - port: https
      protocol: TCP
[...]
```

En regardant côté statefulsets du minio, on retrouve bien le port, mais…
```yaml
kubectl get statefulsets.apps minio -o yaml
[...]

        name: minio
        ports:
        - containerPort: 9000
          name: https
          protocol: TCP
        - containerPort: 9001
          name: console
          protocol: TCP

[...]
```
Étrange… le port HTTPS utilise un **port non standard** !

Serait-ce la source du problème ? 🤔

Je vous épargne les heures de debug et de tests…

🕵️‍♂️ et après plusieurs heures (semaines)...

J’ai fini par trouver la cause du problème. C’est une [**limitation connue**](https://github.com/cilium/cilium/issues/30003) dans la CNI **Cilium** :

> **On ne peut pas avoir deux déploiements qui utilisent le même named port avec des ports différents.**

(D’un côté, c’est presque logique…)

## ✅ Vérification avec Cilium

Pour vérifier cela, on peut utiliser la commande suivante (Merci l’IA 🤖) qui permet de lister l’ensemble des named ports avec leur port :

```bash
kubectl get pods -A -o json | jq -r '.items[] | "\(.metadata.namespace) \(.metadata.name)" + ( .spec.containers[]? | select(.ports) | " \(.ports[]?.name):\(.ports[]?.containerPort)" )' 
```

et si je filtre sur le https  

```bash
[...]
nsxi-platform cluster-api-6556d44fb8-t7mct https:8843
nsxi-platform debezium-onprem-9786b4799-wkfmx https:808
nsxi-platform minio-0 https:9000
nsxi-platform minio-1 https:9000
nsxi-platform minio-2 https:9000
nsxi-platform minio-3 https:9000
nsxi-platform monitor-8465b6d5cb-vl6bl https:8443
nsxi-platform pubsub-54578d7444-8fgcs https:8443
[...]
```

On voit que **NSX Intelligence utilise plusieurs fois le même named port** avec des valeurs différentes...

## 🎯 Correction en Cilium 1.18 !

La correction serait prévue dans **Cilium 1.18** !

Donc **attention** si vous utilisez les **named ports** dans vos **Network Policies** avec Cilium.

## 🚦 Conclusion : un NSX Intelligence en mode "open-bar" 🎉

Concernant mon installation de **NSX Intelligence**, pour le moment, elle va rester avec des **Network Policies qui laissent tout ouvert**…

Il serait possible de corriger les **Network Policies existantes**, mais avec un risque de devoir refaire la correction à chaque mise à jour.

Et vu que la CNI **Cilium n’est pas officiellement supportée par NSX Intelligence**, j’ai des doutes que ce soit corrigé par VMware un jour… 🤷‍♂️
