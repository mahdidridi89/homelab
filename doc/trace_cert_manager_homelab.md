# Trace d’exécution détaillée — cert-manager dans le homelab

## 0. Objectif

Cette phase a pour but d’ajouter une PKI interne au cluster Kubernetes afin de pouvoir émettre et renouveler automatiquement des certificats TLS pour les services locaux comme :

- `nextcloud.home.arpa`
- `grafana.home.arpa`
- `gitea.home.arpa`

Architecture mise en place :

```text
SelfSigned Issuer
       │
       ▼
Homelab Root CA
       │
       ▼
ClusterIssuer homelab-ca
       │
       ├── nextcloud.home.arpa
       ├── grafana.home.arpa
       └── gitea.home.arpa
```

Le `SelfSigned Issuer` sert uniquement au bootstrap. La vraie autorité utilisée ensuite par les applications est la Root CA exposée via le `ClusterIssuer homelab-ca`.

---

# 1. Vérifier qu’aucun cert-manager n’existait déjà

Commande :

```bash
helm list -A
```

But :

- vérifier qu’aucune release Helm `cert-manager` n’était déjà installée ;
- éviter une double installation de composants cluster-wide.

Résultat observé :

```text
cilium
gitea
longhorn
metrics-server
monitoring
nextcloud
tailscale-operator
```

Aucune release `cert-manager`.

Ensuite :

```bash
kubectl get crd | grep cert-manager
```

Retour vide.

Conclusion :

- aucune release Helm cert-manager ;
- aucune CRD cert-manager restante ;
- installation possible depuis un état propre.

---

# 2. Créer le namespace `cert-manager`

Commande :

```bash
kubectl create namespace cert-manager
```

Pourquoi :

- isoler les contrôleurs cert-manager ;
- centraliser les Secrets et objets de bootstrap PKI ;
- faciliter l’administration.

Impact :

- création d’un namespace uniquement ;
- aucune application existante n’est touchée.

---

# 3. Installer cert-manager avec Helm

Commande utilisée :

```bash
helm install cert-manager   oci://quay.io/jetstack/charts/cert-manager   --version v1.21.1   --namespace cert-manager   --set crds.enabled=true
```

Explication des éléments :

### `helm install`

Crée une nouvelle release Helm.

### `cert-manager`

Nom de la release.

### `oci://quay.io/jetstack/charts/cert-manager`

Source du chart sous forme OCI.

### `--version v1.21.1`

Verrouille explicitement la version installée.

C’est important pour la reproductibilité : un futur déploiement ne prendra pas automatiquement une autre version.

### `--namespace cert-manager`

Installe les Pods et Services principaux dans le namespace `cert-manager`.

### `--set crds.enabled=true`

Demande au chart d’installer les CRDs nécessaires à cert-manager.

Une CRD est une `CustomResourceDefinition`, c’est-à-dire une extension de l’API Kubernetes.

---

# 4. Vérifier les Pods cert-manager

Commande :

```bash
kubectl get pods -n cert-manager -o wide
```

Résultat observé :

```text
cert-manager-689c4c5575-std44              1/1 Running
cert-manager-cainjector-6fbb9c8cd6-xhwrp   1/1 Running
cert-manager-webhook-646c95c5ff-ctwnp      1/1 Running
```

Rôle des composants :

## `cert-manager`

Contrôleur principal.

Il surveille les ressources :

```text
Certificate
CertificateRequest
Issuer
ClusterIssuer
```

## `cert-manager-cainjector`

Injecte des bundles CA dans certaines ressources Kubernetes.

## `cert-manager-webhook`

Valide les ressources cert-manager et participe à leur admission dans l’API Kubernetes.

---

# 5. Vérifier les CRDs

Commande :

```bash
kubectl get crd | grep cert-manager
```

Résultat observé :

```text
certificaterequests.cert-manager.io
certificates.cert-manager.io
challenges.acme.cert-manager.io
clusterissuers.cert-manager.io
issuers.cert-manager.io
orders.acme.cert-manager.io
```

Signification :

- `Certificate` : demande de certificat ;
- `CertificateRequest` : demande de signature générée par cert-manager ;
- `Issuer` : autorité limitée à un namespace ;
- `ClusterIssuer` : autorité utilisable dans tout le cluster ;
- `Challenge` et `Order` : objets surtout utilisés avec ACME/Let’s Encrypt.

---

# 6. Créer l’arborescence Git

Commande :

```bash
mkdir -p infrastructure/cert-manager/manifests
```

But :

versionner les manifests de la PKI dans le dépôt du homelab.

Arborescence :

```text
infrastructure/
└── cert-manager/
    └── manifests/
```

