# Workflow Import/Export — Commandes Drush

## Référence Complète des Commandes

```bash
# EXPORT — Active (DB) → Staged (YAML)
ddev drush config:export          # Alias: drush cex
ddev drush cex -y                 # Sans confirmation

# IMPORT — Staged (YAML) → Active (DB)
ddev drush config:import          # Alias: drush cim
ddev drush cim -y                 # Sans confirmation
ddev drush cim --partial          # ⚠️ Importer seulement les fichiers présents (dangereux en prod)

# STATUT — Comparer active vs staged
ddev drush config:status          # Alias: drush cst
# Résultat:
# "Only in active"  → en DB mais pas exporté (tu as des modifs non exportées)
# "Only in staging" → dans les fichiers mais pas importé (nouveau pour toi)
# "Different"       → existe des deux côtés mais contenu différent (conflit potentiel)

# LIRE une config
ddev drush config:get system.site              # Tout l'objet
ddev drush config:get system.site name        # Une clé spécifique
ddev drush config:get node.type.article       # Un Config Entity

# ÉCRIRE une valeur (dangereux — préférer le workflow cex/cim)
ddev drush config:set system.site name 'Mon site'
ddev drush config:set mon_module.settings max_items 25

# DIFF — Voir les différences ligne par ligne
ddev drush config:diff system.site
ddev drush config:diff views.view.frontpage

# ÉDITER — Ouvrir dans $EDITOR
ddev drush config:edit system.site

# SUPPRIMER une config entière
ddev drush config:delete mon_module.settings
ddev drush config:delete views.view.orphaned_view

# LISTER les configs
ddev drush config:list
ddev drush config:list --prefix=node.type     # Filtrer par préfixe
```

---

## `drush deploy` — Commande de Déploiement Complète (D9+)

```bash
# Exécute dans l'ordre correct : updatedb → cim → cr + hook_deploy_N après cim
ddev drush deploy -y
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
$ ddev drush config:status

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
ddev drush cex -y

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
ddev drush cst                          # Doit être clean (no differences)
ddev drush cex -y                       # Exporter ses propres modifs si "Only in active"

# 2. Récupérer la branche
git pull origin feature/ajout-content-type-projet

# 3. Appliquer la config
ddev drush cim -y
ddev drush cr
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
ddev drush cim --partial --source=/chemin/vers/config-partielle/
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
ddev drush php:eval "\Drupal::service('config.installer')->installDefaultConfig('module', 'mon_module');"

# Importer un fichier YAML manuellement (depuis Drush)
ddev drush config:set --input-format=yaml NOM_CONFIG - < config/sync/mon_module.settings.yml
```

---

## Troubleshooting — Commandes de Diagnostic

```bash
# Voir les erreurs de validation de config
ddev drush config:status --format=json

# Vérifier la config d'un module spécifique
ddev drush config:list --prefix=mon_module

# Comparer deux environnements (depuis staging qui a accès aux deux)
ddev drush config:export --destination=/tmp/config-prod
diff -r config/sync/ /tmp/config-prod/

# Réinitialiser la config d'un module à ses valeurs par défaut
ddev drush pm:uninstall mon_module -y
ddev drush pm:enable mon_module -y

# Forcer l'UUID d'un site (résolution de conflit UUID)
ddev drush config:set system.site uuid "VOTRE-UUID-ICI"
```
