# Workflow Import/Export — Commandes Drush

## Référence Complète des Commandes

```bash
# EXPORT — Active (DB) → Staged (YAML)
docker compose exec php drush config:export          # Alias: drush cex
docker compose exec php drush cex -y                 # Sans confirmation

# IMPORT — Staged (YAML) → Active (DB)
docker compose exec php drush config:import          # Alias: drush cim
docker compose exec php drush cim -y                 # Sans confirmation
docker compose exec php drush cim --partial          # ⚠️ Importer seulement les fichiers présents (dangereux en prod)

# STATUT — Comparer active vs staged
docker compose exec php drush config:status          # Alias: drush cst
# Résultat:
# "Only in active"  → en DB mais pas exporté (tu as des modifs non exportées)
# "Only in staging" → dans les fichiers mais pas importé (nouveau pour toi)
# "Different"       → existe des deux côtés mais contenu différent (conflit potentiel)

# LIRE une config
docker compose exec php drush config:get system.site              # Tout l'objet
docker compose exec php drush config:get system.site name        # Une clé spécifique
docker compose exec php drush config:get node.type.article       # Un Config Entity

# ÉCRIRE une valeur (dangereux — préférer le workflow cex/cim)
docker compose exec php drush config:set system.site name 'Mon site'
docker compose exec php drush config:set mon_module.settings max_items 25

# DIFF — Voir les différences ligne par ligne
docker compose exec php drush config:diff system.site
docker compose exec php drush config:diff views.view.frontpage

# ÉDITER — Ouvrir dans $EDITOR
docker compose exec php drush config:edit system.site

# SUPPRIMER une config entière
docker compose exec php drush config:delete mon_module.settings
docker compose exec php drush config:delete views.view.orphaned_view

# LISTER les configs
docker compose exec php drush config:list
docker compose exec php drush config:list --prefix=node.type     # Filtrer par préfixe
```

---

## `drush deploy` — Commande de Déploiement Complète (D9+)

```bash
# Exécute dans l'ordre correct : updatedb → cim → cr + hook_deploy_N après cim
docker compose exec php drush deploy -y
```

**Ordre d'exécution de `drush deploy` :**
1. `drush updatedb` — met à jour le schéma DB + exécute `hook_update_N`
2. `drush config:import` — importe la config YAML
3. `drush cache:rebuild` — vide les caches
4. *(automatiquement)* — `hook_deploy_N` s'exécute après l'étape 2

**Note :** `drush deploy` ne lance PAS `entity:updates` ni `locale:update` automatiquement. Si nécessaire, les ajouter manuellement dans le script de déploiement après `drush deploy`.

`drush deploy` garantit que `hook_deploy_N` s'exécute **après** `drush cim`, ce que `drush updb` seul ne fait pas correctement.

---

## Lire `drush config:status`

```bash
$ docker compose exec php drush config:status

 Name                          State            
 system.site                   Different        ← valeur différente entre DB et YAML
 views.view.frontpage          Only in active   ← en DB, pas encore exporté
 mon_module.settings           Only in staging  ← dans les fichiers, pas encore importé
```

**Règle :** avant tout `drush cim`, résoudre les "Only in active" avec un `drush cex` ou une décision consciente de les écraser.

---

## Workflow Git Complet — Développement Multi-développeurs

### Développeur A — Créer une fonctionnalité

```bash
# 1. Partir d'une branche propre
git checkout -b feature/ajout-content-type-projet

# 2. Travailler via l'UI Drupal (créer le Content Type, les champs, etc.)

# 3. Exporter la config
docker compose exec php drush cex -y

# 4. Vérifier ce qui a changé
git diff config/sync/

# 5. Committer UNIQUEMENT les fichiers config pertinents
git add config/sync/node.type.projet.yml
git add config/sync/field.field.node.projet.*.yml
git add config/sync/core.entity_form_display.node.projet.*.yml
git add config/sync/core.entity_view_display.node.projet.*.yml
git commit -m "feat(content): ajout Content Type Projet avec champs"
git push origin feature/ajout-content-type-projet
```

### Développeur B — Récupérer et appliquer

```bash
# 1. Vérifier son état avant
docker compose exec php drush cst                          # Doit être clean (no differences)
docker compose exec php drush cex -y                       # Exporter ses propres modifs si "Only in active"

# 2. Récupérer la branche
git pull origin feature/ajout-content-type-projet

# 3. Appliquer la config
docker compose exec php drush cim -y
docker compose exec php drush cr
```

### En Production — Script de déploiement

```bash
#!/bin/bash
set -e

# Aller dans le webroot
cd /var/www/html/web

# Mettre à jour les dépendances Composer
cd ..
composer install --no-dev --optimize-autoloader

# Appliquer les mises à jour
cd web

# Option 1 — drush deploy (D9+ recommandé)
../vendor/bin/drush deploy -y

# Option 2 — Manuel (D8 ou si drush deploy non disponible)
../vendor/bin/drush updb -y
../vendor/bin/drush cim -y
../vendor/bin/drush cr
../vendor/bin/drush locale:update  # Si multilingue
```

---

## Import Partiel — `drush cim --partial`

```bash
# Import partiel : n'importe QUE les fichiers présents dans le staging
# Ne supprime PAS les configs absentes du staging
docker compose exec php drush cim --partial --source=/chemin/vers/config-partielle/
```

**Quand l'utiliser :**
- Déployer uniquement un sous-ensemble de configs
- Appliquer des correctifs de config ciblés

