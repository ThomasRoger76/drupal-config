# Multisite Drupal — Config partagée et spécifique par site

Référence complète pour configurer plusieurs sites Drupal sur une même installation, gérer la config partagée vs spécifique par site, et orchestrer les déploiements multisite avec Docker Compose.

---

## Concepts fondamentaux

Un multisite Drupal utilise **un seul codebase** (`vendor/`, `web/core/`, `web/modules/`) mais plusieurs entrées de configuration (`web/sites/site-a/`, `web/sites/site-b/`). Chaque site peut avoir :
- Sa propre base de données
- Son propre répertoire `files/`
- Sa propre configuration (`config/sync` partagée ou séparée)
- Ses propres overrides dans `settings.php`

---

## 1. Structure multisite

```
web/sites/
├── sites.php                    # Mapping domaine → dossier de site
├── default/                     # Site par défaut (fallback)
│   ├── settings.php
│   ├── settings.local.php       # Overrides locaux (hors git)
│   └── files/
├── site-a.example.com/          # Site A
│   ├── settings.php
│   ├── settings.local.php
│   └── files/
└── site-b.example.com/          # Site B
    ├── settings.php
    ├── settings.local.php
    └── files/

config/
├── sync/                        # Config commune à tous les sites
├── sync_site_a/                 # Config spécifique au site A (optionnel)
└── sync_site_b/                 # Config spécifique au site B (optionnel)
```

---

## 2. `sites.php` — Mapping des domaines

```php
<?php
// web/sites/sites.php

// Production
$sites['site-a.example.com'] = 'site-a.example.com';
$sites['www.site-a.example.com'] = 'site-a.example.com'; // avec www
$sites['site-b.example.com'] = 'site-b.example.com';

// Staging (même dossier de site, domaine différent)
$sites['staging-site-a.example.com'] = 'site-a.example.com';
$sites['staging-site-b.example.com'] = 'site-b.example.com';

// Développement local (domaine local → dossier prod)
$sites['site-a.localhost'] = 'site-a.example.com';
$sites['site-b.localhost'] = 'site-b.example.com';

// Avec port (développement local avec proxy)
$sites['8080.site-a.localhost'] = 'site-a.example.com';
```

---

## 3. `settings.php` par site

```php
<?php
// web/sites/site-a.example.com/settings.php

// Base de données dédiée au site A
$databases['default']['default'] = [
  'driver'    => 'mysql',
  'host'      => getenv('MARIADB_HOSTNAME') ?: 'mariadb',
  'port'      => '3306',
  'database'  => getenv('MARIADB_DATABASE_SITE_A') ?: 'site_a',
  'username'  => getenv('MARIADB_USER') ?: 'drupal',
  'password'  => getenv('MARIADB_PASSWORD') ?: 'drupal',
  'prefix'    => '',
  'namespace' => 'Drupal\mysql\Driver\Database\mysql',
  'autoload'  => 'core/modules/mysql/src/Driver/Database/mysql/',
];

// Répertoire de sync : partagé OU spécifique
// Option 1 : config commune (tous les sites partagent la même base)
$settings['config_sync_directory'] = '../config/sync';

// Option 2 : config spécifique par site
// $settings['config_sync_directory'] = '../config/sync_site_a';

// Répertoire files dédié
$settings['file_public_path'] = 'sites/site-a.example.com/files';
$settings['file_private_path'] = '/var/www/private/site-a';

// Hôtes de confiance
$settings['trusted_host_patterns'] = [
  '^site-a\.example\.com$',
  '^www\.site-a\.example\.com$',
  '^site-a\.localhost$',
];

// Hash salt unique par site
$settings['hash_salt'] = getenv('HASH_SALT_SITE_A') ?: 'changez-moi-site-a';

// Inclure les overrides locaux si présents
if (file_exists(__DIR__ . '/settings.local.php')) {
  include __DIR__ . '/settings.local.php';
}
```

---

## 4. Config Split pour multisite (pattern 9 cliniques)

Quand plusieurs sites partagent 90% de la config mais ont chacun des spécificités (logo, couleurs, modules activés), `config_split` est la solution propre.

```bash
composer require drupal/config_split
docker compose exec php drush en config_split -y
```