---

# 7. Créer le SelfSigned Issuer de bootstrap

Fichier :

```text
infrastructure/cert-manager/manifests/selfsigned-issuer.yaml
```

Commande de création :

```bash
cat > infrastructure/cert-manager/manifests/selfsigned-issuer.yaml <<'EOF'
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: homelab-selfsigned
  namespace: cert-manager
spec:
  selfSigned: {}
EOF
```

YAML :

```yaml
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: homelab-selfsigned
  namespace: cert-manager
spec:
  selfSigned: {}
```

Explication de chaque champ :

### `apiVersion: cert-manager.io/v1`

API ajoutée par les CRDs cert-manager.

### `kind: Issuer`

Crée une autorité de signature limitée à un namespace.

### `metadata.name: homelab-selfsigned`

Nom logique de cet Issuer.

### `metadata.namespace: cert-manager`

L’Issuer existe uniquement dans le namespace `cert-manager`.

### `spec.selfSigned: {}`

Indique que cet Issuer utilise le mode auto-signé.

Cet Issuer n’est pas destiné à signer directement tous les certificats du homelab ; il sert uniquement à créer la Root CA.

---

# 8. Appliquer le SelfSigned Issuer

Commande :

```bash
kubectl apply -f infrastructure/cert-manager/manifests/selfsigned-issuer.yaml
```

Vérification :

```bash
kubectl get issuers.cert-manager.io -n cert-manager
```

Résultat :

```text
NAME                 READY
homelab-selfsigned   True
```

`READY=True` signifie que cert-manager peut utiliser cet Issuer.

---

# 9. Créer le certificat Root CA

Fichier :

```text
infrastructure/cert-manager/manifests/root-ca-certificate.yaml
```

YAML :

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: homelab-root-ca
  namespace: cert-manager
spec:
  isCA: true
  commonName: Homelab Root CA
  secretName: homelab-root-ca
  duration: 87600h
  renewBefore: 8760h
  privateKey:
    algorithm: RSA
    size: 4096
  issuerRef:
    name: homelab-selfsigned
    kind: Issuer
    group: cert-manager.io
```

Explication détaillée :

### `kind: Certificate`

Demande à cert-manager de générer un certificat.

### `metadata.name: homelab-root-ca`

Nom Kubernetes de la ressource `Certificate`.

### `metadata.namespace: cert-manager`

Le certificat racine est géré dans `cert-manager`.

### `isCA: true`

Indique que le certificat peut signer d’autres certificats.

C’est le champ qui en fait une autorité de certification.

### `commonName: Homelab Root CA`

Nom lisible de la CA.

### `secretName: homelab-root-ca`

Nom du Secret dans lequel cert-manager stocke :

```text
tls.crt
tls.key
ca.crt
```

### `duration: 87600h`

Environ 10 ans.

Une Root CA a généralement une durée de vie longue.

### `renewBefore: 8760h`

Environ 1 an avant expiration.

cert-manager peut renouveler la CA avant qu’elle expire.

### `privateKey.algorithm: RSA`

Utilisation d’une clé RSA.

### `privateKey.size: 4096`

Clé RSA 4096 bits.

C’est adapté pour une autorité racine.

### `issuerRef.name: homelab-selfsigned`

La Root CA est bootstrapée via l’Issuer auto-signé.

### `issuerRef.kind: Issuer`

Référence un `Issuer` namespaced.

### `issuerRef.group: cert-manager.io`

Groupe API de cert-manager.

---

# 10. Appliquer la Root CA

Commande :

```bash
kubectl apply -f infrastructure/cert-manager/manifests/root-ca-certificate.yaml
```

cert-manager exécute alors :

```text
Certificate
   │
   ▼
CertificateRequest
   │
   ▼
génération clé privée
   │
   ▼
signature
   │
   ▼
Secret homelab-root-ca
```

---

# 11. Vérifier le Secret de la Root CA

Commande observée :

```bash
kubectl get secrets -n cert-manager
```

Résultat :

```text
homelab-root-ca   kubernetes.io/tls
```

Attention :

le Secret contient la clé privée de la Root CA.

Il ne faut pas afficher ou partager :

```text
tls.key
```

Une commande comme :

```bash
kubectl get secret homelab-root-ca -n cert-manager -o yaml
```

afficherait la clé en base64.

Base64 n’est pas du chiffrement.

---

# 12. Différence avec `cert-manager-webhook-ca`

Le namespace contient aussi :

```text
cert-manager-webhook-ca
```

Cette CA appartient au webhook interne de cert-manager.

Elle ne doit pas être confondue avec :

```text
homelab-root-ca
```

Schéma :

```text
cert-manager-webhook-ca
    └── protège le webhook cert-manager

