---
name: config-doctor
description: Diagnostique et répare les problèmes de configuration Drupal — conflits UUID, config/sync désynchronisée, config_split malconfiguré, dépendances manquantes, YAML malformés.
---

# Agent : config-doctor

## Rôle

Diagnostiquer automatiquement l'état de la configuration Drupal et proposer/appliquer les corrections.

## Déclenchement

```bash
/drupal-config-doctor              # Diagnostic complet
/drupal-config-doctor --fix        # Diagnostic + corrections automatiques
/drupal-config-doctor --uuid       # UUID conflicts uniquement
/drupal-config-doctor --split      # config_split status uniquement
```

## Pipeline d'exécution

### Étape 1 — État général
```bash
docker compose exec php drush config:status
docker compose exec php drush status --fields=drupal-version,php-version,db-driver
```

### Étape 2 — Conflits UUID
```bash
# Vérifier que l'UUID du site correspond aux fichiers YAML
docker compose exec php drush php:eval "echo \Drupal::config('system.site')->get('uuid');"
grep "^uuid:" config/default/sync/system.site.yml
# Si différent → corriger automatiquement
```

**Fix automatique UUID :**
```bash
SITE_UUID=$(docker compose exec php drush php:eval "echo \Drupal::config('system.site')->get('uuid');")
sed -i "s/^uuid: .*/uuid: $SITE_UUID/" config/default/sync/system.site.yml
```

### Étape 3 — YAML malformés
```bash
# Vérifier la syntaxe de tous les YAML de sync
find config/default/sync -name "*.yml" | while read f; do
  python3 -c "import yaml; yaml.safe_load(open('$f'))" 2>&1 | grep -v "^$" && echo "❌ $f"
done
```

### Étape 4 — Config_split status
```bash
docker compose exec php drush php:eval "
  foreach (\Drupal::configFactory()->listAll('config_split.config_split') as \$name) {
    \$config = \Drupal::config(\$name);
    echo \$config->get('id') . ': ' . (\$config->get('status') ? 'actif' : 'inactif') . PHP_EOL;
  }
"
```

### Étape 5 — Dépendances manquantes
```bash
docker compose exec php drush php:eval "
  \$storage = \Drupal::service('config.storage');
  \$manager = \Drupal::service('config.manager');
  \$errors = \$manager->findConfigEntityDependents('missing', []);
  foreach (\$errors as \$error) { echo \$error . PHP_EOL; }
"
```

### Étape 6 — Rapport
```markdown
## Config Doctor Report — [DATE]

### ✅ UUID : OK
### ❌ YAML malformés : 2 fichiers
  - config/sync/views.view.broken.yml (ligne 45 : indentation incorrecte)
### ⚠️ Config Split : dev actif en staging (attendu: inactif)
### ✅ Dépendances : 0 manquante

### Commandes correctives suggérées
drush config:delete views.view.broken
# Réexporter : drush cex
```
