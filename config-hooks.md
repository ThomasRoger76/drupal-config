# Hooks de Config — Niveau Expert

## `hook_update_N` vs `hook_deploy_N` — Choisir le Bon

| | `hook_update_N` | `hook_deploy_N` (D9.3+) |
|--|----------------|------------------------|
| **Déclenché par** | `drush updb` | `drush deploy` (après `drush cim`) |
| **Quand s'exécute** | AVANT `drush cim` (dans `drush deploy`) | APRÈS `drush cim` |
| **Usage** | Mises à jour de schéma DB, migrations de données | Nettoyage après import config, réindexation |
| **Accès à la nouvelle config** | ❌ Pas encore importée | ✅ Config déjà en DB |
| **S'exécute** | Autant de fois que déclenché | **Une seule fois** |
| **Fichier** | `mon_module.install` | `mon_module.deploy.php` ou `mon_module.install` |

**Règle :** si ton code dépend d'une config qui vient d'être importée → `hook_deploy_N`. Sinon → `hook_update_N`.

---

## `hook_update_N` pour les Modifications de Config

```php
// mon_module.install

/**
 * Ajouter la clé 'enable_feature' à mon_module.settings.
 */
function mon_module_update_10001(): void {
  $config = \Drupal::configFactory()->getEditable('mon_module.settings');

  // Ajouter une nouvelle clé avec valeur par défaut
  if ($config->get('enable_feature') === NULL) {
    $config->set('enable_feature', TRUE)->save();
  }
}

/**
 * Renommer une clé de config (max_articles → max_items).
 */
function mon_module_update_10002(): void {
  $config = \Drupal::configFactory()->getEditable('mon_module.settings');

  $old_value = $config->get('max_articles');
  if ($old_value !== NULL) {
    $config
      ->set('max_items', $old_value)
      ->clear('max_articles')
      ->save();
  }
}

/**
 * Modifier la config d'un rôle Drupal.
 */
function mon_module_update_10003(): void {
  // Ajouter une permission à un rôle
  $role = \Drupal::entityTypeManager()
    ->getStorage('user_role')
    ->load('editor');

  if ($role && !$role->hasPermission('access mon_module')) {
    $role->grantPermission('access mon_module');
    $role->save();
  }
}

/**
 * Modifier la config d'une View (accès prudent).
 */
function mon_module_update_10004(): void {
  $config = \Drupal::configFactory()->getEditable('views.view.mon_module_articles');

  // Modifier un paramètre de la view
  $config->set(
    'display.default.display_options.items_per_page',
    12
  )->save();
}
```

---

## `hook_deploy_N` — Post-Import (D9.3+)

```php
// Fichier recommandé : mon_module.deploy.php (D9.3+)
// Fallback : mon_module.install (fonctionne mais mélange les responsabilités)
// NE PAS mettre dans mon_module.module

/**
 * Réindexer le search index après la mise à jour des champs.
 *
 * Ce hook s'exécute APRÈS drush cim, donc la config est déjà disponible.
 */
function mon_module_deploy_10001(): void {
  // Récupérer les indexes Search API et les marquer pour réindexation
  if (\Drupal::moduleHandler()->moduleExists('search_api')) {
    $indexes = \Drupal::entityTypeManager()
      ->getStorage('search_api_index')
      ->loadMultiple();

    foreach ($indexes as $index) {
      $index->reindex();
    }
  }
}

/**
 * Migrer des données vers un nouveau champ créé via config_import.
 *
 * La config du nouveau champ est déjà importée quand ce hook tourne.
 */
function mon_module_deploy_10002(array &$sandbox): string {
  // Batch migration
  // \Drupal::entityQuery() est dépréciée D9+ — utiliser getStorage()->getQuery()
  if (!isset($sandbox['total'])) {
    $sandbox['total']    = \Drupal::entityTypeManager()
      ->getStorage('node')
      ->getQuery()
      ->condition('type', 'article')
      ->accessCheck(FALSE)
      ->count()
      ->execute();
    $sandbox['progress'] = 0;
    $sandbox['limit']    = 50;
  }

  $nids = \Drupal::entityTypeManager()
    ->getStorage('node')
    ->getQuery()
    ->condition('type', 'article')
    ->accessCheck(FALSE)
    ->range($sandbox['progress'], $sandbox['limit'])
    ->execute();

  $nodes = \Drupal::entityTypeManager()->getStorage('node')->loadMultiple($nids);
  foreach ($nodes as $node) {
    // Utiliser le nouveau champ field_category (créé via la config importée)
    if ($node->hasField('field_category') && $node->field_category->isEmpty()) {
      $node->set('field_category', 'uncategorized');
      $node->save();
    }
    $sandbox['progress']++;
  }

  $sandbox['#finished'] = $sandbox['progress'] >= $sandbox['total']
    ? 1
    : $sandbox['progress'] / $sandbox['total'];

  return "Migré {$sandbox['progress']}/{$sandbox['total']} nœuds.";
}
```

