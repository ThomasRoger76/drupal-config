# Stratégies d'Overrides par Environnement

## `$config` dans `settings.php` — Override Runtime

Les overrides `$config` s'appliquent **à la lecture** en PHP — ils ne modifient pas la DB et ne s'exportent pas avec `drush cex`.

```php
// settings.local.php (gitignored — uniquement en local)
// settings.php (avec guard `if (getenv('APP_ENV') === 'local')`)

// Override simple d'une valeur
$config['system.site']['name'] = 'Mon Site [LOCAL]';
$config['system.performance']['css']['preprocess'] = 0;
$config['system.performance']['js']['preprocess'] = 0;

// Désactiver l'envoi d'emails en dev (rediriger vers le log)
$config['system.mail']['interface']['default'] = 'devel_mail_log';

// Override d'une valeur imbriquée
$config['smtp.settings']['smtp_host'] = 'localhost';
$config['smtp.settings']['smtp_port'] = 1025;  // Mailhog/Mailpit en local

// Ne JAMAIS mettre des secrets en YAML — les overrider ici
$config['mon_module.settings']['api_key'] = getenv('MON_API_KEY') ?: 'dev-key-local';
$config['search_api.server.elasticsearch']['backend_config']['connector_config']['url'] = 'http://localhost:9200';
```

**Ce que l'override `$config` fait / ne fait pas :**

| | `$config` override | Config en DB (via UI/cex) |
|--|-------------------|--------------------------|
| Visible dans `drush config:get` | ✅ (override appliqué) | ✅ |
| Exporté par `drush cex` | ❌ | ✅ |
| Modifie la DB | ❌ | ✅ |
| Survit à `drush cim` | ✅ | ❌ (écrasé par cim) |
| Sécurisé pour secrets | ✅ (non versionné) | ❌ (dans git) |

---

## `settings.local.php` — Pattern Recommandé

```php
// settings.php — activé si le fichier existe (gitignored)
if (file_exists($app_root . '/' . $site_path . '/settings.local.php')) {
  include $app_root . '/' . $site_path . '/settings.local.php';
}
```

```php
// web/sites/default/settings.local.php (GITIGNORE ce fichier)
<?php

// Désactiver tous les caches en développement
$settings['cache']['bins']['render'] = 'cache.backend.null';
$settings['cache']['bins']['page'] = 'cache.backend.null';
$settings['cache']['bins']['dynamic_page_cache'] = 'cache.backend.null';

// Twig debug
$config['system.performance']['css']['preprocess'] = FALSE;
$config['system.performance']['js']['preprocess'] = FALSE;

// Config spécifiques à l'environnement local
$config['system.site']['name'] = 'Mon Site [LOCAL - ' . getenv('USER') . ']';
$config['mon_module.settings']['api_key'] = 'dev-test-key-not-for-prod';

// Sync directory local si différent
// $settings['config_sync_directory'] = '../config/sync';
```

---

## Module `config_split` — Configs par Environnement

Config Split permet d'avoir des sets de configuration différents selon l'environnement.

```bash
composer require drupal/config_split
ddev drush en config_split -y
```

### Setup : 3 splits recommandés

**Split 1 : `dev` — Modules de développement uniquement**
- Répertoire : `config/dev/`
- Modules complets (désactivés en prod) : `devel`, `kint`, `webprofiler`, `views_ui`
- Status : actif en local, inactif en prod

**Split 2 : `prod` — Config de production uniquement**
- Répertoire : `config/prod/`
- Exemples : taille des caches différente, CDN activé, logging réduit

**Split 3 : `local` — Surcharges locales**
- Répertoire : `config/local/`
- CSS/JS preprocess désactivés, Twig debug

### Configuration dans `settings.php`

```php
// settings.local.php — activer le split "dev"
$config['config_split.config_split.dev']['status'] = TRUE;
$config['config_split.config_split.prod']['status'] = FALSE;
$config['config_split.config_split.local']['status'] = TRUE;
```

```php
// settings.php en production
$config['config_split.config_split.dev']['status'] = FALSE;
$config['config_split.config_split.prod']['status'] = TRUE;
$config['config_split.config_split.local']['status'] = FALSE;
```

