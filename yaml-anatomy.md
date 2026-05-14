# Anatomie YAML & UUIDs

## Structure d'un Fichier Simple Config

```yaml
# system.site.yml — Configuration globale du site
uuid: 'a8a38c7c-1b2c-4d5e-8f9a-0b1c2d3e4f5a'   # UUID du SITE (identifie ce déploiement Drupal)
langcode: fr                                       # Langue par défaut du fichier config
name: 'Mon site Drupal'
mail: 'admin@example.com'
slogan: 'Propulsé par Drupal'
page:
  403: '/access-denied'
  404: '/page-not-found'
  front: /node
admin_compact_mode: false
weight_select_max: 100
default_langcode: fr
```

## Structure d'un Config Entity

```yaml
# node.type.article.yml — Type de contenu "Article"
uuid: 'c3d4e5f6-7890-1234-abcd-ef0123456789'    # UUID de CET OBJET CONFIG (pas celui du site)
langcode: en
status: true
dependencies:
  module:
    - menu_ui
    - path
    - text
    - options
third_party_settings:
  menu_ui:
    available_menus:
      - main
    parent: 'main:'
  pathauto:
    enabled: true
name: 'Article'
type: article
description: "Utiliser les articles pour du contenu daté."
help: ''
new_revision: true
preview_mode: 1
display_submitted: true
```

```yaml
# field.field.node.article.field_image.yml — Définition d'un champ
uuid: 'd5e6f7a8-...'
langcode: en
status: true
dependencies:
  config:
    - field.storage.node.field_image   # ← Dépend du stockage du champ
    - node.type.article                # ← Dépend du Content Type
  module:
    - image
    - node
id: node.article.field_image
field_name: field_image
entity_type: node
bundle: article
label: 'Image principale'
description: ''
required: false
translatable: true
default_value: []
field_type: image
settings:
  file_directory: '[date:custom:Y]-[date:custom:m]'
  file_extensions: 'png gif jpg jpeg webp'
  max_filesize: '2 MB'
  max_resolution: ''
  min_resolution: ''
  alt_field: true
  alt_field_required: true
  title_field: false
  title_field_required: false
  default_image:
    uuid: null
```

---

## Les Deux UUIDs — Ne Pas Confondre

| | UUID du SITE | UUID de l'OBJET |
|--|-------------|-----------------|
| **Où** | `system.site.yml` → clé `uuid:` | Dans chaque config entity → clé `uuid:` |
| **Identifie** | Ce déploiement Drupal (installation) | Cette entité de config spécifique |
| **Problème typique** | Sites différents → UUIDs différents → cim échoue | Objet recréé avec nouvel UUID → conflicts diff |
| **Solution** | Synchroniser avec `drush config:set system.site uuid "..."` | Utiliser le fichier YAML source, pas recréer |

---

## Le Problème UUID du Site — Analyse Complète

### Symptôme

```bash
$ ddev drush cim -y
# Erreur :
[error]  The import failed due to the following reasons:
         Site UUID in source storage does not match the site UUID in target storage.
```

### Cause

Tu essaies d'importer des fichiers YAML d'un site A dans un site B (ou une réinstallation fraîche). L'UUID dans `system.site.yml` est `AAA-BBB`, mais le site cible a un UUID `XXX-YYY`.

### Résolutions

**Option 1 — Synchroniser l'UUID du site cible (recommandé pour les réinstallations)**
```bash
# Lire l'UUID du site source (dans tes fichiers YAML)
grep "^uuid:" config/sync/system.site.yml
# → uuid: 'a8a38c7c-1b2c-4d5e-8f9a-0b1c2d3e4f5a'

# Appliquer cet UUID au site cible
ddev drush config:set system.site uuid "a8a38c7c-1b2c-4d5e-8f9a-0b1c2d3e4f5a"
ddev drush cim -y
```

**Option 2 — Mettre à jour le YAML (si le site cible est le "bon")**
```bash
# Lire l'UUID du site cible
ddev drush config:get system.site uuid

# Mettre à jour le fichier YAML
# → éditer config/sync/system.site.yml → remplacer l'uuid
# Attention : ce changement va dans git — documenter pourquoi
```

**Option 3 — Pour un nouveau site qui reprend la config d'un autre**
```bash
# Script de setup initial (une seule fois)
ddev drush config:set system.site uuid "$(grep '^uuid:' config/sync/system.site.yml | awk '{print $2}' | tr -d "'")"
ddev drush cim -y
```

---

## Lire un Diff de Config — Prendre la Bonne Décision

```bash
# Voir le diff détaillé
ddev drush config:diff views.view.frontpage
```

**Exemple de diff :**
```diff
--- a/config/sync/views.view.frontpage.yml
+++ Active config (database)
@@ -15,7 +15,7 @@
   display:
     default:
       display_options:
-        items_per_page: 10
+        items_per_page: 6
```

### Grille de décision

| Situation | Action |
|-----------|--------|
| Diff sur des valeurs que TU as changées en local | `drush cex` pour exporter tes modifs, puis `drush cim` sera propre |
| Diff sur des valeurs que QUELQU'UN D'AUTRE a changées via git | `drush cim` — les fichiers YAML ont raison |
| Diff sur des valeurs que le CLIENT a changées en prod | Décider consciemment : écraser ou pas ? Utiliser `config_ignore` pour l'avenir |
| Diff sur des UUIDs d'objets (ex: bloc recréé en prod) | Problème plus profond — voir section UUID des objets |

---

## Le bloc `dependencies:` — Comment Drupal Protège l'Intégrité

```yaml
dependencies:
  config:
    - field.storage.node.field_image    # Ce champ ne peut pas exister sans ce stockage
    - node.type.article                  # Ce champ ne peut pas exister sans ce CT
  module:
    - image                              # Module requis
    - node
  theme:
    - olivero                            # Pour les configs de thème
  enforced:
    module:
      - mon_module                       # ← Empêche la suppression de la config si le module est actif
```

**Comportement lors de `drush cim` :**
- Si une dépendance `config:` est absente → la config n'est pas importée (silencieusement ignorée ou erreur)
- Si une dépendance `module:` est absente → la config est ignorée jusqu'à activation du module
- Si `enforced.module:` est déclaré → la config ne peut être supprimée que si le module est désinstallé

---

## Nommage des Fichiers YAML

Convention : `[PROVIDER].[TYPE].[IDENTIFIANT].yml`

```
# Simple Config — un seul fichier par objet
system.site.yml                              # Fournisseur: system, Type: site
node.settings.yml                            # Fournisseur: node, Type: settings
mon_module.settings.yml                      # Fournisseur: mon_module, Type: settings

# Config Entities — un fichier par instance
node.type.article.yml                        # Content Type article
node.type.page.yml                           # Content Type page
field.storage.node.field_image.yml           # Stockage du champ field_image sur node
field.field.node.article.field_image.yml     # Instanciation du champ sur article
views.view.frontpage.yml                     # Vue "frontpage"
user.role.editor.yml                         # Rôle "editor"
taxonomy.vocabulary.tags.yml                 # Vocabulaire "tags"
image.style.thumbnail.yml                    # Style d'image thumbnail
system.menu.main.yml                         # Menu principal
workflows.workflow.editorial.yml             # Workflow éditorial
```