### Créer un split par site

```yaml
# config/sync/config_split.config_split.clinique_rennes.yml
id: clinique_rennes
label: 'Clinique Rennes'
description: 'Configuration spécifique à la clinique de Rennes'
folder: 'config/sync_clinique_rennes'
status: false     # Désactivé par défaut — activé par settings.php du site
weight: 0
module: {}        # Modules activés uniquement sur ce site
theme: {}
blacklist: []     # Configs complètement exclues du sync principal
graylist: []      # Configs copiées (sync principal + split)
```

### Activer le bon split dans settings.php

```php
// web/sites/clinique-rennes.exemple.fr/settings.php
// Activer uniquement le split de Rennes
$config['config_split.config_split.clinique_rennes']['status'] = TRUE;
$config['config_split.config_split.clinique_lorient']['status'] = FALSE;
$config['config_split.config_split.clinique_brest']['status'] = FALSE;
// → Seule la config de Rennes est importée en plus de la config commune
```

### Export et import avec config_split

```bash
# Exporter (génère config/sync/ ET config/sync_clinique_rennes/)
docker compose exec php drush --uri=https://clinique-rennes.exemple.fr cex

# Importer sur le site de Rennes
docker compose exec php drush --uri=https://clinique-rennes.exemple.fr cim -y

# Vérifier le statut des splits actifs
docker compose exec php drush --uri=https://clinique-rennes.exemple.fr \
  php:eval "print_r(\Drupal::config('config_split.config_split.clinique_rennes')->getRawData());"
```

---

## 5. Commandes drush pour multisite

**Règle absolue** : toujours passer `--uri` pour cibler un site précis. Sans `--uri`, drush utilise le site `default`.

```bash
# Vider le cache d'un site spécifique
docker compose exec php drush --uri=https://site-a.example.com cr

# Importer la config sur le site B seulement
docker compose exec php drush --uri=https://site-b.example.com cim -y

# Voir l'état de la config d'un site
docker compose exec php drush --uri=https://site-a.example.com config:status

# Déploiement complet sur un site (updb + cim + cr)
docker compose exec php drush --uri=https://site-a.example.com deploy -y

# Lancer les updates sur tous les sites
for URI in https://site-a.example.com https://site-b.example.com; do
  docker compose exec php drush --uri="${URI}" updb -y
  docker compose exec php drush --uri="${URI}" cim -y
  docker compose exec php drush --uri="${URI}" cr
done
```

### Alias drush par site

```yaml
# drush/sites/site-a.site.yml
site-a:
  host: drupal-php-container  # nom du service Docker
  uri: https://site-a.example.com
  user: www-data
  root: /var/www/html/web

# drush/sites/site-b.site.yml
site-b:
  uri: https://site-b.example.com
  root: /var/www/html/web
```

```bash
# Utilisation des alias
drush @site-a cr
drush @site-b cim -y
```

---

## 6. `docker-compose.yml` pour multisite

Un seul container PHP dessert tous les sites — les domaines sont différenciés par `sites.php` et les `trusted_host_patterns` dans chaque `settings.php`.

```yaml
# docker-compose.yml
services:
  php:
    image: monregistre/drupal-php:latest
    environment:
      # Chaque site a sa propre DB
      MARIADB_HOSTNAME: mariadb
      MARIADB_USER: drupal
      MARIADB_PASSWORD: ${MARIADB_PASSWORD}
      MARIADB_DATABASE_SITE_A: ${BDD_SITE_A_DATABASE:-site_a}
      MARIADB_DATABASE_SITE_B: ${BDD_SITE_B_DATABASE:-site_b}
      # Hash salts distincts par site
      HASH_SALT_SITE_A: ${HASH_SALT_SITE_A}
      HASH_SALT_SITE_B: ${HASH_SALT_SITE_B}
    volumes:
      - ./web:/var/www/html/web
      - ./config:/var/www/html/config
      - files_site_a:/var/www/html/web/sites/site-a.example.com/files
      - files_site_b:/var/www/html/web/sites/site-b.example.com/files

  mariadb:
    image: mariadb:10.11
    environment:
      MARIADB_ROOT_PASSWORD: ${MARIADB_ROOT_PASSWORD}
      MARIADB_USER: drupal
      MARIADB_PASSWORD: ${MARIADB_PASSWORD}
      # Créer plusieurs bases au démarrage
      MARIADB_DATABASE: site_a  # DB principale (les autres créées via init scripts)
    volumes:
      - db_data:/var/lib/mysql
      - ./docker/mariadb/init:/docker-entrypoint-initdb.d  # scripts CREATE DATABASE

  caddy:
    image: caddy:2-alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./docker/caddy/Caddyfile:/etc/caddy/Caddyfile
      # Caddy route site-a.example.com et site-b.example.com vers le même PHP

volumes:
  db_data:
  files_site_a:
  files_site_b:
```