**Risques :**
- Laisse de la config orpheline si on ne maîtrise pas ce qu'on exclut
- `--partial` ne supprime pas les configs obsolètes → accumulation de dette config
- Ne jamais utiliser en prod sans avoir testé l'impact complet

---

## Importer la Config d'un Module Spécifique

```bash
# Importer uniquement la config fournie par un module (depuis son config/install/)
docker compose exec php drush php:eval "\Drupal::service('config.installer')->installDefaultConfig('module', 'mon_module');"

# Importer un fichier YAML manuellement (depuis Drush)
docker compose exec php drush config:set --input-format=yaml NOM_CONFIG - < config/sync/mon_module.settings.yml
```

---

## Troubleshooting — Commandes de Diagnostic

```bash
# Voir les erreurs de validation de config
docker compose exec php drush config:status --format=json

# Vérifier la config d'un module spécifique
docker compose exec php drush config:list --prefix=mon_module

# Comparer deux environnements (depuis staging qui a accès aux deux)
docker compose exec php drush config:export --destination=/tmp/config-prod
diff -r config/sync/ /tmp/config-prod/

# Réinitialiser la config d'un module à ses valeurs par défaut
docker compose exec php drush pm:uninstall mon_module -y
docker compose exec php drush pm:enable mon_module -y

# Forcer l'UUID d'un site (résolution de conflit UUID)
docker compose exec php drush config:set system.site uuid "VOTRE-UUID-ICI"
```

---

## Performance des Imports Config — Grands Projets (1000+ fichiers YAML)

Certains projets accumulent 1200+ fichiers YAML de configuration (typiquement : Views avec beaucoup de displays, nombreux `entity_view_display`, champs répétés par bundle). L'import peut devenir lent (30-120 secondes).

### Diagnostiquer ce qui ralentit

```bash
# Compter le nombre total de fichiers YAML
ls config/sync/ | wc -l

# Les Views avec beaucoup de displays sont les plus lentes à importer
ls -la config/sync/views.view.*.yml | sort -k5 -rn | head -10

# Compter les entity_view_display (souvent nombreux sur les gros projets)
ls config/sync/ | grep "core.entity_view_display" | wc -l

# Mesurer le temps d'import réel
time docker compose exec php drush config:import -y
```

### Optimiser l'import

```bash
# Vider les caches AVANT l'import — évite les locks et conflits de cache
docker compose exec php drush cr
docker compose exec php drush config:import -y
docker compose exec php drush cr

# Import partiel (modules spécifiques) — éviter d'importer tout le projet
# Utile pendant le développement pour n'appliquer qu'un sous-ensemble
docker compose exec php drush config:import --partial --source=config/updates/ -y

# En production — augmenter la mémoire PHP pour les imports lourds
docker compose exec -e PHP_MEMORY_LIMIT=512M php drush deploy -y

# Vérifier l'utilisation mémoire pendant un import
docker compose exec php php -r "
  echo 'Mémoire disponible : ' . ini_get('memory_limit') . PHP_EOL;
  echo 'Mémoire actuelle : ' . round(memory_get_usage() / 1024 / 1024, 2) . ' Mo' . PHP_EOL;
"
```

### Identifier les YAML problématiques

```bash
# Top 10 des fichiers YAML les plus lourds (en octets)
ls -la config/sync/*.yml | sort -k5 -rn | head -10

# Chercher les Views avec plus de 3 displays (souvent lentes)
for f in config/sync/views.view.*.yml; do
  displays=$(grep -c "^  display:" "$f" 2>/dev/null || echo 0)
  echo "$displays $f"
done | sort -rn | head -10

# Modules avec le plus de fichiers de config
ls config/sync/*.yml | sed 's/config\/sync\///' | cut -d. -f1 | sort | uniq -c | sort -rn | head -15
```

### Optimisation en CI/CD

```yaml
# .gitlab-ci.yml — augmenter le timeout pour les imports lourds
deploy:
  stage: deploy
  script:
    - |
      docker compose exec -T php bash -c "
        export PHP_MAX_EXECUTION_TIME=300
        vendor/bin/drush deploy -y
      "
  timeout: 10 minutes   # Augmenter selon la taille du projet (défaut: 1h)
  variables:
    # Augmenter la mémoire PHP pour les imports de config volumineux
    PHP_MEMORY_LIMIT: 512M
```

### Stratégie de réduction du nombre de fichiers YAML

```bash
# Identifier les Views inutilisées (aucune page, aucun block actif)
docker compose exec php drush php:eval "
  \$views = \Drupal::entityTypeManager()->getStorage('view')->loadMultiple();
  foreach (\$views as \$view) {
    if (!\$view->status()) {
      echo 'DÉSACTIVÉE : ' . \$view->id() . PHP_EOL;
    }
  }
"

# Identifier les entity_view_display sans utilisation réelle
# (les display 'default' de bundles non utilisés peuvent être supprimées)
ls config/sync/ | grep "core.entity_view_display" | grep "\.default\." | wc -l

# Supprimer une View inutilisée (et exporter)
docker compose exec php drush php:eval "\Drupal::entityTypeManager()->getStorage('view')->load('view_inutilisee')->delete();"
docker compose exec php drush cex -y
```

> **Règle :** sur les projets avec 1000+ fichiers YAML, prévoir `timeout: 15 minutes` dans le pipeline CI et `PHP_MEMORY_LIMIT=512M`. L'import d'une config HGO (1252 fichiers) prend typiquement 45-90 secondes selon la charge serveur.
