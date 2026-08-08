# Trace d’exécution — Longhorn puis migration Nextcloud

> **Périmètre** : depuis la préparation de Talos pour Longhorn jusqu’au rétablissement de Nextcloud via Cilium et Tailscale.  
> **Source** : reconstruction fidèle à partir de notre échange.  
> **Important** : les éléments explicitement observés dans les sorties sont marqués **confirmés**. Les commandes ou champs YAML non visibles dans l’historique sont marqués **reconstruits** et doivent être comparés au dépôt Git avant d’être considérés comme une copie exacte.

---

## 1. État du cluster au début de cette phase

### Nœuds

| Rôle | Nom | IP |
|---|---|---|
| Control plane | `naruto` | `192.168.1.10` |
| Worker | `kakashi` | `192.168.1.12` |
| Worker | `tsunade` | `192.168.1.14` |

### Versions connues

- Kubernetes : `v1.36.2`
- Talos : `v1.13.7`
- CNI : Cilium
- Stockage initial de Nextcloud/MariaDB : `local-path`
- Stockage cible de Nextcloud : Longhorn
- Release Helm Nextcloud :
  - release : `nextcloud`
  - namespace : `nextcloud`
  - chart : `nextcloud-9.2.5`
  - application : `34.0.2`

---

# 2. Préparation de Talos pour Longhorn

## 2.1 Pourquoi l’extension iSCSI est nécessaire

Longhorn attache des volumes blocs aux nœuds Kubernetes. Sur Talos, les outils iSCSI ne sont pas présents par défaut. L’extension Talos Factory `iscsi-tools` a donc été ajoutée à l’image Talos.

### Résultat confirmé

- Extension Talos Factory ajoutée : `iscsi-tools`
- Longhorn ensuite validé avec un test de persistance.

### Configuration exacte à conserver dans le dépôt Talos

La référence exacte de l’image Talos Factory n’est pas visible dans l’historique fourni. Elle doit être récupérée depuis les fichiers Talos actuellement utilisés.

Exemple de forme attendue, **à ne pas considérer comme la valeur historique exacte** :

```yaml
machine:
  install:
    image: factory.talos.dev/installer/<SCHEMATIC_ID>:v1.13.7
```

Le schematic Talos Factory doit inclure :

```yaml
customization:
  systemExtensions:
    officialExtensions:
      - siderolabs/iscsi-tools
```

---

# 3. Installation de Longhorn

## 3.1 Dépôt Helm

Commande habituelle, **reconstruite** :

```bash
helm repo add longhorn https://charts.longhorn.io
```

Puis :

```bash
helm repo update
```

Le dépôt `longhorn` était ensuite bien présent dans la configuration Helm locale.

## 3.2 Installation Helm

Commande habituelle, **reconstruite** :

```bash
helm install longhorn longhorn/longhorn \
  --namespace longhorn-system \
  --create-namespace
```

La commande historique exacte et les éventuelles valeurs Helm personnalisées ne sont pas visibles dans l’échange.

## 3.3 Vérifications réalisées

Les vérifications ont porté sur :

- les Pods du namespace `longhorn-system` ;
- la disponibilité des managers Longhorn ;
- le fonctionnement du CSI ;
- l’attachement d’un volume ;
- la persistance après recréation d’un Pod.

Exemples de commandes de vérification :

```bash
kubectl get pods -n longhorn-system
```

```bash
kubectl get storageclass
```

```bash
kubectl get volumes.longhorn.io -n longhorn-system
```

---

# 4. Création d’une StorageClass Longhorn à deux réplicas

## 4.1 Constat

La StorageClass Longhorn par défaut utilisait :

```yaml
numberOfReplicas: "3"
```

Avec seulement deux workers de stockage disponibles, une classe à deux réplicas était plus adaptée.

La modification directe de la StorageClass existante n’était pas possible car plusieurs champs d’une StorageClass sont immuables. Une nouvelle StorageClass a donc été créée.

## 4.2 Fichier créé

Chemin confirmé :