homelab-root-ca
    └── signe les certificats du homelab
```

---

# 13. Vérifier la Root CA

Commande :

```bash
kubectl get certificate homelab-root-ca -n cert-manager -o wide
```

Résultat :

```text
NAME              READY   SECRET            ISSUER
homelab-root-ca   True    homelab-root-ca   homelab-selfsigned
```

Statut :

```text
Certificate is up to date and has not expired
```

Conclusion :

la Root CA est valide.

---

# 14. Créer le ClusterIssuer

Pourquoi un `ClusterIssuer` ?

Un `Issuer` est limité à un namespace.

Mais on veut signer des certificats pour :

```text
nextcloud
monitoring
gitea
longhorn-system
...
```

On crée donc une autorité globale.

Fichier :

```text
infrastructure/cert-manager/manifests/ca-clusterissuer.yaml
```

YAML :

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: homelab-ca
spec:
  ca:
    secretName: homelab-root-ca
```

Explication :

### `kind: ClusterIssuer`

Autorité accessible depuis tous les namespaces.

### `metadata.name: homelab-ca`

Nom utilisé par les certificats applicatifs.

### `spec.ca`

Indique que cet Issuer s’appuie sur une CA existante.

### `secretName: homelab-root-ca`

Le ClusterIssuer utilise le certificat et la clé privée de la Root CA stockés dans ce Secret.

---

# 15. Appliquer le ClusterIssuer

Commande :

```bash
kubectl apply -f infrastructure/cert-manager/manifests/ca-clusterissuer.yaml
```

Vérification :

```bash
kubectl get clusterissuers.cert-manager.io -o wide
```

Résultat :

```text
NAME         READY   STATUS
homelab-ca   True    Signing CA verified
```

Cela signifie que cert-manager a vérifié la CA et peut l’utiliser pour signer des certificats.

---

# 16. Créer le premier certificat applicatif : Nextcloud

Fichier :

```text
applications/nextcloud/manifests/nextcloud-certificate.yaml
```

YAML :

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: nextcloud-home-arpa
  namespace: nextcloud
spec:
  secretName: nextcloud-home-arpa-tls
  duration: 8760h
  renewBefore: 720h
  privateKey:
    algorithm: RSA
    size: 2048
  dnsNames:
    - nextcloud.home.arpa
  usages:
    - server auth
    - digital signature
    - key encipherment
  issuerRef:
    name: homelab-ca
    kind: ClusterIssuer
    group: cert-manager.io
```

Explication détaillée :

### `metadata.name: nextcloud-home-arpa`

Nom de la ressource Certificate.

### `namespace: nextcloud`

Le certificat est créé dans le namespace Nextcloud.

### `secretName: nextcloud-home-arpa-tls`

Secret final contenant le certificat et la clé.

L’Ingress Cilium utilisera ce Secret.

### `duration: 8760h`

Environ 1 an.

### `renewBefore: 720h`

Environ 30 jours avant expiration.

cert-manager renouvellera automatiquement le certificat.

### `privateKey.algorithm: RSA`

Clé RSA.

### `privateKey.size: 2048`

2048 bits, adapté à un certificat serveur.

### `dnsNames`

```yaml
dnsNames:
  - nextcloud.home.arpa
```

Le certificat est valide pour ce nom DNS.

Les navigateurs vérifient le SAN (`Subject Alternative Name`) lors de la connexion TLS.

### `usages: server auth`

Autorise l’utilisation comme certificat serveur.

### `usages: digital signature`

Autorise les opérations de signature numérique.

### `usages: key encipherment`

Usage classique lié à RSA/TLS.

### `issuerRef.name: homelab-ca`

Le certificat est signé par notre ClusterIssuer.

### `issuerRef.kind: ClusterIssuer`

Référence une autorité cluster-wide.

### `issuerRef.group: cert-manager.io`

Groupe API cert-manager.

---

# 17. Appliquer le certificat Nextcloud

Commande :

```bash
kubectl apply -f applications/nextcloud/manifests/nextcloud-certificate.yaml
```

Workflow :

```text
Certificate nextcloud-home-arpa
          │
          ▼
CertificateRequest
          │
          ▼
ClusterIssuer homelab-ca
          │
          ▼
Homelab Root CA
          │
          ▼
signature
          │
          ▼
