---
project: homelab
type: log
tags: [homelab, log, kubernetes, k3s]
---
# Homelab - Journal de travail

## 2026-11-22 - Homepage : nouvelle mise en page et épinglage de version

**2026-11-22 | 13:20**

Refonte de la landing page `homepage.lamiphil.dev` pour qu'elle serve de vrai tableau de bord LAN. L'image bascule de `ghcr.io/gethomepage/homepage:latest` à `v2.8.0` (`imagePullPolicy: IfNotPresent`), ce qui fige la version actuellement en prod. Le widget `resources` utilisait `backend: resources` (invalide) et est désormais `backend: kubernetes` + `expanded: true` pour afficher CPU/RAM/disque du cluster.

`widgets.yaml` affiche maintenant une bande d'info `datetime` + `search` + `kubernetes` (cluster et nœuds) + `resources`. `services.yaml` est regroupé en quatre colonnes — Applications (Immich, File Browser, Praxis), Monitoring (Grafana, Prometheus, Alertmanager), Identity (Authentik), Infrastructure (Cloudflare dashboard/DNS/R2 → dashboard Grafana R2, Fastmail) — chaque tuile portant son `namespace` + `app` pour que Homepage affiche le statut des pods via la ClusterRole existante. `settings.yaml` reçoit titre, thème sombre slate, fond Unsplash flouté, dispositions par groupe et `statusStyle: dot`.

Les widgets de service (Immich, Grafana, Prometheus, Authentik) ont été préparés puis retirés : la génération des tokens a échoué deux fois à cause d'une règle SOPS que je lisais mal (le nom du fichier source doit matcher `*-secret.sops.yaml`). Les commandes shred étaient exécutées sans vérification, donc les plaintext tokens ont été perdus. Les tokens Authentik et Immich créés durant l'essai doivent être supprimés dans leurs UI respectives. À reprendre plus tard avec `sops -e -i /tmp/homepage-widgets-secret.sops.yaml` suivi d'une vérification `grep ENC\[AES256_GCM` avant tout `mv` ou `shred`.

`HOMEPAGE_ALLOWED_HOSTS` reste à `gethomepage.dev,192.168.42.61,homepage.homelab,homepage.lamiphil.dev`.

**2026-11-22 | 13:45**

Corrigé l'icône Fastmail et clôturé la refonte Homepage.

- `mail.svg` n'existe pas dans `dashboard-icons` ; remplacé par `fastmail.svg` (commit `4546f22`).
- Flux a rapproché, la tuile Fastmail affiche maintenant son logo après un rafraîchissement forcé (service worker).
- Widgets de service (Immich, Grafana, Prometheus, Authentik) reportés : deux pertes successives des tokens plaintext à cause d'une mauvaise lecture de la règle SOPS (le nom du fichier d'entrée doit matcher `*-secret.sops.yaml`). Phase 1 (mise en page + épinglage `v2.8.0` + fix `resources`) livrée et stable.

### Prochaine étape

- Ajouter les widgets de service en utilisant `sops -e -i /tmp/homepage-widgets-secret.sops.yaml` puis `grep ENC\[AES256_GCM` avant tout `shred`.
- Supprimer dans Authentik et Immich les tokens `homepage-widget` / `homepage` créés lors des tentatives.

## 2026-11-19 - Nettoyage des docs publiques

**2026-11-19 | 22:06**

Le dépôt est public sur GitHub. Passe de nettoyage des docs pour réduire la valeur de reconnaissance ciblée, sans toucher au comportement du cluster.

- `AGENTS.md`, `_log.md` : retiré les noms de comptes (`akadmin`, l'utilisateur quotidien), le prénom de l'opérateur, les mentions du fournisseur SMTP et l'email personnel `philippe.lamy@lamiphil.dev` (remplacé par l'alias générique `admin@lamiphil.dev`).
- `clusters/homelab/filebrowser/deployment.yaml` : commentaire scrubé (`Only the Documents folder`), aucun impact runtime.
- Laissé tel quel : le domaine `lamiphil.dev` et les hostnames `*.lamiphil.dev` (déjà exposés via les logs Certificate Transparency), l'org GitHub `lamiphil/homelab`, les manifests fonctionnels (adresses IP MetalLB, `loadBalancerIP`, `admin_email` Grafana).
- Non traité à la demande : réécriture de l'historique (`git filter-repo`) pour scrubber les anciens commits et les emails d'auteur professionnels.

