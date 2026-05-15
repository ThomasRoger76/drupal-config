# Config Translation — i18n de la Configuration Drupal

Référence complète pour traduire la configuration Drupal (labels de menus, titres de Views, chaînes de formulaires, etc.) via le module `config_translation` et les fichiers `.po`.

---

## Concepts fondamentaux

La **Config Translation** traduit la configuration de Drupal (menus, Views, blocs, types de contenu…), **pas** le contenu éditorial (nodes, terms). C'est une distinction critique.

| | Content Translation | Config Translation |
|--|--|--|
| Ce qui est traduit | Nodes, termes de taxo, paragraphes | Menus, labels Views, blocs, formulaires |
| Module requis | `content_translation` | `config_translation` |
| Stockage | Tables `entity_revision` | Tables `locale_*` de la DB |
| Exportable dans `config/sync/` | Non (données) | Non — les traductions ne sont PAS dans les YAML |
| Format de fichier | N/A | Fichiers `.po` |

**Point clé** : les traductions de config ne vivent **pas** dans les fichiers YAML de `config/sync/`. Elles sont stockées dans la base de données (tables `locale_*`) et exportées/importées via des fichiers `.po`.

---

## 1. Modules requis

```bash
docker compose exec php drush en config_translation locale language -y
```

- `language` — gestion des langues actives
- `locale` — stockage et import/export `.po`
- `config_translation` — surcharge de configuration par langue

### Ajouter une langue

```bash
# Ajouter le français
docker compose exec php drush language:add fr
# Vérifier les langues actives
docker compose exec php drush php:eval "
  foreach (\Drupal::service('language_manager')->getLanguages() as \$lang) {
    echo \$lang->getId() . ' — ' . \$lang->getName() . PHP_EOL;
  }
"
```

---

## 2. Où vit la config traduite

```
Les traductions de config → tables locale_* de la DB
                           PAS dans config/sync/

Structure des tables :
  locale_source    : chaînes sources (anglais)
  locale_target    : traductions par langue
  locale_file      : fichiers .po importés (traçabilité)
```

Pour vérifier qu'une traduction est bien stockée :

```bash
docker compose exec php drush php:eval "
  // Chercher les traductions d'une chaîne spécifique
  \$results = \Drupal::database()->select('locale_target', 'lt')
    ->fields('lt', ['source', 'translation', 'language'])
    ->condition('lt.language', 'fr')
    ->range(0, 10)
    ->execute()
    ->fetchAll();
  foreach (\$results as \$r) {
    echo \$r->language . ' | ' . substr(\$r->source, 0, 40) . ' → ' . \$r->translation . PHP_EOL;
  }
"
```

---

## 3. Importer des traductions `.po`

```bash
# Importer un fichier .po officiel Drupal pour une langue
docker compose exec php drush locale:import fr \
  web/sites/default/files/translations/drupal-11.fr.po

# Importer les traductions d'un module custom (marqué "customized" pour ne pas être écrasé par locale:update)
docker compose exec php drush locale:import fr \
  web/modules/custom/mon_module/translations/fr.po \
  --type=customized

# Mettre à jour les traductions contrib depuis drupal.org
docker compose exec php drush locale:update

# Forcer la mise à jour même si la traduction est récente
docker compose exec php drush locale:update --langcodes=fr
```

### Structure d'un fichier `.po` minimal

```po
# Traductions françaises de mon_module
# Copyright (C) 2024
msgid ""
msgstr ""
"Language: fr\n"
"Content-Type: text/plain; charset=UTF-8\n"

msgid "Save configuration"
msgstr "Enregistrer la configuration"

msgid "The configuration has been saved."
msgstr "La configuration a été enregistrée."

# Contexte pour éviter les collisions de chaînes identiques
msgctxt "mon_module:block"
msgid "Title"
msgstr "Titre du bloc"
```

### Exporter les traductions actuelles

```bash
# Via l'UI : /admin/config/regional/translate/export
# Via drush (export en .po)
docker compose exec php drush php:eval "
  // Forcer la reconstruction du cache de traduction
  \Drupal::service('locale.storage')->resetCache();
  echo 'Cache locale vidé.';
"
# Exporter via l'UI pour récupérer les .po custom à versionner dans git
```

---

## 4. Traduire la config via PHP

```php
use Drupal\language\ConfigurableLanguageManagerInterface;

// Charger et modifier la config d'un site dans une langue spécifique
$language_manager = \Drupal::service('language_manager');

// Écrire la traduction d'une config
$fr_config = $language_manager->getLanguageConfigOverride('fr', 'system.site');
$fr_config->set('name', 'Mon Site Français')->save();

$fr_config = $language_manager->getLanguageConfigOverride('fr', 'system.site');
$fr_config->set('slogan', 'Bienvenue sur notre site')->save();

// Lire la config traduite
$site_name_fr = $language_manager
  ->getLanguageConfigOverride('fr', 'system.site')
  ->get('name');

// Lire avec fallback sur la config source si traduction absente
$site_name = \Drupal::config('system.site')->get('name'); // langue active courante
```