Secret nextcloud-home-arpa-tls
```

---

# 18. Vérifier le certificat Nextcloud

Commande :

```bash
kubectl get certificate -n nextcloud -o wide
```

Résultat observé :

```text
NAME                  READY   SECRET                    ISSUER
nextcloud-home-arpa   True    nextcloud-home-arpa-tls   homelab-ca
```

Statut :

```text
Certificate is up to date and has not expired
```

Le certificat est donc prêt.

---

# 19. Arborescence Git obtenue

```text
infrastructure/
└── cert-manager/
    └── manifests/
        ├── selfsigned-issuer.yaml
        ├── root-ca-certificate.yaml
        └── ca-clusterissuer.yaml

applications/
└── nextcloud/
    └── manifests/
        └── nextcloud-certificate.yaml
```

---

# 20. Commandes exécutées dans l’ordre

```bash
helm list -A
```

```bash
kubectl get crd | grep cert-manager
```

```bash
kubectl create namespace cert-manager
```

```bash
helm install cert-manager   oci://quay.io/jetstack/charts/cert-manager   --version v1.21.1   --namespace cert-manager   --set crds.enabled=true
```

```bash
kubectl get pods -n cert-manager -o wide
```

```bash
kubectl get crd | grep cert-manager
```

```bash
mkdir -p infrastructure/cert-manager/manifests
```

```bash
kubectl apply -f infrastructure/cert-manager/manifests/selfsigned-issuer.yaml
```

```bash
kubectl get issuers.cert-manager.io -n cert-manager
```

```bash
kubectl apply -f infrastructure/cert-manager/manifests/root-ca-certificate.yaml
```

```bash
kubectl get certificate homelab-root-ca -n cert-manager -o wide
```

```bash
kubectl apply -f infrastructure/cert-manager/manifests/ca-clusterissuer.yaml
```

```bash
kubectl get clusterissuers.cert-manager.io -o wide
```

```bash
kubectl apply -f applications/nextcloud/manifests/nextcloud-certificate.yaml
```

```bash
kubectl get certificate -n nextcloud -o wide
```

---

# 21. État actuel

cert-manager :

```text
controller    Running
cainjector    Running
webhook       Running
```

Root CA :

```text
homelab-root-ca
READY=True
```

ClusterIssuer :

```text
homelab-ca
READY=True
STATUS=Signing CA verified
```

Certificat Nextcloud :

```text
nextcloud-home-arpa
READY=True
```

Secret TLS :

```text
nextcloud-home-arpa-tls
```

---

# 22. Étape suivante

Le certificat existe, mais l’Ingress Cilium ne l’utilise pas encore.

La prochaine modification sera dans le `values.yaml` Nextcloud.

Actuellement :

```yaml
ingress:
  enabled: true
  className: cilium
  path: /
  pathType: Prefix
  tls: []
```

Cible :

```yaml
ingress:
  enabled: true
  className: cilium
  path: /
  pathType: Prefix
  tls:
    - secretName: nextcloud-home-arpa-tls
      hosts:
        - nextcloud.home.arpa
```

Architecture finale :

```text
Navigateur
   │
   │ HTTPS
   ▼
Cilium Ingress
   │
   │ Secret TLS
   ▼
nextcloud-home-arpa-tls
   │
   ▼
Service nextcloud
   │
   ▼
Pod Nextcloud
```

---

# 23. Confiance de la Root CA sur les clients

Même lorsque Cilium utilisera correctement le certificat, le navigateur ne fera pas automatiquement confiance à notre CA privée.

Il faudra installer le certificat public de :

```text
Homelab Root CA
```

dans le magasin de confiance des appareils :

```text
Mac
iPhone
Windows
Linux
```

Après cela :

```text
https://nextcloud.home.arpa
```

pourra être affiché comme certificat valide par les clients.

---

# 24. Sécurité de la Root CA

Le Secret :

```text
homelab-root-ca
```

est extrêmement sensible.

Il contient la clé privée capable de signer tous les certificats du homelab.

Règles :

- ne jamais committer `tls.key` dans Git ;
- ne jamais partager le YAML complet du Secret ;
- sauvegarder la CA de manière sécurisée ;
- contrôler les droits RBAC sur le namespace `cert-manager`;
- prévoir une procédure de restauration de la CA.

---

# 25. Vision conceptuelle

cert-manager ajoute une couche PKI à Kubernetes :

```text
Certificate
     │
     ▼
cert-manager
     │
     ▼
Issuer / ClusterIssuer
     │
     ▼
CA
     │
     ▼
Secret TLS
     │
     ▼
Ingress Cilium
```

La responsabilité est bien séparée :

- Kubernetes héberge les objets et Secrets ;
- cert-manager gère le cycle de vie des certificats ;
- la CA signe ;
- Cilium termine TLS ;
- Nextcloud reste derrière un Service Kubernetes.