Validation : `kubectl kustomize clusters/homelab` rend sans changement, `git diff --check` OK.

## 2026-11-19 - Documentation à jour des sauvegardes

**2026-11-19 | 21:56**

Rafraîchi la documentation pour refléter l'état actuel du cluster après le nettoyage de `/srv/media`, le cadrage de File Browser sur `Documents/` et l'ajout de la sauvegarde Immich vers R2.

- `docs/homelab-topology.drawio` : la pile minipc n'affiche plus Homepage (planifiée librement et actuellement sur magos), le volume `/srv/media` est séparé en `/srv/media/photos` (bibliothèque Immich) et `/srv/media/Documents` (NFS pour File Browser), Grafana lit maintenant `/srv/praxis` en read-only, et trois arcs de sauvegarde pointent vers Cloudflare R2 avec leur heure et leur bucket.
- `docs/homelab-backups.drawio` : nouveau diagramme dédié aux trois CronJobs nocturnes (Immich 03:00, Praxis 03:30, Authentik 04:00), chacune avec sa source, sa méthode rclone, son bucket R2 et sa politique de rétention 30 jours.
- `docs/homelab-access.drawio` : inchangé côté flux ; seule l'étiquette du tuile sauvegarde Authentik gagne l'heure (04:00).
- `README.md` : la ligne minipc ne liste plus Homepage, une section « Nightly backups » embarque le nouveau diagramme et un tableau synthétique (source, heure, nœud, bucket, rétention) qui renvoie vers AGENTS.md pour les restaurations.
- AGENTS.md laissé tel quel : les tables de stockage, de secrets et les procédures de restauration étaient déjà à jour (commits 6f216e5, 44d3f59, b402568).

Les PNG sont régénérés avec `drawio -x -f png -e -s 2 -b 20` et réparés via `repair_png.py` ; les XML draw.io sont réembarqués. Aucun manifeste Kubernetes touché.

## 2026-11-19 - Coût mensuel R2 et correction de la taille stockée

**2026-11-19 | 17:40**

Ajout de quatre panneaux « Projected monthly storage » au dashboard `Homelab R2 Backups` (Praxis, Authentik, Immich, plus un total compte). Le coût est calculé à partir du dernier `max.payloadSize` renvoyé par l'API GraphQL Cloudflare, multiplié par `0,015 / 1e9` via une transformation Grafana `calculateField` (binary). Le panneau total agrège les trois buckets avec `groupBy` + `reduce` et soustrait les 10 Go-mois gratuits ; une valeur négative signifie que le compte est encore sous le free tier. Les coûts d'opérations Class A/B restent cachés : ils sont négligeables pour un usage sauvegarde (quelques milliers de requêtes par mois).

En vérifiant les chaînes de calcul, j'ai découvert que le parser backend d'Infinity 3.11.1 n'évalue pas les expressions JSONata (`$sum`, additions, multiplications) : il ne sait que suivre un chemin brut. Le panneau « Stored size per bucket » renvoyait donc en réalité que des `null` depuis le déploiement. J'ai basculé toutes les colonnes de taille vers `max.payloadSize` seul (metadata < 0.1 %) et déporté toute l'arithmétique côté Grafana. Les tailles courantes vues en live : praxis 148 KB, authentik 6.3 MB, immich 20.2 GB.

Commit : `MONITORING - Afficher le cout mensuel R2 et corriger la taille`. AGENTS.md mis à jour pour documenter la contrainte JSONata et le modèle de coût.

## 2026-11-19 - Nettoyage de /srv/media et File Browser

**2026-11-19 | 17:22**