### Traduire un label de Views

```php
// Modifier le titre d'une vue dans une langue spécifique
$language_manager = \Drupal::service('language_manager');
$fr_override = $language_manager->getLanguageConfigOverride('fr', 'views.view.frontpage');
$fr_override->set('label', 'Page d\'accueil')->save();

// Modifier le titre d'un display spécifique
$fr_override->set('display.default.display_title', 'Défaut')->save();
```

### Traduire un menu

```php
// Titre d'un menu en français
$fr_override = \Drupal::service('language_manager')
  ->getLanguageConfigOverride('fr', 'system.menu.main');
$fr_override->set('label', 'Navigation principale')->save();
```

---

## 5. Workflow de traduction de config dans un projet

```
1. Développer en anglais (langue de référence = langcode du YAML)
   → Tous les labels, titres de Views, menus sont en anglais

2. Exporter la config : drush cex
   → Les YAML contiennent les chaînes anglaises, pas les traductions

3. Traduire dans l'UI ou via fichiers .po
   → /admin/config/regional/translate
   → OU préparer des .po et les importer

4. Exporter les traductions custom en .po et versionner dans git
   → web/modules/custom/mon_module/translations/fr.po
   → web/sites/default/files/translations/ pour les fichiers contrib

5. En prod / staging : drush locale:update (modules contrib)
   ET drush locale:import fr .../fr.po --type=customized (custom)

6. Pipeline CI/CD :
   docker compose exec php drush deploy
   docker compose exec php drush locale:update
   docker compose exec php drush locale:import fr web/modules/custom/*/translations/fr.po --type=customized
```

---

## 6. `langcode` dans les YAML de config

```yaml
# system.site.yml — exemple
langcode: fr    # Langue par défaut du site (pas la langue des traductions !)
uuid: xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
name: 'Mon Site'
slogan: ''
# Les traductions françaises sont dans locale_* si langcode != fr
# Si langcode: fr, les valeurs YAML sont déjà "en français"
```

**Important** : le `langcode` dans le YAML est la langue de la valeur stockée, pas une instruction de traduction. Si `langcode: fr`, le nom du site dans le YAML est déjà en français — les traductions dans `locale_*` s'appliquent aux autres langues actives.

---

## 7. Débogage des traductions manquantes

```bash
# Vérifier si une chaîne est connue du système de traduction
docker compose exec php drush php:eval "
  \$results = \Drupal::database()
    ->select('locale_source', 'ls')
    ->fields('ls', ['source', 'context', 'version'])
    ->condition('ls.source', '%Save%', 'LIKE')
    ->range(0, 5)
    ->execute()
    ->fetchAll();
  foreach (\$results as \$r) {
    echo \$r->source . ' [' . \$r->context . ']' . PHP_EOL;
  }
"

# Vider le cache de traduction (si les traductions n'apparaissent pas)
docker compose exec php drush cr
docker compose exec php drush locale:update --langcodes=fr
```

---

## 8. Config Translation dans le pipeline CI/CD

```yaml
# .gitlab-ci.yml — import des traductions après déploiement
deploy:translations:
  stage: post-deploy
  script:
    # Mettre à jour les traductions des modules contrib
    - docker compose exec php drush locale:update --langcodes=fr
    # Importer les traductions custom versionnées dans git
    - |
      for PO_FILE in web/modules/custom/*/translations/fr.po; do
        if [ -f "${PO_FILE}" ]; then
          docker compose exec php drush locale:import fr "${PO_FILE}" --type=customized
        fi
      done
    - docker compose exec php drush cr
```

---

## Anti-patterns

| ❌ À ne jamais faire | ✅ Bonne pratique | Raison |
|---------------------|------------------|--------|
| Chercher les traductions dans `config/sync/` YAML | Regarder dans les tables `locale_*` | Les traductions ne sont pas dans les YAML |
| Modifier directement les YAML pour traduire | `drush locale:import` ou UI translate | `drush cim` écrasera les modifications |
| Ne pas versionner les `.po` custom | Versionner dans `translations/fr.po` du module | Perte des traductions au redéploiement |
| `drush locale:update` sans `--type=customized` sur l'import | Toujours `--type=customized` pour le custom | Les traductions custom seraient écrasées |
| Mélanger Content Translation et Config Translation | Modules séparés, stockages séparés | Confusion sur où chercher/modifier |

## Voir aussi

- `config-fundamentals.md` — Cycle de vie Active/Staged, drush cex/cim
- `environment-overrides.md` — `$config[]` overrides par environnement
- `workflow-commands.md` — Ordre des commandes en déploiement