```text
infrastructure/longhorn/manifests/storageclass-2replicas.yaml
```

Nom confirmé :

```text
longhorn-2replicas
```

## 4.3 YAML

Les champs ci-dessous reprennent les éléments confirmés. Certains paramètres optionnels peuvent différer du fichier exact actuellement présent dans le dépôt.

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: longhorn-2replicas
provisioner: driver.longhorn.io
allowVolumeExpansion: true
reclaimPolicy: Delete
volumeBindingMode: Immediate
parameters:
  numberOfReplicas: "2"
  staleReplicaTimeout: "30"
  fsType: ext4
```

## 4.4 Application

Commande confirmée dans son principe :

```bash
kubectl apply -f infrastructure/longhorn/manifests/storageclass-2replicas.yaml
```

Résultat :

- StorageClass `longhorn-2replicas` créée avec succès.

---

# 5. Création du PVC Longhorn de Nextcloud

## 5.1 Fichier créé

Chemin confirmé :

```text
applications/nextcloud/manifests/nextcloud-longhorn-pvc.yaml
```

## 5.2 Caractéristiques confirmées

- nom : `nextcloud-nextcloud-longhorn`
- namespace : `nextcloud`
- taille : `100Gi`
- StorageClass : `longhorn-2replicas`

## 5.3 YAML

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nextcloud-nextcloud-longhorn
  namespace: nextcloud
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: longhorn-2replicas
  resources:
    requests:
      storage: 100Gi
```

## 5.4 Application

```bash
kubectl apply -f applications/nextcloud/manifests/nextcloud-longhorn-pvc.yaml
```

## 5.5 Résultat confirmé

Le PVC est passé à l’état :

```text
Bound
```

avec la StorageClass :

```text
longhorn-2replicas
```

---

# 6. Tentative de migration du volume Nextcloud

## 6.1 PVC source et destination

### Source

```text
nextcloud-nextcloud
```

StorageClass :

```text
local-path
```

### Destination

```text
nextcloud-nextcloud-longhorn
```

StorageClass :

```text
longhorn-2replicas
```

## 6.2 Arrêt temporaire de Nextcloud

Commande exécutée :

```bash
kubectl scale deployment nextcloud --replicas=0 -n nextcloud
```

Objectif :

- garantir qu’aucun fichier ne soit modifié pendant la copie ;
- éviter une migration incohérente.

## 6.3 Pod temporaire de migration

Fichier confirmé :

```text
applications/nextcloud/manifests/nextcloud-volume-migration-pod.yaml
```

Le YAML exact n’est pas visible dans l’historique. Le manifeste ci-dessous est une reconstruction fonctionnelle de ce qui a été utilisé :

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nextcloud-volume-migration
  namespace: nextcloud
spec:
  restartPolicy: Never
  containers:
    - name: migration
      image: alpine:3.20
      command:
        - /bin/sh
        - -c
        - sleep infinity
      volumeMounts:
        - name: source
          mountPath: /source
        - name: destination
          mountPath: /destination
  volumes:
    - name: source
      persistentVolumeClaim:
        claimName: nextcloud-nextcloud
    - name: destination
      persistentVolumeClaim:
        claimName: nextcloud-nextcloud-longhorn
```

## 6.4 Commande de copie

Commande exécutée dans le Pod de migration :

```bash
cp -a /source/. /destination/
```

## 6.5 Résultat de l’inspection

- le PVC source était vide ;
- le PVC destination ne contenait que `lost+found`.

Conclusion :

- aucune donnée utilisateur Nextcloud n’avait encore été stockée ;
- aucune migration de fichiers n’était réellement nécessaire.

## 6.6 Nettoyage

Le Pod temporaire a ensuite été supprimé :

```bash
kubectl delete pod -n nextcloud nextcloud-volume-migration
```

Les PVC n’ont pas été supprimés.

---

# 7. Bascule de Nextcloud vers le PVC Longhorn

## 7.1 Valeur Helm modifiée

Dans :

```text
applications/nextcloud/values.yaml
```

la persistance Nextcloud a été configurée avec :

```yaml
persistence:
  enabled: true
  existingClaim: nextcloud-nextcloud-longhorn