Nettoyage de `/srv/media` sur minipc, après inventaire et confirmation dossier par dossier. Supprimés définitivement : `Movies/` (263G), `TV/` (89G) et `downloads/` (340G, copies des films plus un livre audio et un ISO), restes de la pile streaming retirée le 2026-11-15, et `.Trash-1000/` (12M). Environ 690G libérés ; le disque passe de 772G à 82G utilisés. Aucun pod ne montait ces dossiers et Immich n'a aucune bibliothèque externe (6086 assets, tous sous `/data`). `photos/` n'a pas été touché.

File Browser ne montre plus que `Documents/` : le volume NFS est monté avec `subPath: Documents` sur `/srv`. La racine (`/srv`) et le scope utilisateur (`.`) de sa base restent valides.

PV/PVC `immich-upload-local` retirés de Git (vides, plus montés depuis le 2026-11-15 ; les uploads Immich sont dans `photos/upload`). Flux les a supprimés ; le dossier vide `/srv/media/immich-upload` a été supprimé à la main sur minipc. Immich vérifié sain après le prune (4 déploiements 1/1, API OK). La première sauvegarde R2 d'Immich était encore en cours à ce moment.

**2026-11-19 | 18:33**

Terminé le nettoyage de `/srv/media` et la restriction de File Browser à `Documents/`.

- J'ai fait l'inventaire avant toute suppression : taille, nombre de fichiers, dernière modification et workload pour chaque dossier. Chaque suppression a été confirmée dossier par dossier.
- `Movies/`, `TV/`, `downloads/` et `.Trash-1000/` sont supprimés pour de bon (~690G). `photos/` (bibliothèque Immich live) et `lost+found/` sont intacts.
- File Browser monte seulement `Documents/` (`44d3f59`), toujours derrière Authentik.
- PV/PVC `immich-upload-local` retirés ; une autre session les a inclus dans `db08e8d` et `b402568` (titres `MONITORING`). L'opérateur a supprimé le dossier vide sur minipc.
- Immich vérifié sain après le prune : 4 déploiements 1/1, API OK.

### Prochaine étape

