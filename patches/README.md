# Patches appliqués aux sources VROOM

Le `Dockerfile` clone VROOM (`VROOM_RELEASE`, ex. `v1.15.0`) puis applique, dans l'ordre,
chaque fichier `patches/*.patch` avec `git apply`. Si un patch ne s'applique plus
(mise à jour de VROOM), le build échoue volontairement : relire la section du patch
concerné ci-dessous pour savoir s'il est encore nécessaire.

Patch « historique » non présent ici : `MAX_PRIORITY` passé de 100 à 10000 par `sed`
directement dans le `Dockerfile` (`src/structures/typedefs.h`).

---

## 0001-break-pickup-delivery-order-choice.patch

### Symptôme

Un *shipment* (pickup + delivery) dont la livraison a un créneau **tôt** est laissé dans
`unassigned`, alors que le véhicule a une **pause plus tard dans la journée** et qu'une
tournée valide existe (pickup → livraison → … → pause). Avec la même livraison déclarée
en *job* simple, VROOM la planifie sans problème. La priorité n'a aucun effet.

Cas réel (nov. 2026) : livraison obligatoire avec un créneau 05:00–10:00 à environ 80 min de l'atelier,
véhicule 08:00–18:45, pause de 45 min entre 13:00 et 14:30 → commande jamais planifiée.
Si on supprime la pause, ou si on élargit le créneau jusqu'à 13:45, elle passe.

Règle observée empiriquement avant correction : le shipment n'est accepté que si
`début véhicule + durée de la pause + trajet ≤ fin du créneau de livraison`,
autrement dit seulement si l'ordre « pause → pickup → livraison » tient.

### Cause (VROOM v1.15.0, toujours présent sur master au 17/11/2026, commit 877c2dd)

`src/structures/vroom/tw_route.cpp`, `TWRoute::is_valid_addition_for_tw` parcourt les
tâches insérées et, quand une pause est en attente, appelle `order_choice` pour décider
localement « tâche d'abord » ou « pause d'abord ».

1. Quand la tâche est un **pickup** suivi de sa livraison dans la plage insérée,
   `tighter_next` est construit à partir de la **livraison** :
   `tighter_next.latest = min(next.latest - next.travel, next_j.tws.back().end)`,
   c'est-à-dire la fin du créneau de livraison (10:00 dans le cas réel).
2. Dans `order_choice`, le test « job then break » :
   ```cpp
   if (job_then_break_end + travel_after_break > next.latest) {
     // Starting the break is possible but then next step is not.
     oc.add_break_first = true;
     return oc;
   }
   ```
   vérifie en fait *pickup → pause → livraison* (fin de pause à 13:45, ce qui dépasse 10:00).
   Comme c'est impossible, la fonction renvoie immédiatement « pause d'abord ».
3. Ce `return` court-circuite le bloc dédié aux pickups plus bas
   (« For a pickup, we favor putting the pickup first… »), qui teste justement
   *pickup → livraison → pause*. Son commentaire décrit exactement ce piège
   (« avoid heuristically forcing a … choice resulting in invalid options »).
4. Avec « pause d'abord », on obtient pause à 13:00–13:45, puis pickup, puis livraison vers 15:10,
   hors créneau. `is_valid_addition_for_tw` renvoie `false` pour toutes les positions,
   et le shipment reste non assigné.

Un *job* simple n'est pas touché : il n'y a pas de livraison « suivante » qui resserre `next.latest`.

### Correctif

Dans la branche ci-dessus, si la tâche est un `PICKUP`, on teste d'abord
*pickup → livraison → pause* (même calcul que le bloc pickup existant). Si cet ordre tient,
on choisit `add_job_first`. Sinon on garde le comportement d'origine (`add_break_first`).
La suite de `is_valid_addition_for_tw` revérifie ensuite normalement la livraison et la pause,
le patch ne fait donc qu'ouvrir une option que VROOM écartait à tort.

### Protocole de reproduction (anonymisé, sans OSRM)

Les fichiers se trouvent dans `tests/break_pickup_delivery/`. Ils utilisent une matrice de durées fixe
avec deux lieux : 0 = dépôt, 1 = client à 40 min. Les heures sont en secondes depuis le début de journée.

| Élément  | Valeur |
|----------|--------|
| Véhicule | `time_window` [0, 36000] (10 h), départ et retour au lieu 0 |
| Pause    | 45 min (`service` 2700), fenêtre [18000, 23400] (de +5 h à +6 h 30) |
| Shipment | pickup au lieu 0, livraison au lieu 1, `service` 600, créneau [0, 7200] (avant +2 h) |

```bash
# binaire vroom directement, pas besoin de routeur grâce à "matrices"
IMG=ghcr.io/vroom-project/vroom-docker:v1.15.0   # ou l'image construite depuis ce repo
D=$PWD/tests/break_pickup_delivery

# 1. Bug : le shipment est non assigné (attendu sans patch)
docker run --rm -v $D:/d --entrypoint vroom $IMG -i /d/break_pickup_delivery.json

# 2. Témoin : même livraison en job simple → assignée
docker run --rm -v $D:/d --entrypoint vroom $IMG -i /d/control_job.json

# 3. Preuve de faisabilité : tournée imposée pickup → livraison → pause, mode -c
#    → aucune violation (pause à 10200 + 7800 s d'attente = 18000)
docker run --rm -v $D:/d --entrypoint vroom $IMG -c -i /d/plan_check.json
```

Résultats attendus :

| Fichier | v1.15.0 officielle | Image patchée |
|---------|--------------------|---------------|
| `break_pickup_delivery.json` | `unassigned` : pickup 1 et delivery 1 | `unassigned` : `[]` |
| `control_job.json` | `[]` | `[]` |
| `plan_check.json` (`-c`) | aucune violation | aucune violation |

### Lors d'une mise à jour de VROOM

1. Lancer le test 1 sur l'image **officielle** de la nouvelle version. Si le shipment est
   assigné, le bug est corrigé en amont : supprimer le patch.
2. Sinon, vérifier que `git apply` passe encore. Si ce n'est pas le cas, rechercher dans
   `tw_route.cpp` le commentaire « Starting the break is possible but then next step is not »
   et réappliquer la même logique à la main.
3. Reconstruire et vérifier que le test 1 rend `unassigned: []`.