```

## 7.2 Mise à jour Helm

La première tentative a échoué :

```bash
helm upgrade nextcloud applications/nextcloud -n nextcloud -f applications/nextcloud/values.yaml
```

Erreur :

```text
unable to detect chart ... Chart.yaml: no such file or directory
```

Cause :

- `applications/nextcloud` contient les valeurs et les manifests ;
- ce dossier n’est pas lui-même un chart Helm.

La commande correcte a ensuite utilisé le chart distant :

```bash
helm upgrade nextcloud nextcloud/nextcloud \
  --version 9.2.5 \
  -n nextcloud \
  -f applications/nextcloud/values.yaml
```

---

# 8. Réinitialisation de l’installation Nextcloud

## 8.1 Symptôme initial

Nextcloud était dans un état d’installation incomplet :

```text
installed: false
```

Le fichier `config.php` existait mais ne contenait pas :

```php
'installed' => true,
```

Les logs ont notamment signalé :

```text
The Login is already being used
```

## 8.2 Décision

Comme aucune donnée utilisateur n’existait encore, la base Nextcloud a été réinitialisée.

Actions réalisées :

```sql
DROP DATABASE nextcloud;
CREATE DATABASE nextcloud;
```

Les droits ont été conservés ou réappliqués pour l’utilisateur Nextcloud.

## 8.3 Marqueur `CAN_INSTALL`

Nextcloud affichait ensuite :

```text
It looks like you are trying to reinstall your Nextcloud.
However the file CAN_INSTALL is missing.
```

Le marqueur a été créé :

```bash
kubectl exec -n nextcloud nextcloud-64d8d659b7-t5x24 -- \
  touch /var/www/html/config/CAN_INSTALL
```

Puis le Pod a été recréé :

```bash
kubectl delete pod -n nextcloud nextcloud-64d8d659b7-t5x24
```

Résultat :

```text
1/1 Running
```

---

# 9. Configuration Helm Nextcloud finale connue

Contenu observé de `applications/nextcloud/values.yaml`, après suppression du forçage global de protocole :

```yaml
nextcloud:
  host: nextcloud.home.arpa

  trustedDomains:
    - nextcloud.home.arpa
    - 192.168.1.53
    - 100.78.189.101
    - nextcloud.taildf9a0f.ts.net

  configs:
    proxy.config.php: |-
      <?php
      $CONFIG = array (
        'trusted_proxies' => array (
          '10.244.0.0/16',
        ),
      );

  existingSecret:
    enabled: true
    secretName: nextcloud-admin-secret
    usernameKey: nextcloud-username
    passwordKey: nextcloud-password

internalDatabase:
  enabled: false

externalDatabase:
  enabled: true
  type: mysql
  host: nextcloud-mariadb:3306
  database: nextcloud

  existingSecret:
    enabled: true
    secretName: nextcloud-services-secret
    usernameKey: db-username
    passwordKey: db-password

mariadb:
  enabled: true

  auth:
    database: nextcloud
    username: nextcloud
    existingSecret: nextcloud-services-secret

  architecture: standalone

  primary:
    persistence:
      enabled: true
      storageClass: local-path
      accessMode: ReadWriteOnce
      size: 8Gi

redis:
  enabled: true
  architecture: standalone

  auth:
    enabled: true
    existingSecret: nextcloud-services-secret
    existingSecretPasswordKey: redis-password

  master:
    persistence:
      enabled: false

  replica:
    replicaCount: 0
    persistence:
      enabled: false

persistence:
  enabled: true
  existingClaim: nextcloud-nextcloud-longhorn

ingress:
  enabled: true
  className: cilium
  path: /
  pathType: Prefix
  tls: []

startupProbe:
  enabled: true
  initialDelaySeconds: 10
  periodSeconds: 10
  timeoutSeconds: 5
  failureThreshold: 60
  successThreshold: 1