- Vérifier que le job `immich-backup-first` (première sauvegarde R2 d'Immich) termine avec succès.

## 2026-11-19 - Tableau de bord Grafana des sauvegardes R2

**2026-11-19 | 17:05**

Objectif : visualiser dans Grafana la fraicheur, la taille, le nombre d'objets et les operations Class A/B des trois buckets R2 (`praxis-backups`, `authentik-backups`, `immich-backups`).

Decision : requeter l'API Cloudflare GraphQL Analytics directement depuis Grafana via le plugin `yesoreyeram-infinity-datasource`. Aucun exporter Prometheus maintenu ne couvre `r2StorageAdaptiveGroups` / `r2OperationsAdaptiveGroups`, et l'approche Infinity reutilise exactement le pattern du plugin `frser-sqlite-datasource` (preinstall_sync + `additionalDataSources`), sans nouveau workload ni storage persistant. La fraicheur des sauvegardes reste cote Prometheus via `kube_cronjob_status_last_successful_time` (kube-state-metrics).

Token dedie cree sur Cloudflare (politique "Read analytics & logs", scope compte uniquement, aucun acces aux contenus R2). Le token est lu depuis l'env `CLOUDFLARE_API_TOKEN` fourni par `grafana-cloudflare-r2`, et Grafana le substitue dans `secureJsonData.httpHeaderValue1` du datasource au provisioning (`Authorization: Bearer $CLOUDFLARE_API_TOKEN`). Le tag de compte Cloudflare n'etant pas un secret, il est fige en variable constante masquee du dashboard.

Modifications GitOps :

- `clusters/homelab/monitoring/kube-prometheus-stack/helmrelease.yaml` : plugin `yesoreyeram-infinity-datasource@4.1.8` ajoute a `preinstall_sync`, `grafana-cloudflare-r2` ajoute a `envFromSecrets`, et nouveau datasource `Cloudflare R2` (UID `cloudflare-r2`, URL `https://api.cloudflare.com/client/v4/graphql`, `allowedHosts` restreint a `api.cloudflare.com`).
- `clusters/homelab/monitoring/kube-prometheus-stack/grafana-cloudflare-r2-secret.sops.yaml` : Secret SOPS portant `CLOUDFLARE_API_TOKEN`.
- `clusters/homelab/monitoring/kube-prometheus-stack/r2-backups-dashboard.yaml` : dashboard `Homelab R2 Backups` (UID `homelab-r2-backups`, refresh 30 s, timezone browser, defaut `now-7d`). Trois stats de fraicheur (une par CronJob, seuils 28 h / 36 h), quatre graphes Cloudflare (taille, nombre d'objets, Class A, Class B) avec une serie par bucket. UQL Infinity agrege les lignes GraphQL (sum, order by time).
- `clusters/homelab/kustomization.yaml` : enregistre les deux nouveaux fichiers.

Validations avant commit : `kubectl kustomize clusters/homelab` rend la ConfigMap du dashboard ; `helm template kube-prometheus-stack-79.5.0` confirme le datasource, le plugin et l'envFrom ; appel GraphQL reel avec l'exact body d'un panneau retourne 83 points pour `praxis-backups` sur 7 jours ; Prometheus retourne les ages de sauvegarde pour `praxis-backup` et `authentik-backup` (Immich n'a pas encore eu sa premiere execution a 03:00, la carte affichera "Never" jusque-la). Deux fuites de jeton pendant la rédaction ont ete corrigees : le token a ete regenere, l'encryption SOPS est maintenant faite via stdin sans jamais ecrire de clair sur disque.

## 2026-11-19 - Dashboard Cluster Resources

**2026-11-19 | 17:02**

Nouveau dashboard Grafana `Homelab Cluster Resources` (UID `homelab-cluster-resources`) pour la consommation de ressources, filtrable par noeud et par namespace. Il complete le dashboard existant `Homelab Cluster Health` (laisse intact) : une regarde la sante, l'autre la consommation.

- ConfigMap `clusters/homelab/monitoring/kube-prometheus-stack/cluster-resources-dashboard.yaml` (label `grafana_dashboard=1`), ajoutee a `clusters/homelab/kustomization.yaml` juste apres la ConfigMap health.
- Variables `node` et `namespace` (multi-select, option All), construites avec `label_values(node_uname_info, nodename)` et `label_values(kube_pod_info, namespace)`. La variable node ne liste que les trois noeuds vivants (magos, minipc, datavault) ; les anciens hostnames encore presents dans l'etiquette brute `nodename` de Prometheus sont exclus.
- Timezone navigateur, plage par defaut 6 h, rafraichissement 30 s, seuils communs 75 %/90 %.
- 14 panneaux sur un ecran bureau (grid 24, hauteur ~35) :
  - ligne 1 : cinq cartes (CPU cluster %, RAM cluster %, disque racine max %, pods CPU filtres, pods RAM filtres) ;
  - ligne 2 : CPU, RAM et I/O reseau par noeud (TX dessine en negatif) ;
  - ligne 3 : systemes de fichiers par noeud/mountpoint, table PVC filtree par namespace ;
  - ligne 4 : CPU et memoire des pods en requests vs limits vs utilisation reelle ;
  - ligne 5 : tables de restarts (increase 1 h) et d'OOMKills (changes 24 h).
- Les metriques node-exporter sont jointes a `node_uname_info` par `on(instance) group_left(nodename)` puis filtrees sur `$node`. Les panneaux pods utilisent les etiquettes `namespace` et `node` deja presentes sur cAdvisor et kube-state-metrics, donc filtrent les deux variables sans jointure.
- Les volumes hostPath (/srv sur minipc, /datavault sur datavault, /srv/praxis et /srv/authentik) n'apparaissent pas dans `kubelet_volume_stats_*` : le panneau systemes de fichiers par noeud couvre leur usage.
- Liens croises : le nouveau dashboard pointe vers `Homelab Cluster Health`. L'ancien n'est pas modifie.
- Validation : chaque requete PromQL interrogee a Prometheus via `kubectl port-forward`, kustomize rendu complet, `kubectl apply --dry-run=server` sur la ConfigMap, parsing JSON et `git diff --check` OK. Aucun `kubectl apply` persistant.

## 2026-11-19 - Sauvegarde Immich vers R2

**2026-11-19 | 16:40**

Mesure prealable sur minipc : `/data/upload` contient 64 GiB d'originaux (6 144 fichiers), le reste de `/data` (thumbs, encoded-video) est regenerable et exclu. La base PostgreSQL Immich fait 207 MB (~40 MB compresse). Croissance observee : ~10 GiB par an sur 2025-2026 ; le pic 2023 (+30 GiB) etait un import en bloc.

Comparaison de cinq destinations faite (R2 Standard, R2 Infrequent Access, Backblaze B2, Hetzner Storage Box BX11, AWS Glacier Deep Archive) plus variantes rclone nu vs restic/kopia. Decision : Cloudflare R2 Standard avec `rclone sync` pour reutiliser exactement le pattern des CronJob `praxis-backup` et `authentik-backup`. Cout attendu : ~1 USD/mois aux 65 GiB actuels, ~2 USD/mois au double, egress gratuit pour les restaurations.

R2 n'expose pas encore le versioning d'objets dans le tableau de bord. Protection de suppression obtenue autrement : `rclone sync --backup-dir r2:immich-backups/trash/<date> --suffix -<ts>` deplace tout fichier ecrase ou supprime dans `trash/`, et la regle de cycle de vie `expire-trash-30d` sur le prefixe `trash/` purge apres 30 jours. Chiffrement cote client refuse (le compte R2 est deja scopé par token) ; les originaux Immich restent lisibles dans le tableau de bord R2.

Ajouts GitOps :

- `clusters/homelab/immich/backup-cronjob.yaml` : CronJob `immich-backup` a 03:00 America/Toronto, nodeSelector `minipc`, concurrencyPolicy Forbid, `activeDeadlineSeconds: 21600`. InitContainer `dump` (`postgres:16.10-alpine`, UID 70) : `pg_dump --format=custom` puis `pg_restore --list` pour valider. Container `upload` (`rclone/rclone:1.76.0`, UID 65532) : sync originals vers R2 avec `--backup-dir`/`--suffix`/`--bwlimit 8M`/`--tpslimit 50`, puis `rclone copyto` du dump sous `db/immich-<ts>.dump`, puis `rclone cat` de controle. Montage PVC `immich-library-local` en lecture seule, root filesystem read-only, caps drop ALL.
- `clusters/homelab/immich/immich-backup-r2-secret.sops.yaml` : Secret `immich-backup-r2` chiffre SOPS, cinq cles `RCLONE_CONFIG_R2_*`. Token R2 d'ecriture limite au bucket `immich-backups`.
- Entrees ajoutees a `clusters/homelab/kustomization.yaml`.
- AGENTS.md : section `## Immich backups` avec procedure de restauration.

A faire cote Cloudflare avant que le job serve a quelque chose : activer la regle de cycle de vie sur prefixe `trash/` (`Delete uploaded objects: 30 days`) et, en option, `abort-incomplete-7d` sur tout le bucket. Premiere execution a declencher manuellement via `kubectl -n immich create job immich-backup-manual --from=cronjob/immich-backup` ; sur Wi-Fi minipc, prevoir ~15 h pour les 64 GiB initiaux (reprise automatique via rclone).

**2026-11-19 | 16:55**

Deploye le CronJob `immich-backup` (commit `6f216e5`) et declenche la premiere execution manuelle.

- Flux a reconcilie `main@6f216e5` ; le Secret SOPS `immich-backup-r2` s'ouvre avec les cinq cles `RCLONE_CONFIG_R2_*`.
- `kubectl -n immich create job immich-backup-first --from=cronjob/immich-backup` : pod `immich-backup-first-pqgkr` sur `minipc`, initContainer `dump` termine en 17 s (dump custom de 42 MB valide par `pg_restore --list`), container `upload` en cours (`rclone sync` des 64 GiB d'originaux).
- Le `logs -f` initial a expire sur `PodInitializing` pendant le pull de l'image postgres ; le job lui-meme avance normalement.

### Prochaine etape

- Ajouter dans le tableau de bord R2 les regles de cycle de vie `expire-trash-30d` (prefixe `trash/`) et `abort-incomplete-7d`.
- Surveiller la fin du premier `immich-backup-first` et confirmer les objets sous `originals/` et `db/` dans R2.

## 2026-11-19 - TLS pour les services du homelab

**2026-11-19 | 16:34**

Ajouté une section Infrastructure à Homepage avec les liens Cloudflare (dashboard, DNS lamiphil.dev, R2 buckets) et le fournisseur mail. Icônes cloudflare.svg et mail.svg. Commit 7ce262c poussé et réconcilié par Flux.

**2026-11-19 | 16:10**

Homepage : ajout d'une section Infrastructure avec trois liens vers le dashboard Cloudflare (dashboard, DNS de la zone lamiphil.dev, R2 buckets praxis-backups et authentik-backups). Homepage n'est pas protégée par Authentik ; la section utilise l'icône cloudflare.svg fournie par Homepage.

**2026-11-19 | 09:28**

Documentation : le README décrit l'accès aux services et le SSO, avec deux diagrammes draw.io à icônes (`docs/homelab-access` et `docs/homelab-topology`, sources `.drawio` et exports `.drawio.png` modifiables). Les anciens diagrammes de la topologie à six nœuds sont supprimés.

**2026-11-19 | 04:13**

Bilan du déploiement SSO Authentik :

- Homepage reste ouvert. Prometheus, Alertmanager, File Browser et Praxis passent par le ForwardAuth (`b9a0ebe`). Grafana (`d1f3a6e`) et Immich (web et mobile) utilisent l'OIDC natif.
- Plus aucun service protégé n'a d'IP MetalLB : `.70`, `.71`, `.72`, `.80` et `.96` sont fermées (`171fbdd`, `8f07161`). Seuls Traefik `.60` et Homepage `.61` gardent une IP.
- La configuration faite dans l'interface d'Authentik n'est pas dans Git : le groupe `homelab-users`, l'utilisateur quotidien, le fournisseur proxy `homelab-forward-auth` et l'application `Homelab` (liés à l'outpost intégré), et les fournisseurs OAuth2 `grafana` et `immich` (confidentiels, Authorization Code et Refresh token seulement, clé de signature auto-signée, consentement implicite). Seule la sauvegarde nocturne R2 d'Authentik permet de la restaurer.
- Les réglages OAuth d'Immich sont aussi dans son interface d'administration, donc ils dépendent de la base Immich et non de Git.
- Points de secours : le formulaire local de Grafana, `https://immich.lamiphil.dev/auth/login?autoLaunch=0`, et `ak create_recovery_key 10 l'administrateur Authentik` dans le worker Authentik.
- Reste à faire : enrôler un authentificateur (TOTP et jetons statiques) pour le compte quotidien, puis rendre la MFA obligatoire dans l'étape `default-authentication-mfa-validation`. Optionnel : retirer la deuxième connexion de File Browser et de Praxis.

**2026-11-19 | 04:05**

L'application mobile Immich utilise maintenant `https://immich.lamiphil.dev` avec connexion Authentik. Le LoadBalancer Immich `.80` devient ClusterIP : seuls Traefik `.60` et Homepage `.61` gardent une IP MetalLB.

**2026-11-19 | 03:59**

Grafana SSO validé; Immich OAuth activé dans l'interface (auto-launch actif, connexion par mot de passe via `/auth/login?autoLaunch=0`) et compte existant lié. Le LoadBalancer Grafana `.70` devient ClusterIP. Immich garde `.80` tant que l'application mobile n'utilise pas `https://immich.lamiphil.dev`.

**2026-11-19 | 03:51**

Grafana : connexion OIDC native avec Authentik (application `grafana`, RS256, groupe `homelab-users` -> rôle Admin). Le client ID et le secret sont dans `grafana-oidc-secret.sops.yaml`. Le stockage de Grafana est éphémère, donc l'admin local est recréé à chaque démarrage avec l'email `admin@lamiphil.dev` pour que la connexion SSO s'y rattache. Le formulaire local reste disponible en secours. Immich passe aussi en OIDC (application `immich`), configuré dans l'interface d'administration d'Immich.

**2026-11-19 | 03:36**

Homepage n'est plus derrière Authentik, à la demande de l'opérateur. Prometheus, Alertmanager, File Browser et Praxis passent maintenant par le ForwardAuth Authentik; le BasicAuth du monitoring et son Secret SOPS sont supprimés. Les LoadBalancers `.71`, `.72` et `.96` deviennent ClusterIP pour supprimer tout contournement. File Browser et Praxis gardent leur propre connexion pour l'instant. Grafana et Immich passeront en OIDC natif.

**2026-11-19 | 03:26**

Pilote SSO : Homepage passe par un middleware Traefik ForwardAuth vers l'outpost intégré d'Authentik (fournisseur `homelab-forward-auth`, mode domaine, cookie `lamiphil.dev`, accès réservé au groupe `homelab-users`). Le LoadBalancer `.61` reste temporairement comme retour arrière et contourne encore l'authentification.

**2026-11-19 | 03:03**

Flux a appliqué la configuration SMTP et redémarré le serveur et le worker Authentik sans erreur. La commande `ak test_email` a confirmé l'envoi d'un courriel à `admin@lamiphil.dev` par le fournisseur SMTP.

**2026-11-19 | 03:00**

J'ai configuré l'envoi SMTP Authentik par le fournisseur SMTP sur le port 587 avec STARTTLS. Les courriels utilisent `Homelab <homelab@lamiphil.dev>` comme expéditeur; l'identifiant et le mot de passe applicatif sont conservés dans un Secret chiffré SOPS.

**2026-11-19 | 02:32**

Flux a déployé Authentik avec succès. Le HelmRelease, PostgreSQL, le serveur, le worker et le certificat `authentik-tls` sont Ready; `https://auth.lamiphil.dev/if/flow/initial-setup/` répond en HTTPS avec un certificat valide.

J'ai exécuté une sauvegarde manuelle complète et vérifié dans R2 la présence de la paire PostgreSQL et fichiers Authentik. Le Job de validation a ensuite été supprimé.

**2026-11-19 | 02:16**

J'ai ajouté Authentik `2026.8.5` à `auth.lamiphil.dev` comme futur fournisseur d'identité. Le serveur, le worker et PostgreSQL sont épinglés sur `datavault`; les secrets sont chiffrés avec SOPS, et les données persistantes restent sous `/srv/authentik`.

Un CronJob crée chaque nuit un dump PostgreSQL validé et une archive des fichiers Authentik, puis les envoie dans le bucket R2 dédié `authentik-backups`, dont la rétention est gérée par une règle de cycle de vie de 30 jours. L'activation MFA et les intégrations SSO seront configurées après la création du premier compte administrateur.

**2026-11-19 | 01:58**

J'ai remplacé tous les liens internes de Homepage par les endpoints HTTPS `*.lamiphil.dev` et ajouté Praxis à la section Applications.

**2026-11-19 | 01:53**

J'ai ajouté `homepage.lamiphil.dev` à `HOMEPAGE_ALLOWED_HOSTS` après que le test HTTPS a retourné `400 Host validation failed`.

**2026-11-19 | 01:45**

J'ai préparé la première phase de migration des services vers Traefik et HTTPS.

- Ajouté des Ingress cert-manager pour Homepage, File Browser, Immich, Grafana, Prometheus et Alertmanager sous `*.lamiphil.dev`.
- Réutilisé le ClusterIssuer `letsencrypt-prod` et le point d'entrée Traefik `websecure` validés avec Praxis.
- Protégé Prometheus et Alertmanager avec un Middleware BasicAuth et un hash bcrypt chiffré par SOPS; aucun mot de passe en clair n'est conservé.
- Conservé temporairement les Services LoadBalancer existants pour permettre la validation et un retour arrière avant leur conversion en ClusterIP.

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
- Créé des snapshots etcd avant le retrait, avant le changement de membership et après la migration. Les snapshots, tokens, configurations et sommes SHA-256 sont sauvegardés hors dépôt sous `/home/admin/backups/homelab/k3s/` avec des permissions `0600`.
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
