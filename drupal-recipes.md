---
name: drupal-config — drupal recipes
description: Drupal Recipes (D10.3+ / D11) — nouveau mécanisme de distribution de configuration, remplaçant progressivement les distributions. Commandes drush recipe, structure YAML, config actions, différences avec les modules.
---

# Drupal Recipes — D10.3+ / D11

## Concept

Les Recipes remplacent progressivement les **distributions** et les **starter kits**. Une Recipe est un ensemble de configuration, de modules et d'actions applicable à n'importe quel site Drupal.

```
Une Recipe ≠ Un Module
- Recipe : appliquée une seule fois, modifie le site en place
- Module : installé en permanence, peut être désinstallé
```

## Structure d'une Recipe

```
web/recipes/mon_projet_blog/
├── recipe.yml          # Manifest principal
├── config/             # Config YAML à importer
│   ├── node.type.article.yml
│   └── field.storage.node.field_image.yml
└── content/            # Contenu de démonstration (optionnel)
```

```yaml
# recipe.yml
name: 'Mon Projet — Blog'
description: 'Configure le type de contenu Article, les champs et les Views'
type: 'Site'   # ou 'Module', 'Theme'

# Modules à installer avant d'appliquer la recipe
install:
  - node
  - views
  - field
  - text
  - image

# Config actions — transformations sur la config existante
config:
  actions:
    # Créer si n'existe pas
    node.type.article:
      createIfNotExists:
        name: 'Article'
        description: 'Article de blog standard'
        new_revision: true
        preview_mode: 1
        help: ''
        display_submitted: true

    # Modifier une config existante
    system.site:
      simpleConfigUpdate:
        name: 'Mon Site Blog'

    # Ajouter une permission à un rôle
    user.role.editor:
      grantPermissions:
        - 'create article content'
        - 'edit own article content'
        - 'delete own article content'

# Appliquer d'autres Recipes en dépendance
recipes:
  - core/recipes/editorial_workflow   # Workflows de contenu
```

## Commandes

```bash
# Appliquer une recipe
docker compose exec php drush recipe web/recipes/mon_projet_blog

# Vérifier les dépendances d'une recipe
docker compose exec php drush recipe:info web/recipes/mon_projet_blog

# Lister les recipes disponibles
ls web/recipes/
ls web/core/recipes/   # Recipes core Drupal

# Recipes core disponibles en D11
ls web/core/recipes/
# → audio_media_type, document_media_type, editorial_workflow,
#   image_media_type, local_video_media_type, remote_video_media_type...
```

## Config Actions Disponibles

```yaml
config:
  actions:
    # Créer si n'existe pas (idempotent)
    node.type.page:
      createIfNotExists:
        name: 'Page'

    # Mise à jour simple de config
    system.site:
      simpleConfigUpdate:
        front: '/node'
        403: '/access-denied'
        404: '/not-found'

    # Accorder des permissions à un rôle
    user.role.authenticated:
      grantPermissions:
        - 'access content'
        - 'search content'

    # Révoquer des permissions
    user.role.anonymous:
      revokePermissions:
        - 'access content'

    # Ajouter un module à une Config Entity
    # (ex: ajouter un champ à un Content Type existant)
    field.field.node.article.field_image:
      createIfNotExists:
        label: 'Image principale'
        field_type: 'image'
        required: false
```

## Différence Recipe vs Module vs Distribution

| Critère | Recipe | Module | Distribution |
|---------|--------|--------|-------------|
| Installation | `drush recipe` — une fois | `drush en` — permanent | Profil complet |
| Réversible | ❌ Non (modifie en place) | ✅ `drush pm-uninstall` | ❌ Non |
| Config incluse | ✅ Oui (actions) | ✅ `config/install/` | ✅ Profile config |
| Dépendances | Recipes parentes | `dependencies:` | Profile dependencies |
| Usage | Appliquer un pattern sur un site existant | Fonctionnalité permanente | Nouveau site from scratch |

## Drupal Recipes et `config/optional/`

```
mon_module/config/
├── install/          # Importé TOUJOURS à l'activation du module
│   └── mon_module.settings.yml
└── optional/         # Importé SEULEMENT si les dépendances sont disponibles
    └── views.view.mon_articles.yml   # Import seulement si views est actif
    └── field.field.node.article.field_mon_champ.yml  # Si node + article existent
```

`config/optional/` est l'équivalent "déclaratif" des Config Actions dans les modules.

## Anti-Patterns Recipes

| ❌ | ✅ | Raison |
|----|----|--------|
| Recipe sans idempotence (double apply casse le site) | `createIfNotExists` pour toute création | Les recipes peuvent être appliquées plusieurs fois |
| Mettre des secrets dans recipe.yml | Variables d'env ou Key module | recipe.yml est dans git |
| Recipe qui installe 30 modules | Séparer en Recipes composables | Debugging difficile, trop couplé |
