---
title: "[NSX Intelligence] Problème lors du déploiement de NSX Application Platform (NAPP) sous RKE2"
date: 2025-02-11
tags: NSX-Intelligence NAPP RKE2 Kubernetes VMware
---

Lors d’une nouvelle installation de **NSX Intelligence** (ou plutôt **Security Intelligence** maintenant), j’ai rencontré un petit problème inattendu !

## 🚨 Blocage lors des pré-checks de NAPP

Lors du déploiement de **NSX Application Platform (NAPP)**, j’ai été bloqué avant même l’installation à cause des **pré-checks**.

J’obtenais le message d’erreur suivant :

```bash
Check Kubernetes cluster dns domain : Failed (1 Issue)
```
![Erreur Precheck Cluster DNS Domain](/assets/nsxi-kubeadm-config/napp_precheck_dns_error.png)  

et quand on regarde dans les détails :

```bash
Precheck execution failed, Please contact Administrator
```

![Détail de l'erreur....](/assets/nsxi-kubeadm-config/napp_precheck_dns_error_full.png)  

Zut c'est moi l'administrateur...

## 🎯 Hypothèse initiale : Un problème de DNS ❌

Après quelques recherches sur Google… rien de concluant !

## 🔍 Investigation dans les logs

Direction les logs de **NAPP** sur le **NSX Manager** :

L'ensemble des logs concernant l'installation de NAPP est situé `/var/log/proton/napps.log`

On investige dans le fichier....

```bash
    cat /var/log/proton/napps.log 

    [...]
    2024-09-30 13:41:49,906 INFO nsx_kubernetes_lib.vmware.kubernetes.service.kubectl.kubectl_117_service[331]:get_cluster_dns_domain Getting cluster dns domain
    2024-09-30 13:41:49,906 INFO nsx_kubernetes_lib.vmware.kubernetes.service.kubectl.kubectl_117_service[350]:execute Executing command kubectl get configmap kubeadm-config -n kube-system -o yaml --kubeconfig=/config/vmware/napps/.kube/config
    2024-09-30 13:41:49,906 INFO nsx_kubernetes_lib.vmware.kubernetes.common.utility[23]:execute ['kubectl', 'get', 'configmap', 'kubeadm-config', '-n', 'kube-system', '-o', 'yaml', '--kubeconfig=/config/vmware/napps/.kube/config']
    2024-09-30 13:41:49,956 ERROR nsx_kubernetes_lib.vmware.kubernetes.common.utility[57]:execute Error executing command 'kubectl get configmap kubeadm-config -n kube-system -o yaml --kubeconfig=/config/vmware/napps/.kube/config',  'Error from server (NotFound): configmaps "kubeadm-config" not found\n'
    2024-09-30 13:41:49,956 ERROR __main__[76]:main Error executing function get_cluster_dns_domain. Error message: Error from server (NotFound): configmaps "kubeadm-config" not found\n
    [...]
```

En creusant un peu, j’ai trouvé que le **pré-check vérifie que le domaine interne est bien `cluster.local`** (c’est sous-entendu dans la [doc…](https://techdocs.broadcom.com/us/en/vmware-security-load-balancing/vdefend/vmware-nsx-application-platform/4-2/deploying-and-managing-the-nsx-application-platform/deploying-the-nsx-application-platform/configuring-your-environment-for-manual-deployment/manual-deployment-requirements.html) 🙃).

D’après les logs, il tente de récupérer la **ConfigMap `kubeadm-config`** dans le namespace kube-system :
```bash
    kubectl get configmap kubeadm-config -n kube-system
```
💡 **Problème** : sur mon cluster… **aucune ConfigMap `kubeadm-config`** ❌

## 🔎 Analyse du problème

Dans cette installation, un **cluster Kubernetes sous Rancher RKE2 (1.29…)** avait été déployé.

👉 **Particularité** : Rancher **n’utilise pas `kubeadm`** pour l’installation de ses clusters, mais son propre mécanisme !

Le client disposait déjà d’autres clusters sous Rancher, et comme souvent, la **compatibilité avec NAPP est limitée**… et la [doc de VMware](https://techdocs.broadcom.com/us/en/vmware-security-load-balancing/vdefend/vmware-nsx-application-platform/4-2/deploying-and-managing-the-nsx-application-platform/deployment-requirements-for-napp/nsx-application-platform-deployment-prerequisites.html) devient de moins en moins précise sur ce sujet.

![La Documentation VMware sur les clusters K8S supportés](/assets/nsxi-kubeadm-config/napp_broadcom_doc_k8s.png)  

## ✅ Solution trouvée

1. **Se connecter à un autre cluster Kubernetes** (non RKE2) où `kubeadm` est utilisé.  
2. **Récupérer uniquement la partie concernée de la ConfigMap `kubeadm-config`**.

Sur un cluster `kubeadm` fonctionnel, exécuter :

```bash
    kubectl get configmap kubeadm-config -n kube-system -o yaml > kubeadm-config.yaml
```

3. **Éditer le fichier pour ne garder que la section `clusterConfiguration`** :

```yaml
    apiVersion: v1
    kind: ConfigMap
    metadata:
      name: kubeadm-config
      namespace: kube-system
    data:
      ClusterConfiguration: |
        apiVersion: kubeadm.k8s.io/v1beta3
        kind: ClusterConfiguration
        clusterName: my-cluster
        networking:
          dnsDomain: cluster.local
```
4. **L’installer sur mon cluster Rancher RKE2** :
```bash
    kubectl apply -f kubeadm-config.yaml
```
5. **Relancer les pré-checks**… et cette fois, **tout passe !** 🎉

## 🔥 Conclusion

Si vous utilisez Rancher RKE2 avec **NSX Application Platform (NAPP)**, pensez à recréer manuellement la **ConfigMap `kubeadm-config`** pour éviter ce souci !

🔧 **Bonus** : Il serait intéressant que VMware améliore la compatibilité avec Rancher, ou au moins documente mieux cette contrainte…

💬 **Et vous, avez-vous rencontré ce genre de souci ?** 😊
