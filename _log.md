---
project: homelab
type: log
tags: [homelab, log, kubernetes, k3s]
---
# Homelab - Journal de travail

## 2026-11-15 - Simplification des services

**2026-11-15 | 17:59**

Le redémarrage de `minipc` a restauré son état `Ready`.

- Le déploiement Helm de kube-prometheus-stack restait bloqué par un ancien pod node-exporter du Pi dont le sandbox containerd était réservé.
- Supprimé ce pod; le DaemonSet l'a recréé sainement et les six node-exporters sont devenus disponibles.
- La mise à niveau Helm a ensuite réussi.
- Grafana n'est plus attaché à `ubuntuserver` et tourne maintenant sur `minipc`.
- L'API Kubernetes est `Ready`; `archlinux` est de nouveau `NotReady` et sera retiré selon la topologie cible.

**2026-11-15 | 17:49**

J'ai préparé `magos` à exécuter les workloads avant le retrait de `ubuntuserver`.

- Le redémarrage de `magos` a d'abord échoué parce qu'une entrée vide était restée sous `node-taint` dans sa configuration k3s; la correction manuelle a restauré etcd et l'API.
- Retiré la taint `node-role.kubernetes.io/control-plane=true:NoSchedule` du Node `magos` en direct.
- Retiré le `nodeSelector` Grafana qui l'attachait à `ubuntuserver`.
- Flux a appliqué la révision `2dc7682`; les workloads retirés sont en cours de suppression.
- `minipc` (`192.168.42.11`) est hors ligne et inaccessible; son kubelet ne publie plus son état. Il doit revenir `Ready` avant le retrait du control-plane actuel.

**2026-11-15 | 17:31**

Le démarrage de `magos` a restauré le quorum etcd et l'API Kubernetes.

- `magos` est redevenu joignable à `192.168.42.15`; etcd a commencé à écouter sur le port 2380 après environ 25 secondes.
- `ubuntuserver` a été élu leader etcd au terme 7.
- L'API a retrouvé l'état `Ready` environ 15 secondes plus tard.
- `magos`, `minipc`, `datavault`, `pi` et `ubuntuserver` sont `Ready`; `archlinux` est `NotReady`.

**2026-11-15 | 17:27**

J'ai identifié la cause de la réponse `ServiceUnavailable` de l'API Kubernetes.

- Le cluster etcd contient deux voters: `ubuntuserver` (`192.168.42.10`) et `magos` (`192.168.42.15`).
- `magos` est inaccessible depuis le poste local et depuis `ubuntuserver`; la résolution ARP échoue et etcd reçoit `no route to host` sur le port 2380.
- Avec un membre sur deux hors ligne, etcd perd son quorum et ne peut élire de leader.
- k3s reste en état `activating` sur `ubuntuserver` et retourne `503 runtime core not ready`.
- Le disque et la mémoire de `ubuntuserver` ne sont pas saturés.

**2026-11-15 | 17:20**

J'ai retiré de l'état GitOps Pi-hole, Headlamp, draw.io, la pile de streaming, les clients qBittorrent et leur VPN, Loki, Alloy et la prise en charge NVIDIA.

- Retiré aussi qUI, devenu inutile sans qBittorrent, ainsi que le namespace `streaming`.
- Nettoyé les liens Homepage et la source de données Loki dans Grafana.
- Conservé les données locales sous `/srv`; seuls les manifests Kubernetes ont été retirés.
- Mis à jour la documentation avec la cible `magos`, `minipc` et `datavault`.
- Le cluster n'a pas été modifié directement; Flux supprimera les ressources après commit et réconciliation.
- La validation en direct reste bloquée par une réponse `ServiceUnavailable` de l'API Kubernetes.

## 2026-08-24 - Exploration du cluster et ajout de magos

**2026-08-24 | 11:42**

J'ai exploré le repo GitOps et validé son rendu local sans déchiffrer les secrets SOPS ni modifier le cluster pendant l'analyse.

- Confirmé l'architecture k3s + Flux, le réseau MetalLB, les workloads, le stockage et les dépendances externes.
- Exécuté `kubectl kustomize clusters/homelab` avec succès; le rendu contient 105 objets Kubernetes.
- Relevé les principaux risques opérationnels et de sécurité, notamment Headlamp avec `cluster-admin`, l'absence de backups applicatifs, la dépendance aux volumes `hostPath`, les versions flottantes et la documentation qui a dérivé des manifests.

J'ai ensuite préparé `ubuntuserver` pour accueillir un deuxième control-plane en migrant son datastore SQLite vers embedded etcd.

- Sauvegarde pré-migration créée dans `/var/backups/k3s/pre-etcd-20260824-152401`.
- Migration effectuée avec le paramètre temporaire `cluster-init: true`, puis ce paramètre a été retiré après validation et redémarrage.
- Confirmé que `ubuntuserver` est `Ready` avec les rôles `control-plane,etcd,master`.
- Créé les snapshots `post-migration-20260824-152632-ubuntuserver-1787585193` et `pre-magos-20260824-152727-ubuntuserver-1787585250`.

J'ai ajouté `magos` (`192.168.42.15`) comme deuxième serveur k3s et membre etcd.

- Installé la même version que le control-plane existant: `v1.33.5+k3s1`.
- Configuré `magos` avec les rôles `control-plane,etcd,master` et la taint `node-role.kubernetes.io/control-plane=true:NoSchedule`.
- Corrigé un premier fichier `/etc/rancher/k3s/config.yaml` invalide causé par un heredoc mal terminé, puis redémarré k3s avec succès.
- Confirmé la promotion de `magos` de learner à voter etcd, son état `Ready`, la réussite complète de `/readyz?verbose` et la connectivité Flannel.
- Créé le snapshot post-ajout `post-magos-20260824-153723-ubuntuserver-1787585844`.

Points à suivre:

- Faire la rotation du server token puisqu'il a été exposé pendant les opérations.
- Ajouter un troisième membre etcd: un cluster à deux voters ne tolère aucune panne.
- Brancher idéalement `magos` en Ethernet plutôt qu'en Wi-Fi pour assurer la stabilité d'etcd.
- Corriger le version skew: `archlinux` est en `v1.34.3+k3s1` et `datavault` en `v1.35.5+k3s1`, devant le control-plane `v1.33.5+k3s1`.
- Surveiller l'avertissement ponctuel `Failed to allocate directory watch: Too many open files` et les latences etcd si elles persistent.
