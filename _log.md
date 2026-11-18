---
project: homelab
type: log
tags: [homelab, log, kubernetes, k3s]
---
# Homelab - Journal de travail

## 2026-11-17 - Tableau de bord Grafana des habitudes Praxis

J'ai ajouté le tableau de bord `Praxis — Habitudes` (UID `praxis-habits`), conçu après une séance de questions.

- Source: la base SQLite live, en lecture seule, via le plugin `frser-sqlite-datasource` `4.0.6`. Prometheus ne garde que 10 jours, donc il ne convient pas à l'historique des habitudes.
- Grafana est maintenant fixé sur `datavault`. Il monte `/srv/praxis` en lecture seule avec le groupe supplémentaire `65532`. Le répertoire passe en mode `750`.
- Panneaux: % d'habitudes faites par jour avec la note de la journée, séries en cours, grille habitude × jour. Plage par défaut: 30 jours, mise en page pour navigateur de bureau.
- Règles: un jour non enregistré est un trou et coupe les séries. Les séries partent d'aujourd'hui si la journée est enregistrée, sinon d'hier. Une habitude n'apparaît que les jours où elle était active.
- Les requêtes SQL ont été testées sur une copie des vraies données et sur un scénario synthétique (trou, habitude archivée, nouvelle habitude).

## 2026-11-17 - Sauvegarde de Praxis vers Cloudflare R2

J'ai ajouté la CronJob `praxis-backup`. Elle tourne chaque nuit à 03:30 (heure de Toronto) sur `datavault`.

- `sqlite3 .backup` prend un instantané cohérent pendant que Praxis tourne. `PRAGMA integrity_check` doit répondre `ok`.
- rclone envoie la copie dans le bucket R2 `praxis-backups`. Une règle de cycle de vie R2 supprime les copies après 30 jours.
- Test de restauration réussi: la copie téléchargée depuis R2 est intègre et contient les mêmes nombres de lignes que la base live (14 habitudes, 1 jour, 14 valeurs).
- La procédure de restauration est dans `AGENTS.md`.

## 2026-11-17 - TLS Let's Encrypt avec cert-manager

J'ai installé cert-manager `v1.21.3` avec le challenge DNS-01 sur Cloudflare.

- Le token Cloudflare est limité à la zone `lamiphil.dev` et chiffré avec SOPS.
- cert-manager vérifie les enregistrements TXT via `1.1.1.1` et `1.0.0.1`, parce que l'UDM répond localement pour `lamiphil.dev`.
- Il y a deux ClusterIssuers, `letsencrypt-staging` et `letsencrypt-prod`. Un certificat de staging temporaire a été émis puis supprimé pour valider la chaîne avant la prod.
- L'Ingress Praxis utilise maintenant `letsencrypt-prod` et le secret `praxis-tls`.
- Le déploiement s'est fait en deux commits, parce que la Kustomization Flux plate refuse les ClusterIssuers tant que les CRDs n'existent pas.

## 2026-11-17 - Déploiement de Praxis

**2026-11-17 | 21:56**

J'ai ajouté les manifests Praxis sous `clusters/homelab/praxis/`.

- Image publique `ghcr.io/lamiphil/praxis:0.1.0`, jamais `latest`.
- Mise à jour vers `0.1.1`: `Referrer-Policy: same-origin`. Avec `no-referrer`, les navigateurs envoyaient `Origin: null` et la connexion était refusée («origine interdite»).
- Mise à jour vers `0.1.2`: changer la date dans le sélecteur charge la journée tout de suite. Avant, il fallait cliquer sur «Voir», et la page pouvait afficher une date pendant que les cases et l'enregistrement visaient une autre journée.
- Un seul replica avec la stratégie `Recreate`, puisque SQLite n'accepte qu'un écrivain.
- Stockage: PV hostPath `Retain` sur `datavault:/srv/praxis`, avec une node affinity qui fixe le pod sur `datavault`. Le répertoire doit exister et appartenir à l'UID/GID `65532`.
- Conteneur non root, système de fichiers racine en lecture seule, sondes sur `/healthz`, cookie de session `Secure`.
- Ingress Traefik sur `praxis.lamiphil.dev`, entrypoint `websecure` seulement. Le certificat reste celui par défaut de Traefik jusqu'à la configuration DNS-01.
- Le mot de passe et la clé de session sont dans `praxis-secret.sops.yaml`.

## 2026-11-16 - Dashboard de santé du cluster

**2026-11-16 | 22:53**

J'ai désactivé les dashboards Grafana fournis par défaut par `kube-prometheus-stack`.

- Configuré `grafana.defaultDashboardsEnabled: false` dans le HelmRelease.
- Le dashboard GitOps `Homelab Cluster Health` reste provisionné séparément et sera conservé.
- Aucun changement aux règles Prometheus ou Alertmanager.

