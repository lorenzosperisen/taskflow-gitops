## Jour 3 — Lab du matin : rollback automatique (2.0.0 → 2.1.0 → abort → 2.2.0)

**Date :** 15 octobre 2026  
**Environnement :** WSL2 Ubuntu, cluster kind `kind-cicd`, Argo CD, Argo Rollouts  
**Application :** `taskflow` · **Namespace :** `taskflow` · **Dossier surveillé :** `apps/taskflow`  
**Postmortem :** [docs/postmortem-2.1.0.md](docs/postmortem-2.1.0.md) · **Preuves :** [docs/preuves/](docs/preuves/)

### Objectif

Brancher un test de charge comme **porte de qualité pendant le canary**, puis livrer une version défectueuse (2.1.0) **sans intervenir**, pour vérifier que la chaîne l'annule seule. Ensuite : prouver la cause racine, remettre Git en accord avec la réalité par un revert, livrer le correctif (2.2.0) et rédiger le postmortem.

---

### Partie A — L'étalon et l'analyse

#### A0. Mise en route de mon cluster

Chaque membre du trinôme a son propre cluster local. Sur le mien, `kubectl get application -n argocd` ne renvoyait rien : Argo CD était installé (pods `Running`, CRD présente), mais **aucune Application ne lui indiquait quel dépôt surveiller**. La veille, elle avait été créée sur un autre cluster, et le seul point commun entre nos clusters est le dépôt Git.

```bash
sed -i 's/VOTRE-PSEUDO/lorenzosperisen/' argocd/application.yaml
kubectl apply -f argocd/application.yaml
```

**Observé :** `Synced / Progressing`, puis `Synced / Healthy` en quelques secondes.  
**Explication :** ce `kubectl apply` ne modifie pas la production, il amorce la boucle GitOps. À partir d'un cluster vide, Argo CD a recréé seul le namespace et toutes les ressources décrites dans Git.

**Incident technique :** `./scripts/charge.sh` échouait avec `env: $'bash\r': No such file or directory`. Le dépôt avait aussi été manipulé avec Git Bash, qui avait converti les scripts au format Windows (CRLF). Sous WSL, la première ligne du script était lue `bash\r`.

**Correction :** `git config core.autocrlf input`, puis `sed -i 's/\r$//' scripts/*.sh`. Cette correction est propre à mon poste et n'est pas committée.

#### A1–A4. Production en 2.0.0, analyse branchée

```bash
kubectl argo rollouts get rollout taskflow -n taskflow
kubectl get analysistemplate,configmap,svc -n taskflow
```

**Observé :**
- Rollout en stratégie **Canary**, image **2.0.0 (stable)**, 4 pods sur 4 Ready, **Healthy**.
- Aucun Deployment.
- Objets présents : `analysistemplate/robustesse-k6`, `configmap/k6-robustesse`, et les services `taskflow` et `taskflow-canary`. Ils ont été ajoutés par la **PR #9** `feat/analyse-auto`, mergée à 10:48:47.

| Étape | Contenu | Rôle |
|---|---|---|
| 1 | `setWeight: 25` | 1 pod sur 4 en nouvelle version |
| 2 | `pause: 20s` | Laisse le pod canary démarrer |
| 3 | `analysis` → `robustesse-k6` | **Porte de qualité** : Job k6 contre `taskflow-canary` |
| 4–7 | 50 % → pause → 75 % → pause | Seulement si l'analyse réussit |

**Explication :**
- L'analyse est une **étape bloquante**. Si k6 respecte les seuils (`http_req_failed < 2 %`, `p(95) < 250 ms`), le Rollout continue seul jusqu'à 100 %. Sinon, k6 sort en erreur, le Job échoue, l'AnalysisRun passe en `Failed` et Argo Rollouts fait un **abort automatique**.
- k6 vise `taskflow-canary`, qui ne contient que les pods de la nouvelle version. Via le service principal, 75 % des requêtes iraient vers la stable et diluraient les erreurs de la nouvelle version.

#### A2. L'étalon de la 2.0.0

```bash
./scripts/charge.sh http://taskflow | tee docs/preuves/etalon-2.0.0.txt
```

| Mesure (k6, 5 VUs, 30 s) | 2.0.0 | Seuil |
|---|---|---|
| Requêtes | 671 | — |
| `http_req_failed` | **0,00 %** ✓ | < 2 % |
| `p(95)` | **33,54 ms** ✓ | < 250 ms |
| Moyenne / médiane / max | 21,34 / 10,96 / 789,35 ms | — |