---

## Config Schema — `config/schema/mon_module.schema.yml`

Sans schéma, ta config n'est pas typée, pas traduisible, et génère des avertissements.

```yaml
# config/schema/mon_module.schema.yml

# Simple Config Object
mon_module.settings:
  type: config_object
  label: 'Mon Module — Paramètres'
  mapping:
    max_items:
      type: integer
      label: 'Nombre maximum d''items'
    mode:
      type: string
      label: 'Mode d''affichage'
    enable_feature:
      type: boolean
      label: 'Activer la fonctionnalité'
    api_endpoint:
      type: uri
      label: 'URL de l''endpoint API'
    contact_email:
      type: email
      label: 'Email de contact'
    description:
      type: text
      label: 'Description (traduisible)'
    options:
      type: sequence
      label: 'Options'
      sequence:
        type: string
        label: 'Option'
    nested_mapping:
      type: mapping
      label: 'Configuration imbriquée'
      mapping:
        sub_key:
          type: string
          label: 'Clé imbriquée'

# Config Entity (si tu crées un type d'entité custom)
mon_module.mon_entite.*:
  type: config_entity
  label: 'Mon Entité'
  mapping:
    id:
      type: string
      label: 'Identifiant machine'
    label:
      type: label
      label: 'Libellé (traduisible via config_translation)'
    status:
      type: boolean
      label: 'Activé'
    description:
      type: text
      label: 'Description'
```

### Types de données disponibles dans le schéma

| Type | Exemple | Usage |
|------|---------|-------|
| `string` | `'hello'` | Chaîne générique |
| `label` | `'Mon label'` | String traduisible via Config Translation |
| `text` | Long texte | Texte multilignes traduisible |
| `boolean` | `true`/`false` | Booléen |
| `integer` | `42` | Entier |
| `float` | `3.14` | Décimal |
| `uri` | `'https://...'` | URL validée |
| `email` | `'a@b.com'` | Email validé |
| `sequence` | `[...]` | Tableau de valeurs |
| `mapping` | `{...}` | Objet imbriqué |
| `config_object` | Racine | Simple Config |
| `config_entity` | Racine | Config Entity |

---

## Config Events — Réagir au Cycle de Vie

```php
<?php
// src/EventSubscriber/ConfigEventSubscriber.php
namespace Drupal\mon_module\EventSubscriber;

use Drupal\Core\Config\ConfigCrudEvent;
use Drupal\Core\Config\ConfigEvents;
use Drupal\Core\Config\ConfigImporterEvent;
use Symfony\Component\EventDispatcher\EventSubscriberInterface;

final class ConfigEventSubscriber implements EventSubscriberInterface {

  public static function getSubscribedEvents(): array {
    return [
      ConfigEvents::SAVE    => 'onConfigSave',
      ConfigEvents::DELETE  => 'onConfigDelete',
      ConfigEvents::IMPORT  => 'onConfigImport',   // Après import complet
    ];
  }

  public function onConfigSave(ConfigCrudEvent $event): void {
    $config = $event->getConfig();

    // Réagir à la sauvegarde de system.site
    if ($config->getName() === 'system.site') {
      $new_name = $config->get('name');
      $old_name = $config->getOriginal('name');

      if ($new_name !== $old_name) {
        // Invalider le cache quand le nom du site change
        \Drupal::cache('render')->invalidateAll();
      }
    }

    // Réagir à la sauvegarde de notre propre config
    if ($config->getName() === 'mon_module.settings') {
      // Régénérer quelque chose...
    }
  }

  public function onConfigDelete(ConfigCrudEvent $event): void {
    if ($event->getConfig()->getName() === 'mon_module.settings') {
      // Nettoyage quand notre config est supprimée
    }
  }

  public function onConfigImport(ConfigImporterEvent $event): void {
    // Déclenché après que l'import complet a fini
    $importer = $event->getConfigImporter();
    $processed = $importer->getProcessedConfiguration();

    // $processed contient les configs créées/modifiées/supprimées
  }
}
```

```yaml
# mon_module.services.yml
services:
  mon_module.config_event_subscriber:
    class: Drupal\mon_module\EventSubscriber\ConfigEventSubscriber
    tags:
      - { name: event_subscriber }
```

---

## `hook_config_schema_info_alter` — Modifier le Schéma d'un Autre Module

```php
// Dans mon_module.module
function mon_module_config_schema_info_alter(array &$definitions): void {
  // Étendre le schéma d'un Content Type pour y ajouter des third_party_settings
  if (isset($definitions['node.type'])) {
    $definitions['node.type']['mapping']['third_party_settings']['mapping']['mon_module'] = [
      'type'    => 'mapping',
      'label'   => 'Mon Module settings',
      'mapping' => [
        'enabled' => ['type' => 'boolean', 'label' => 'Activé'],
        'mode'    => ['type' => 'string', 'label' => 'Mode'],
      ],
    ];
  }
}
```