livenessProbe:
  enabled: true
  initialDelaySeconds: 10
  periodSeconds: 10
  timeoutSeconds: 5
  failureThreshold: 6
  successThreshold: 1

readinessProbe:
  enabled: true
  initialDelaySeconds: 10
  periodSeconds: 10
  timeoutSeconds: 5
  failureThreshold: 6
  successThreshold: 1
```

---

# 10. Ingress Cilium local

## 10.1 Service généré

Nom :

```text
cilium-ingress-nextcloud
```

État observé :

```text
TYPE: LoadBalancer
EXTERNAL-IP: 192.168.1.53
PORTS:
  80/TCP
  443/TCP
```

Ingress :

```text
nextcloud.home.arpa
```

Backend :

```text
nextcloud:8080
```

Le Service Nextcloud route vers le Pod sur le port 80 :

```text
10.244.1.239:80
```

## 10.2 Absence de TLS local

La configuration Helm contient :

```yaml
ingress:
  tls: []
```

Conséquence :

- HTTP local fonctionnel ;
- HTTPS local sur `192.168.1.53:443` refusé ;
- `nextcloud.home.arpa` doit être utilisé en HTTP tant qu’aucun certificat local n’est configuré.

Test confirmé :

```bash
curl -I http://nextcloud.home.arpa/login
```

Résultat :

```text
HTTP/1.1 200 OK
server: envoy
```

Test HTTPS :

```bash
curl -vkI https://nextcloud.home.arpa/login
```

Résultat :

```text
Connection refused
```

---

# 11. Résolution DNS locale

Au départ :

```bash
dig +short nextcloud.home.arpa
```

ne retournait rien.

Une entrée locale a été ajoutée sur le Mac :

```text
192.168.1.53 nextcloud.home.arpa
```

Commande utilisée :

```bash
echo '192.168.1.53 nextcloud.home.arpa' | sudo tee -a /etc/hosts
```

Après cela :

```bash
curl -I http://nextcloud.home.arpa/login
```

retournait bien :

```text
HTTP/1.1 200 OK
```

Le ping ICMP ne répondait pas, ce qui n’est pas un test pertinent pour un LoadBalancer Cilium.

---

# 12. Problème `overwriteprotocol`

## 12.1 Symptôme

Nextcloud redirigeait initialement vers :

```text
https://nextcloud.home.arpa/login
```

alors que l’Ingress Cilium local n’avait pas de TLS.

Cause observée dans `config.php` :

```php
'overwriteprotocol' => 'https',
```

Une première correction Helm a temporairement ajouté :

```php
'overwriteprotocol' => 'http',
```

dans `proxy.config.php`.

Cela a corrigé l’accès local, mais risquait de casser l’accès HTTPS via Tailscale, car la même valeur était imposée globalement.

## 12.2 Analyse des fichiers chargés

Fichiers observés dans :

```text
/var/www/html/config
```

notamment :

```text
config.php
proxy.config.php
reverse-proxy.config.php
redis.config.php
```

`reverse-proxy.config.php` utilise les variables d’environnement suivantes lorsqu’elles existent :

```text
OVERWRITEHOST
OVERWRITEPROTOCOL
OVERWRITECLIURL
OVERWRITEWEBROOT
OVERWRITECONDADDR
TRUSTED_PROXIES
FORWARDED_FOR_HEADERS
```

Test :

```bash
kubectl exec -n nextcloud deploy/nextcloud -- printenv OVERWRITEPROTOCOL
```

Résultat :

```text
exit code 1
```

Conclusion :

- la variable n’était pas définie ;
- la surcharge venait bien des fichiers PHP.

## 12.3 Correction définitive

La ligne suivante a été supprimée du `values.yaml` :

```php
'overwriteprotocol' => 'http',
```

Puis Helm a régénéré `proxy.config.php` sans forçage de protocole.

Commande Helm :

```bash
helm upgrade nextcloud nextcloud/nextcloud \
  --version 9.2.5 \
  -n nextcloud \
  -f applications/nextcloud/values.yaml
