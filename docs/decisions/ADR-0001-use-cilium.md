# ADR-0001 — Utiliser Cilium comme CNI

## Statut

Accepté

## Contexte

Le cluster Kubernetes Talos utilise actuellement Flannel comme CNI.

Nous souhaitons disposer de fonctionnalités réseau plus avancées, notamment
les Network Policies et l'observabilité avec Hubble.

## Décision

Remplacer Flannel par Cilium.

kube-proxy sera conservé pendant cette première migration afin de limiter
le nombre de changements effectués simultanément.

## Conséquences

- Flannel sera désactivé dans la configuration Talos.
- Cilium sera installé comme CNI Kubernetes.
- kube-proxy restera actif.
- Une interruption temporaire du réseau des Pods est attendue pendant la migration.
- Hubble pourra être activé après validation du fonctionnement de Cilium.