```bash
# Script d'initialisation des bases — docker/mariadb/init/01-create-databases.sql
CREATE DATABASE IF NOT EXISTS site_a CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE DATABASE IF NOT EXISTS site_b CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
GRANT ALL PRIVILEGES ON site_a.* TO 'drupal'@'%';
GRANT ALL PRIVILEGES ON site_b.* TO 'drupal'@'%';
```

---

## 7. Pipeline CI/CD multisite

```yaml
# .gitlab-ci.yml — déploiement sur tous les sites
.deploy_site: &deploy_site
  script:
    - docker compose exec php drush --uri="${SITE_URI}" updb -y
    - docker compose exec php drush --uri="${SITE_URI}" cim -y
    - docker compose exec php drush --uri="${SITE_URI}" cr
    - docker compose exec php drush --uri="${SITE_URI}" locale:update

deploy:site-a:
  <<: *deploy_site
  variables:
    SITE_URI: https://site-a.example.com

deploy:site-b:
  <<: *deploy_site
  variables:
    SITE_URI: https://site-b.example.com
  needs: [deploy:site-a]  # Déploiement séquentiel (ou parallel: si indépendants)
```

---

## 8. Installer un nouveau site dans un multisite existant

```bash
# 1. Créer le dossier du site
mkdir -p web/sites/site-c.example.com

# 2. Créer settings.php à partir d'un site existant
cp web/sites/site-a.example.com/settings.php web/sites/site-c.example.com/settings.php
# → Modifier : base de données, trusted_host_patterns, hash_salt, config_sync_directory

# 3. Ajouter l'entrée dans sites.php
# $sites['site-c.example.com'] = 'site-c.example.com';

# 4. Créer la base de données
docker compose exec mariadb mysql -uroot -p${MARIADB_ROOT_PASSWORD} -e \
  "CREATE DATABASE site_c; GRANT ALL ON site_c.* TO 'drupal'@'%';"

# 5. Installer Drupal sur ce site
docker compose exec php drush --uri=https://site-c.example.com site:install \
  --db-url=mysql://drupal:${MARIADB_PASSWORD}@mariadb/site_c \
  --site-name="Site C" -y

# 6. Importer la config commune
docker compose exec php drush --uri=https://site-c.example.com cim -y
```

---

## Anti-patterns multisite

| ❌ À ne jamais faire | ✅ Bonne pratique | Impact |
|---------------------|------------------|--------|
| Même DB pour plusieurs sites | Une DB dédiée par site | Mélange de données, suppressions en cascade |
| Config sync commune sans `config_split` | `config_split` pour les différences par site | Configurations écrasées entre sites |
| Pas d'alias drush | Toujours `--uri=` ou aliases `.site.yml` | Impossible de cibler un site depuis CI |
| `trusted_host_patterns` générique `.*` | Patterns stricts par site | Host header injection |
| Hash salt identique pour tous les sites | Hash salt unique par site dans `.env` | Sessions et tokens inter-sites compromis |
| `drush cim` sans `--uri` en multisite | Toujours préciser `--uri` | Importe sur le mauvais site |
| Files partagés entre sites | Volumes Docker séparés par site | Collisions de noms de fichiers |

## Voir aussi

- `config-fundamentals.md` — Répertoire sync, UUID, drush cex/cim
- `environment-overrides.md` — `config_split`, `config_ignore`, overrides `$config[]`
- `workflow-commands.md` — Ordre de déploiement, `drush deploy`
- `drupal-docker` — Stack Docker Compose, Caddy, PHP-FPM
