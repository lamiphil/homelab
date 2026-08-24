---
project: homelab
type: log
tags: [homelab, log, kubernetes, k3s]
---
# Homelab - Journal de travail

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