**Explication :**
- L'étalon prouve que les seuils sont réalistes : la version stable les respecte avec une marge d'environ ×7 sur le p95. Si la 2.1.0 échoue, ce sera dû à la version, pas à un seuil trop strict.
- On regarde le **p95** plutôt que le max ou la moyenne. Le max (789 ms) est une requête isolée de démarrage, et un seuil sur le max échouerait pour une seule requête. La moyenne, elle, masquerait une lenteur touchant une partie des requêtes.

---

### Partie B — L'incident

#### B1. Livraison de la 2.1.0 sans intervention

PR #10 `feat/taskflow-2.1.0` : une seule ligne modifiée dans `apps/taskflow/rollout.yaml`.

```diff
-          image: ghcr.io/9m7fjfpv9k-cyber/taskflow:2.0.0
+          image: ghcr.io/9m7fjfpv9k-cyber/taskflow:2.1.0
```

Merge à **12:18:47**, puis suivi avec `kubectl argo rollouts get rollout taskflow -n taskflow --watch`, **sans aucune commande**.

**Observé :**
- 12:19:34 : Argo CD synchronise. 1 pod stable est remplacé par 1 pod 2.1.0, soit 3 pods en 2.0.0 et 1 en 2.1.0.
- 12:19:45 : le pod 2.1.0 est **Ready**, palier 25 %.
- 12:20:05 : l'AnalysisRun et le Job k6 démarrent.
- 12:20:42 : l'AnalysisRun passe en **Failed** et le Rollout en **Degraded** (abort). Le canary est réduit à 0 et la stable remontée à 4.
- Argo CD affiche **Synced + Degraded**. Git demande toujours la 2.1.0.

![Canary 2.1.0 annulé automatiquement](images/canary-2.1.0-annule.png)

**Explication :**
- **La chaîne a décidé seule.** Aucun humain n'a agi entre le merge et l'abort (1 min 55 s).
- Le pod était Ready (probes vertes), mais k6 mesurait le comportement réel de la nouvelle version.
- Argo CD reste Synced parce que l'abort agit sur le cluster et ne modifie pas Git.

#### B2. Enquête : les 4 preuves

Les événements Kubernetes ne sont conservés qu'environ 1 h. Les preuves ont donc été collectées immédiatement et enregistrées dans `docs/preuves/`.

| # | Question | Commande | Ce qu'elle établit |
|---|---|---|---|
| 1 | Quoi ? | `kubectl argo rollouts get rollout taskflow -n taskflow` | Révision 2 (2.1.0) `ScaledDown`, Rollout `Degraded`, 2.0.0 stable |
| 2 | Quelle règle ? | `kubectl -n taskflow describe analysisrun taskflow-df976ccb5-2-2` | `Metric "test-de-charge-k6" assessed Failed due to failed (1) > failureLimit (0)` |
| 3 | Combien ? | `kubectl -n taskflow logs job/<job-k6>` | `http_req_failed` = **28,47 %** ✗ · `p(95)` = **328,87 ms** ✗ |
| 4 | Quand ? | `kubectl -n taskflow get events --sort-by=.lastTimestamp -o custom-columns=…` | Chronologie à la seconde, `RolloutAborted` à 10:20:42Z (12:20:42 à Paris) |

![k6 2.1.0 : seuils franchis](images/k6-2.1.0-echec.png)

**Explication :**
- **Deux défauts** : environ 28 % des réponses ne sont pas des 200, et chaque requête prend environ 300 ms de plus. Le minimum passe de 3 ms à 298 ms, donc la lenteur est systématique et non occasionnelle.
- **Pourquoi les probes étaient vertes :** les événements montrent que la readiness probe interroge `GET /health`. Cette route répond 200 dès que le processus tourne. Elle ne mesure ni le taux d'erreur ni la latence des routes métier.

#### B3. Revert de la PR #10

Bouton **Revert** sur la PR #10, ce qui crée la **PR #11**, relue puis mergée à **13:05:04**.

**Observé :**
- Rollout **Healthy**, image 2.0.0.
- **Aucun nouveau pod** : c'est le même ReplicaSet `c6cf57bd6`, renuméroté en révision 3. Les pods ont gardé leur âge.
- La révision 2 reste visible, avec son AnalysisRun et son Job en échec.

![Après le revert : Healthy, mêmes pods](images/revert-healthy.png)

