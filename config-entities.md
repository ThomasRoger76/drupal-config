# Simple Config vs Config Entities

## La Distinction Fondamentale

| | **Simple Config** | **Config Entity** |
|--|-------------------|-------------------|
| **Base PHP** | `ConfigObject` | `ConfigEntityBase` |
| **Lecture** | `\Drupal::config('system.site')` | `EntityTypeManager->getStorage('node_type')` |
| **Écriture** | `configFactory()->getEditable()->set()->save()` | `$entity->save()` |
| **YAML** | Un seul fichier par objet | Un fichier par instance |
| **UUID** | UUID du SITE (`system.site.yml`) | UUID propre à chaque instance |
| **Listable** | `configFactory()->listAll('prefix.')` | `$storage->loadMultiple()` |
| **CRUD** | ConfigFactory | EntityStorageInterface |
| **Exemples** | `system.site`, `node.settings`, `mon_module.settings` | Content Types, Views, Rôles, Vocabulaires |

---

## Exemples par Catégorie — Catalogue

### Simple Config (module settings)
```
system.site              → Nom du site, email, page 403/404
system.performance       → Agrégation CSS/JS, cache pages
node.settings            → Options globales du module node
user.settings            → Enregistrement, vérification email
contact.settings         → Paramètres contact
search.settings          → Configuration de recherche
image.settings           → Options de la librairie image
mon_module.settings      → Ta propre config custom
```

### Config Entities (instances individuelles)
```
# Content Types
node.type.article
node.type.page

# Views
views.view.frontpage
views.view.content

# Champs — stockage (partagé entre bundles)
field.storage.node.field_image
field.storage.node.field_tags

# Champs — instance sur un bundle
field.field.node.article.field_image
field.field.node.article.field_tags

# Affichage d'entité (Manage Display)
core.entity_view_display.node.article.default
core.entity_view_display.node.article.teaser

# Formulaire d'entité (Manage Form Display)
core.entity_form_display.node.article.default

# Rôles
user.role.anonymous
user.role.authenticated
user.role.editor
user.role.administrator

# Vocabulaires
taxonomy.vocabulary.tags
taxonomy.vocabulary.categories

# Menus
system.menu.main
system.menu.footer

# Styles d'image
image.style.thumbnail
image.style.medium
image.style.large

# Blocs configurés
block.block.olivero_branding
block.block.olivero_main_menu

# Workflows
workflows.workflow.editorial

# Responsive Image Styles
responsive_image.styles.hero
```

---

## CRUD Programmatique des Config Entities

```php
use Drupal\Core\Entity\EntityTypeManagerInterface;

// Injection recommandée
public function __construct(
  private readonly EntityTypeManagerInterface $entityTypeManager,
) {}

// CHARGER un Content Type
$type = $this->entityTypeManager->getStorage('node_type')->load('article');
if ($type) {
  echo $type->label();   // 'Article'
  echo $type->id();      // 'article'
}

// CHARGER une View
$view = $this->entityTypeManager->getStorage('view')->load('frontpage');

// CHARGER un Rôle
$role = $this->entityTypeManager->getStorage('user_role')->load('editor');

// CHARGER TOUS les Content Types
$all_types = $this->entityTypeManager->getStorage('node_type')->loadMultiple();

// CHARGER par condition
$vocabularies = $this->entityTypeManager->getStorage('taxonomy_vocabulary')->loadMultiple();

// LISTER les configs d'un type depuis ConfigFactory (alternative)
$config_names = \Drupal::configFactory()->listAll('node.type.');
// → ['node.type.article', 'node.type.page', 'node.type.blog']

// CRÉER un nouveau rôle programmatiquement
$role = \Drupal::entityTypeManager()->getStorage('user_role')->create([
  'id'    => 'redacteur',
  'label' => 'Rédacteur',
]);
$role->grantPermission('create article content');
$role->grantPermission('edit own article content');
$role->save();
// → génère user.role.redacteur.yml lors du prochain drush cex

// MODIFIER un Content Type existant
$type = $this->entityTypeManager->getStorage('node_type')->load('article');
$type->set('description', 'Nouvelles description.');
$type->save();
```

---

## Gestion des Dépendances — Cascade de Suppression

### Comment Drupal calcule les dépendances

Quand tu supprimes une config, Drupal supprime en cascade toutes les configs qui en dépendent :

```
Suppression de node.type.article
  → Supprime field.field.node.article.*     (champs de ce CT)
  → Supprime core.entity_view_display.node.article.*
  → Supprime core.entity_form_display.node.article.*
  → Supprime views.view.* (Views qui requièrent ce type)
  → Supprime system.menu.main (si des liens pointent vers ce CT)
```

### Vérifier les dépendances avant suppression

```php
// Trouver tout ce qui dépend d'une config
$dependent_configs = \Drupal::service('config.manager')
  ->findConfigEntityDependents('config', ['node.type.article']);

foreach ($dependent_configs as $dep) {
  echo $dep->getConfigDependencyName() . "\n";
}
```

```bash
# Via Drush — voir ce qui sera supprimé
docker compose exec php drush pm:uninstall mon_module --simulate    # Dry-run : affiche ce qui serait supprimé sans exécuter
```

### Désinstaller un module sans perdre la config

```bash
# 1. Vérifier les dépendances
docker compose exec php drush pm:uninstall mon_module --no

# 2. Exporter la config du module avant désinstallation
docker compose exec php drush config:export
git diff config/sync/

# 3. Supprimer les config entities du module manuellement si nécessaire
docker compose exec php drush config:delete mon_module.settings
docker compose exec php drush config:delete views.view.mon_module_view

# 4. Désinstaller
docker compose exec php drush pm:uninstall mon_module -y
docker compose exec php drush cex -y   # Exporter l'état final
```

---

## Installer la Config depuis un Module Custom

### Au premier install — `config/install/`

```
mon_module/
├── config/
│   ├── install/           # Importé automatiquement à l'activation du module
│   │   ├── mon_module.settings.yml
│   │   └── views.view.mon_module_articles.yml
│   └── optional/          # Importé seulement si les dépendances sont disponibles
│       └── core.entity_view_display.node.article.mon_module.yml
```

### Au réinstall ou forcer l'import

```php
// Dans hook_install ou hook_update_N
\Drupal::service('config.installer')->installDefaultConfig('module', 'mon_module');

// Ou installer une config optionnelle spécifique depuis les fichiers du module
$config_path = \Drupal::service('extension.list.module')->getPath('mon_module') . '/config/install';
$source = new \Drupal\Core\Config\FileStorage($config_path);
\Drupal::configFactory()
  ->getEditable('mon_module.settings')
  ->setData($source->read('mon_module.settings'))
  ->save();
```

---

## Différences selon l'Entité Config

| Entité | Storage ID | Exemple de config name |
|--------|-----------|----------------------|
| Content Type | `node_type` | `node.type.article` |
| Vocabulary | `taxonomy_vocabulary` | `taxonomy.vocabulary.tags` |
| Role | `user_role` | `user.role.editor` |
| View | `view` | `views.view.frontpage` |
| Image Style | `image_style` | `image.style.thumbnail` |
| Responsive Image Style | `responsive_image_style` | `responsive_image.styles.hero` |
| Block | `block` | `block.block.olivero_branding` |
| Menu | `menu` | `system.menu.main` |
| Workflow | `workflow` | `workflows.workflow.editorial` |
| Webform | `webform` | `webform.webform.contact` |
| Field Storage | `field_storage_config` | `field.storage.node.field_image` |
| Field Instance | `field_config` | `field.field.node.article.field_image` |