### Exemple de fichier `config_split.config_split.dev.yml`

```yaml
# config/sync/config_split.config_split.dev.yml
# Créé automatiquement après configuration via l'UI
langcode: en
status: true
dependencies: {}
id: dev
label: 'Development'
description: 'Modules et config uniquement en local/dev'
weight: 0
storage: collection              # 'collection' = dossier séparé (config/dev/)
directory: config/dev            # Chemin relatif au projet
module:
  devel: 0                       # Module à inclure uniquement dans ce split
  kint: 0
  webprofiler: 0
  views_ui: 0
theme: {}
config: {}                       # Configs individuelles à splitter (sans désactiver le module)
```

### Workflow avec Config Split

```bash
# En LOCAL — exporter avec les splits actifs
ddev drush cex -y
# → génère config/sync/ (commun) + config/dev/ (modules dev)

# En PROD — importer avec les splits prod actifs
drush cim -y
# → importe config/sync/ (commun) + config/prod/ (prod only)
# → IGNORE config/dev/ (split dev inactif en prod)
```

**Structure des répertoires :**
```
config/
├── sync/           # Config commune (tous les envs)
│   ├── system.site.yml
│   ├── node.type.article.yml
│   └── ...
├── dev/            # Config split "dev" (local/staging seulement)
│   ├── core.extension.yml          # Avec devel, kint activés
│   ├── devel.settings.yml
│   └── ...
├── prod/           # Config split "prod" (production seulement)
│   └── ...
└── local/          # Config split "local" (développeur uniquement)
    └── ...
```

---

## Module `config_ignore` — Exclure des Configs de l'Import

Config Ignore empêche certaines configs d'être écrasées lors d'un `drush cim`.

```bash
composer require drupal/config_ignore
ddev drush en config_ignore -y
```

### Configuration (`/admin/config/development/configuration/ignore`)

```yaml
# Contenu du fichier config/sync/config_ignore.settings.yml
# ⚠️ Le fichier ne contient PAS son propre nom comme clé racine
langcode: en
ignored_config_entities:
  - system.menu.main           # Menu que le client édite en prod
  - system.menu.footer         # Idem
  - webform.*                  # Tous les formulaires Webform (wildcard)
  - google_analytics.*         # Config GA (différente par env)
  - 'contact.form.*'           # Tous les formulaires de contact
```

**Cas d'usage typiques :**
- Menus édités par le client en production
- Config d'analytics (Google Analytics ID différent par env)
- Formulaires Webform maintenus par les éditeurs
- Configuration spécifique d'un bloc Drupal éditable

**⚠️ Attention :** Config ignorée ne veut pas dire jamais mise à jour. Si tu dois changer la config ignorée, tu dois le faire manuellement en prod OU retirer temporairement l'ignore.

---

## Module `config_readonly` — Verrouiller la Production

Config Readonly empêche toute modification de configuration via l'UI en production.

```bash
composer require drupal/config_readonly
ddev drush en config_readonly -y
```

```php
// settings.php en production uniquement
$settings['config_readonly'] = TRUE;

// Ou conditionnel selon l'environnement
if (getenv('APP_ENV') === 'production') {
  $settings['config_readonly'] = TRUE;
}
```

Résultat : tout formulaire qui modifierait la config affiche un message d'erreur. Force le workflow Git.

---

## Pattern Complet Multi-Environnements

```
LOCAL dev
  ├── settings.local.php   → $config overrides + cache null + config_split.dev actif
  ├── config/sync/         → source de vérité commune
  ├── config/dev/          → modules dev (config_split)
  └── config/local/        → prefs développeur (config_split)

STAGING
  ├── settings.staging.php → DB staging, config_split.prod partiellement actif
  └── config/sync/         → même que local (git pull + drush deploy)

PRODUCTION
  ├── settings.php         → config_readonly = TRUE, config_split.prod actif
  ├── settings.secret.php  → clés API, passwords (gitignored, déployé hors git)
  └── config/sync/         → même que local (git pull + drush deploy)
```
