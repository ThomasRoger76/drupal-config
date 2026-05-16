---
name: drupal-config
description: Use when managing Drupal configuration between environments, debugging UUID conflicts on drush cim, configuring sync directory in settings.php, implementing environment overrides with $config or config_split, understanding Simple Config vs Config Entities, resolving config dependency cascades, writing hook_deploy_N post-import scripts, translating configuration with config_translation, managing config/optional/ conditional imports, applying Drupal Recipes (recipe.yml), or configuring multisite with per-site config_split in Drupal 8-11+
---

# Drupal Config Management — Référence Complète

## Overview

Référentiel complet du système de gestion de configuration Drupal 8-11+ : cycle de vie Active/Staged, workflow import/export, anatomie YAML, UUIDs, overrides par environnement, entités de config, hooks de déploiement.

## ⚠️ Les 3 Règles d'Or — Graver dans le marbre

> **1. Git est la source de vérité.**  
> Jamais d'installation de module directement en production via l'interface. Toujours : local → `drush cex` → git commit → déploiement → `drush cim`.

> **2. Jamais de `drush cim` sans `drush cex` préalable.**  
> Vérifier que ta config locale est propre (`drush config:status`) avant d'importer celle des autres. Sinon, tes modifications locales non exportées sont silencieusement écrasées.

> **3. L'UUID du site est unique.**  
> Si tu réinstalles Drupal de zéro, tu dois synchroniser l'UUID du site source dans `system.site.yml` — sinon `drush cim` échoue avec "Site UUID does not match".

---

## Quick Decision Table

