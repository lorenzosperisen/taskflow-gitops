# Lab 2 — Déploiements Blue-Green et Canary

## Objectif

Découvrons deux stratégies de déploiement progressif avec **Kubernetes, Argo CD et Argo Rollouts**, en utilisant l'application TaskFlow.

## 1. Déploiement Blue-Green

Nous avons remplacé le Deployment Kubernetes par un Rollout utilisant la stratégie `blueGreen`
![Configuration Blue-Green](images/etape1.2_lab2.png)

- Version active : `1.0.0`
- Version preview : `1.1.0`
- Vérification de la nouvelle version avant sa mise en production
- Promotion manuelle de `1.1.0`

**Résultat :** la version `1.1.0` est devenue stable, avec 4 pods disponibles.

## 2. Déploiement Canary

Nous avons ensuite configuré la stratégie `canary` pour déployer progressivement la version `2.0.0`.

Progression observée : **25 % → 50 % → 75 % → 100 %**, avec des promotions manuelles.

Commande de suivi :

```bash
kubectl argo rollouts get rollout taskflow -n taskflow --watch
```

**Résultat:**
![Progression canary](images/canary75.png)
![Observation canary](images/observeCan.png)

Vérification avec `bash scripts/observe.sh` : **40 requêtes HTTP 200**, toutes sur la version `2.0.0`.

## 4. Bilan

Ce lab nous a permis de comprendre comment **Argo Rollouts contrôle les mises en production**, tandis qu'**Argo CD synchronise Kubernetes avec Git**.

Les deux stratégies ont fonctionné. La version finale est `2.0.0`, stable avec 4 réplicas.

Nous recommnaderons l'application de canary pour eviter un temps d'interruption chez le client ou bien un épuisement des ressources avec le deploiement avec Bluegreen .

## 5. Captures d'écran

À joindre : Blue-Green avant/après promotion, progression Canary, état final `Healthy` et résultat du script `observe.sh`.