```

Résultat :

```text
STATUS: deployed
REVISION: 8
```

Vérification du rollout :

```bash
kubectl rollout status deployment/nextcloud -n nextcloud --timeout=120s
```

Résultat :

```text
deployment "nextcloud" successfully rolled out
```

Puis la clé persistante a été supprimée avec `occ` :

```bash
kubectl exec -n nextcloud deploy/nextcloud -- \
  su -s /bin/sh www-data -c \
  'php /var/www/html/occ config:system:delete overwriteprotocol'
```

Résultat :

```text
System config value overwriteprotocol deleted
```

Comportement final :

- requête HTTP locale → redirection HTTP ;
- requête HTTPS Tailscale → protocole HTTPS conservé.

---

# 13. Ingress Tailscale

## 13.1 Ingress observé

Nom :

```text
nextcloud-tailscale
```

Classe :

```text
tailscale
```

Nom public Tailnet :

```text
nextcloud.taildf9a0f.ts.net
```

Backend :

```text
nextcloud:8080
```

TLS :

```text
SNI routes nextcloud
```

## 13.2 Test TLS confirmé

Commande :

```bash
curl -vkI https://nextcloud.taildf9a0f.ts.net/login
```

Résultats réseau :

- résolution vers `100.78.189.101` ;
- connexion TCP 443 réussie ;
- TLS 1.3 ;
- certificat Let’s Encrypt valide ;
- certificat CN :
  `nextcloud.taildf9a0f.ts.net`.

Le serveur retournait initialement :

```text
HTTP/2 400
```

---

# 14. Correction de `trusted_domains`

## 14.1 État réel avant correction

Commande :

```bash
kubectl exec -n nextcloud deploy/nextcloud -- \
  su -s /bin/sh www-data -c \
  'php /var/www/html/occ config:system:get trusted_domains'
```

Résultat :

```text
nextcloud.home.arpa
```

Le domaine Tailscale présent dans `values.yaml` n’avait donc pas été ajouté dans la configuration persistante de l’installation existante.

## 14.2 Ajout du domaine Tailscale

Commande :

```bash
kubectl exec -n nextcloud deploy/nextcloud -- \
  su -s /bin/sh www-data -c \
  'php /var/www/html/occ config:system:set trusted_domains 1 --value=nextcloud.taildf9a0f.ts.net'
```

Résultat :

```text
System config value trusted_domains => 1 set to string nextcloud.taildf9a0f.ts.net
```

Vérification :

```text
nextcloud.home.arpa
nextcloud.taildf9a0f.ts.net
```

## 14.3 Résultat final

- URL Tailscale fonctionnelle ;
- connexion Nextcloud réussie ;
- synchronisation des photos iPhone lancée avec succès.

---

# 15. Secrets et sécurité

Un mot de passe Redis est apparu en clair pendant l’affichage de `config.php`.

Il n’est volontairement pas reproduit dans ce document.

Action recommandée :

1. renouveler la valeur `redis-password` dans le Secret `nextcloud-services-secret` ;
2. mettre à jour la source déclarative de ce Secret ;
3. redéployer la release Nextcloud ;
4. vérifier Redis et Nextcloud ;
5. éviter à l’avenir d’afficher tout le fichier `config.php` lorsque seules quelques clés sont nécessaires.

---

# 16. État final connu

## Nextcloud

```text
Running
Ready 1/1
```

## MariaDB

```text
Running
Ready 1/1
```

## Redis

```text
Running
Ready 1/1
```

## Stockage Nextcloud

```text
PVC: nextcloud-nextcloud-longhorn
StorageClass: longhorn-2replicas
Taille: 100Gi
État: Bound
```

## Accès

### Tailscale

```text
https://nextcloud.taildf9a0f.ts.net
```

État :

```text
fonctionnel
```

### Local Cilium

```text
http://nextcloud.home.arpa
```

Le chemin HTTP répond côté cluster, mais l’utilisation dans le navigateur restait à finaliser. L’accès HTTPS local n’est pas configuré.

---

# 17. Arborescence des fichiers créés ou modifiés

```text
infrastructure/
└── longhorn/
    └── manifests/
        └── storageclass-2replicas.yaml