**2026-11-16 | 22:45**

J'ai ajouté un dashboard Grafana GitOps `Homelab Cluster Health`.

- Vue compacte pour navigateur desktop avec une fenêtre de six heures et un rafraîchissement de 30 secondes.
- Ajouté les états de l'API, de `magos`, des Nodes, des pods et des workloads, les redémarrages récents, les alertes, ainsi que l'utilisation CPU, mémoire, disque et PVC.
- Appliqué des seuils d'avertissement à 75 % et critiques à 90 %.
- Exclu du résumé de santé `Watchdog` et les alertes incompatibles avec k3s `KubeProxyDown`, `KubeControllerManagerDown` et `KubeSchedulerDown`.
- Aucun changement aux règles Prometheus ou Alertmanager.

## 2026-11-15 - Simplification des services

**2026-11-15 | 22:48**

J'ai corrigé le HelmRelease Immich.

- Le chart `0.12.1` n'accepte plus `immich.persistence.upload`; seul `immich.persistence.library` est monté dans le pod serveur à `/data`.
- Retiré la valeur invalide sans supprimer le PVC, le PV ni le répertoire `/srv/media/immich-upload`; les données existantes sont donc préservées.

**2026-11-15 | 22:44**

J'ai terminé la migration vers la topologie à trois noeuds.

- Désinstallé la release Helm Loki orpheline et vérifié qu'aucune ressource ni aucun secret Helm Loki ne reste.
- Créé des snapshots etcd avant le retrait, avant le changement de membership et après la migration. Les snapshots, tokens, configurations et sommes SHA-256 sont sauvegardés hors dépôt sous `/home/lamiphil/backups/homelab/k3s/` avec des permissions `0600`.
- Reconfiguré `minipc`, `datavault` et le kubeconfig local pour utiliser l'API de `magos` à `https://192.168.42.15:6443`.
- Drained `pi` et `archlinux`, désactivé leurs services k3s-agent, puis supprimé leurs objets Node et leurs pods DaemonSet orphelins.
- Drained `ubuntuserver`, retiré son membre etcd avec l'annotation gérée par k3s, désactivé son service k3s, puis supprimé son objet Node.
- Retiré l'ancien endpoint `192.168.42.10` de la configuration de `magos`; son redémarrage comme unique membre etcd a réussi.
- Le cluster final contient `magos`, `minipc` et `datavault`, tous `Ready`; tous les pods sont `Running` et la Kustomization Flux est saine.
- Le HelmRelease Immich reste en erreur sur le schéma `immich.persistence.upload`, mais ses quatre pods sont sains. Cette erreur existait avant la migration.

**2026-11-15 | 18:14**

J'ai analysé l'indisponibilité de `minipc` après avoir obtenu un accès SSH.

- Le redémarrage précédent n'était pas propre: aucun arrêt n'est enregistré et systemd-journald a remplacé un journal corrompu au démarrage.
- Les derniers logs conservés, datés du 25 septembre, montrent que k3s-agent ne pouvait plus joindre `192.168.42.10:6443` (`no route to host`) et ne pouvait plus publier l'état du Node.
- Les logs s'arrêtent ensuite brutalement; ils ne permettent pas de distinguer une panne d'alimentation d'un gel complet de l'hôte.
- `minipc` utilise uniquement le Wi-Fi via une ancienne carte Intel 7260; un test de 20 paquets n'a perdu aucun paquet, mais la latence variait de 0,6 à 299 ms avec une moyenne de 107 ms. L'interface Ethernet est inactive.
- Cinq erreurs `Too many open files` affectaient `systemd-networkd-wait-online` les 26 et 27 août. Elles indiquent une ancienne pression sur les descripteurs, sans preuve qu'elle a causé l'arrêt final.
- Après redémarrage, k3s-agent, la mémoire, le disque, les températures et les unités systemd sont sains.

**2026-11-15 | 18:05**

J'ai débloqué la suppression des namespaces retirés.

- `pihole` était retenu par trois finalizers MetalLB `service.kubernetes.io/load-balancer-cleanup` datant d'anciennes suppressions; retiré ces finalizers des Services déjà en cours de suppression.
- `streaming` et `nvidia-device-plugin` étaient retenus par les pods Jellyfin et NVIDIA sur `archlinux`, dont le kubelet est indisponible; supprimé de force ces deux objets Pod.
- Le PV Jellyfin utilisait la politique `Retain` et le partage NFS `192.168.42.11:/srv/media`; Kubernetes n'a donc pas demandé la suppression des données NFS. Le chemin n'a pas pu être vérifié par SSH.
- Les namespaces `pihole`, `streaming` et `nvidia-device-plugin` sont maintenant supprimés.
- L'API Kubernetes reste saine.

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