| Besoin | Outil | Référence |
|--------|-------|-----------|
| Exporter la config active → fichiers YAML | `drush cex` | [workflow-commands.md](workflow-commands.md) |
| Importer les fichiers YAML → base | `drush cim` | [workflow-commands.md](workflow-commands.md) |
| Voir les différences avant import | `drush config:status` | [workflow-commands.md](workflow-commands.md) |
| Déployer (updb → cim → cr dans l'ordre correct) | `drush deploy` | [workflow-commands.md](workflow-commands.md) |
| Lire une valeur de config depuis PHP | `\Drupal::config('nom')->get('cle')` | [config-fundamentals.md](config-fundamentals.md) |
| Écrire une config depuis PHP | `configFactory()->getEditable()->set()->save()` | [config-fundamentals.md](config-fundamentals.md) |
| Configurer le répertoire de sync | `$settings['config_sync_directory']` dans settings.php | [config-fundamentals.md](config-fundamentals.md) |
| Comprendre pourquoi `drush cim` échoue (UUID) | Synchroniser les UUIDs | [yaml-anatomy.md](yaml-anatomy.md) |
| Lire l'anatomie d'un fichier YAML config | uuid, langcode, dependencies, mapping | [yaml-anatomy.md](yaml-anatomy.md) |
| Comprendre un diff de config | `drush config:diff NOM` | [yaml-anatomy.md](yaml-anatomy.md) |
| Config différente en local vs prod | `$config['nom']['cle']` dans settings.php | [environment-overrides.md](environment-overrides.md) |
| Modules actifs seulement en local | module `config_split` | [environment-overrides.md](environment-overrides.md) |
| Exclure une config de l'import (éditée en prod) | module `config_ignore` | [environment-overrides.md](environment-overrides.md) |
| Bloquer toute modification UI en prod | module `config_readonly` | [environment-overrides.md](environment-overrides.md) |
| Distinguer Simple Config vs Config Entity | Type de stockage et API PHP | [config-entities.md](config-entities.md) |
| Lister/charger des Config Entities depuis PHP | `EntityTypeManager->getStorage('node_type')` | [config-entities.md](config-entities.md) |
| Comprendre les cascades de suppression | `dependencies:` dans les YAML | [config-entities.md](config-entities.md) |
| Modifier une config dans un hook_update | `\Drupal::configFactory()->getEditable()` | [config-hooks.md](config-hooks.md) |
| Script post-import qui tourne une seule fois | `hook_deploy_N()` | [config-hooks.md](config-hooks.md) |
| Typer et valider sa config custom | `config/schema/mon_module.schema.yml` | [config-hooks.md](config-hooks.md) |
| Config importée seulement si dépendances OK | `config/optional/` dans le module | [config-optional.md](config-optional.md) |
| Différence install/ vs optional/ | optional = conditionnel aux dépendances | [config-optional.md](config-optional.md) |
| Forcer l'import d'une optional config après install | `config.installer` service ou `drush config:import --partial` | [config-optional.md](config-optional.md) |
| Ordre CI/CD correct (config avant migrations) | `drush deploy` puis `drush migrate:import` | [config-optional.md](config-optional.md) |
| Importer config optional en hook_install | `\Drupal::service('config.installer')->installOptionalConfig()` | [config-optional.md](config-optional.md) |
| Réagir à une sauvegarde / suppression / import de config | `ConfigEvents::SAVE`, `DELETE`, `IMPORT` via EventSubscriber | [config-hooks.md](config-hooks.md) |
| Distribuer un set de config réutilisable | Drupal Recipes (`recipe.yml`) | [drupal-recipes.md](drupal-recipes.md) |
| Appliquer une recipe sur un site existant | `drush recipe web/recipes/mon_recipe` | [drupal-recipes.md](drupal-recipes.md) |
| Config Actions (createIfNotExists, grantPermissions) | `config.actions:` dans recipe.yml | [drupal-recipes.md](drupal-recipes.md) |
| Traduire la config (labels de menus, Views, blocs) | Module `config_translation` + `locale` | [config-translation.md](config-translation.md) |
| Importer des traductions `.po` en masse | `drush locale:import fr fichier.po` | [config-translation.md](config-translation.md) |
| Traduire la config programmatiquement | `language_manager->getLanguageConfigOverride()` | [config-translation.md](config-translation.md) |
| Comprendre où sont stockées les traductions de config | Tables `locale_*` de la DB (pas dans YAML) | [config-translation.md](config-translation.md) |
| Mettre à jour les traductions contrib depuis drupal.org | `drush locale:update` | [config-translation.md](config-translation.md) |
| Configurer plusieurs sites sur une installation | `sites.php` + `settings.php` par site | [multisite.md](multisite.md) |
| Config partagée + config spécifique par site | `config_split` par site, activé dans `settings.php` | [multisite.md](multisite.md) |
| Exécuter drush sur un site multisite spécifique | `drush --uri=https://site.com` | [multisite.md](multisite.md) |
| Installer un nouveau site dans un multisite | `drush site:install --uri=` + `sites.php` | [multisite.md](multisite.md) |
| Docker Compose multisite (un PHP, plusieurs DBs) | Variables env `MARIADB_DATABASE_SITE_*` par site | [multisite.md](multisite.md) |

## Anti-Patterns Critiques

| ❌ À ne jamais faire | ✅ Bonne pratique | Raison |
|---------------------|------------------|--------|
| Modifier la config en prod via UI sans exporter | Toujours local → cex → commit → déployer | Source de vérité corrompue |
| `drush cim` sans `drush cex` préalable | `drush cst` d'abord, puis cex si "only in active" | Perte silencieuse des modifs locales |
| Stocker une clé API dans les YAML (git) | `$config['nom']['api_key']` dans settings.php | Fuite de secrets dans git |
| `drush cim --partial` en production | Import complet standard | Laisse de la config orpheline |
| Supprimer un module sans gérer sa config | `drush config:delete` + vérifier les dépendances | Cascade de suppression inattendue |
| `$config_directories['sync']` (D8 syntax) | `$settings['config_sync_directory']` | Supprimé en D9 |
| `hook_update_N` pour des données qui dépendent de la config importée | `hook_deploy_N` (s'exécute après cim) | L'update s'exécute avant que la config soit en DB |
| Configurer config_ignore trop large en prod | Cibler précisément les configs éditables | Empêche les mises à jour légitimes |

## Évolution par Version Majeure

| Feature | D8 | D9 | D10 | D11 |
|---------|----|----|-----|-----|
| Config Management System | ✅ | ✅ | ✅ | ✅ |
| `$config_directories['sync']` (settings.php) | ✅ seul | ⚠️ déprécié | ❌ supprimé | ❌ |
| `$settings['config_sync_directory']` | ❌ | ✅ | ✅ standard | ✅ |
| `hook_deploy_N` | ❌ | ✅ D9.3+ | ✅ | ✅ |
| `drush deploy` (commande) | ❌ | ✅ | ✅ | ✅ |
| Config Events (save, delete, import) | ✅ | ✅ | ✅ | ✅ |
| Config Translation (core) | ✅ | ✅ | ✅ | ✅ |
| `config_split` (contrib) | ✅ | ✅ | ✅ | ✅ |
| `config_ignore` (contrib) | ✅ | ✅ | ✅ | ✅ |

## Auto-Amélioration

- **[lessons.md](lessons.md)** — Bugs et pièges découverts en projet réel.
- **[CHANGELOG.md](CHANGELOG.md)** — Historique des versions (v1.0 courante).

## See Also

- `drupal-core` — Config API PHP (lecture/écriture programmatique simple)
- `drupal-theming` — Config du thème (`config/install/` du thème)
- `drupal-testing` — Tester la config avec KernelTest
- `drupal-deployment` — drush deploy, workflow de déploiement, drush aliases
- `drupal-docker` — Environnement Docker Compose, config, hooks, services
- `drupal-multilingual` — Config Translation, overrides par langue
- `drupal-migration` — Config après migration de version, `hook_deploy_N`