applications/
└── nextcloud/
    ├── values.yaml
    └── manifests/
        ├── nextcloud-longhorn-pvc.yaml
        └── nextcloud-volume-migration-pod.yaml
```

---

# 18. Commandes principales dans l’ordre chronologique

```bash
# Vérification Longhorn
kubectl get pods -n longhorn-system
kubectl get storageclass

# Création StorageClass 2 réplicas
kubectl apply -f infrastructure/longhorn/manifests/storageclass-2replicas.yaml

# Création PVC Nextcloud Longhorn
kubectl apply -f applications/nextcloud/manifests/nextcloud-longhorn-pvc.yaml

# Arrêt Nextcloud pour migration
kubectl scale deployment nextcloud --replicas=0 -n nextcloud

# Création du Pod de migration
kubectl apply -f applications/nextcloud/manifests/nextcloud-volume-migration-pod.yaml

# Copie
cp -a /source/. /destination/

# Suppression du Pod temporaire
kubectl delete pod -n nextcloud nextcloud-volume-migration

# Mise à jour Helm avec le PVC Longhorn
helm upgrade nextcloud nextcloud/nextcloud \
  --version 9.2.5 \
  -n nextcloud \
  -f applications/nextcloud/values.yaml

# Création du marqueur de réinstallation
kubectl exec -n nextcloud <POD> -- touch /var/www/html/config/CAN_INSTALL

# Recréation du Pod
kubectl delete pod -n nextcloud <POD>

# Vérification Services
kubectl get svc -n nextcloud

# Vérification Ingress
kubectl describe ingress -n nextcloud

# Vérification HTTP
curl -I http://nextcloud.home.arpa

# Vérification configuration active
kubectl exec -n nextcloud deploy/nextcloud -- \
  su -s /bin/sh www-data -c \
  'php /var/www/html/occ config:system:get overwriteprotocol'

# Suppression du forçage de protocole
kubectl exec -n nextcloud deploy/nextcloud -- \
  su -s /bin/sh www-data -c \
  'php /var/www/html/occ config:system:delete overwriteprotocol'

# Vérification trusted_domains
kubectl exec -n nextcloud deploy/nextcloud -- \
  su -s /bin/sh www-data -c \
  'php /var/www/html/occ config:system:get trusted_domains'

# Ajout du domaine Tailscale
kubectl exec -n nextcloud deploy/nextcloud -- \
  su -s /bin/sh www-data -c \
  'php /var/www/html/occ config:system:set trusted_domains 1 --value=nextcloud.taildf9a0f.ts.net'

# Test final Tailscale
curl -vkI https://nextcloud.taildf9a0f.ts.net/login
```

---

# 19. Points à compléter pour obtenir une trace 100 % exacte

Les éléments suivants ne sont pas visibles intégralement dans l’historique de conversation :

1. la commande exacte d’installation initiale de Longhorn ;
2. le `values.yaml` Helm Longhorn éventuellement utilisé ;
3. l’identifiant exact du schematic Talos Factory incluant `iscsi-tools` ;
4. le YAML exact du Pod de migration ;
5. le YAML exact de la StorageClass s’il contient des paramètres supplémentaires ;
6. les commandes SQL exactes et les `GRANT` utilisés pendant la réinitialisation ;
7. le manifeste exact de l’Ingress Tailscale.

Pour finaliser une documentation strictement identique au dépôt, il faudra remplacer les blocs marqués **reconstruits** par le contenu actuel de ces fichiers.

---

# 20. Recommandations suivantes

- migrer aussi le PVC MariaDB de `local-path` vers Longhorn ;
- activer des sauvegardes Longhorn récurrentes ;
- définir une destination de backup externe ;
- renouveler le mot de passe Redis exposé ;
- ajouter TLS à l’Ingress local avec cert-manager ou utiliser exclusivement Tailscale ;
- corriger les avertissements Pod Security du chart Nextcloud ;
- versionner et committer tous les manifests ;
- documenter les procédures de restauration.