**Explication :**
- L'abort protège les utilisateurs, et **le revert remet la vérité dans Git**. Sans lui, Git décrirait une version défectueuse comme état voulu, et une relance pourrait la redéployer.
- Aucun pod n'est recréé parce que le template demandé a le même hash que la version stable qui tourne déjà : il n'y a rien à déployer.
- Le commit de la 2.1.0 n'est pas effacé. Un nouveau commit l'annule, ce qui garde l'historique complet.

#### B4. Livraison du correctif 2.2.0

PR #12 `feat/taskflow-2.2.0` (image `2.0.0` → `2.2.0`), mergée à **13:20**, puis aucune intervention.

**Observé :**
- Palier 25 %, puis l'AnalysisRun `taskflow-7ddd57d788-4-2` passe en **Successful**.
- Le Rollout enchaîne **seul** 50 %, 75 % et 100 % (≈ 2 min 20 s du premier pod canary au 100 %).
- Résultat final : **2.2.0 stable**, 4 pods, **Healthy**.

![Analyse 2.2.0 réussie, progression automatique](images/analysisrun-2.2.0-successful.png)

![Canary 2.2.0 promu à 100 %](images/canary-2.2.0-promu.png)

**Explication :** c'est la contre-épreuve. La même porte de qualité, avec les mêmes seuils, laisse passer une version saine sans aucun `promote`. Le mécanisme mesure, il ne bloque pas aveuglément.

#### B5. Postmortem

Rédigé sans reproche dans [docs/postmortem-2.1.0.md](docs/postmortem-2.1.0.md) : résumé, chronologie horodatée, cause racine avec preuves, actions menées et actions d'amélioration.

---

### Journal des déploiements

| Heure | PR | Version | Action | Observé | Explication |
|---|---|---|---|---|---|
| 10:48:47 | #9 | 2.0.0 | Ajout de l'analyse k6 au canary | Rollout canary, AnalysisTemplate, ConfigMap, `taskflow-canary` | Porte de qualité branchée après le palier 25 % |
| 11:16 | — | 2.0.0 | Création de l'Application sur mon cluster | Synced / Healthy | Amorçage de la boucle GitOps |
| 11:38 | — | 2.0.0 | Étalon `charge.sh` | 0,00 % d'erreurs, p95 = 33,54 ms ✓ | Référence du comportement normal |
| 12:18:47 | #10 | 2.1.0 | Merge, aucune intervention | Pod Ready, AnalysisRun Failed, abort à 12:20:42, Synced / Degraded | Probes vertes, mais k6 mesure 28,47 % d'erreurs et un p95 de 328,87 ms |
| 13:05:04 | #11 | 2.0.0 | Revert de la PR #10 | Healthy, aucun nouveau pod | Git remis en accord avec le cluster, de façon tracée |
| 13:20 | #12 | 2.2.0 | Merge, aucune intervention | AnalysisRun Successful, 25 → 50 → 75 → 100 % automatique | Version saine : promotion sans humain |

### Comparaison des trois versions

| Mesure (k6, 5 VUs, 30 s) | Étalon 2.0.0 | Incident 2.1.0 | Correctif 2.2.0 | Seuil |
|---|---|---|---|---|
| `http_req_failed` | 0,00 % ✓ | **28,47 %** ✗ | 0,00 % ✓ | < 2 % |
| `p(95)` | 33,54 ms ✓ | **328,87 ms** ✗ | 34,02 ms ✓ | < 250 ms |
| Minimum | 3,04 ms | 298,32 ms | 2,64 ms | — |
| Requêtes | 671 | 295 | 701 | — |
| Décision | — | **Abort automatique** | **Promotion automatique** | — |

### Ce que nous retenons

1. **Un test de charge avec des seuils peut décider seul d'annuler une livraison.** La 2.1.0 a été arrêtée en 37 s d'analyse, après environ 57 s d'exposition sur 25 % du trafic.
2. **L'abort protège les utilisateurs, le revert remet la vérité dans Git.** Les deux sont nécessaires : l'un est automatique, l'autre humain et tracé par PR.
3. **Les probes ne voient pas ce que vivent les utilisateurs.** Un pod peut être Ready tout en échouant une requête sur quatre. Seule une mesure du comportement réel sous charge le révèle.
4. **On mesure la nouvelle version seule.** Le service `taskflow-canary` évite que les erreurs soient diluées par le trafic servi par la version stable.
