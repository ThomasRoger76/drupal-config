# config/optional/ — Configuration Conditionnelle

## Principe : install vs optional

```
mon_module/config/
├── install/          # Importé TOUJOURS à l'activation du module
│   ├── mon_module.settings.yml         # Paramètres du module
│   └── user.role.mon_role.yml          # Rôle créé par le module
└── optional/         # Importé SEULEMENT si les dépendances sont satisfaites
    ├── views.view.mon_articles.yml                          # Si views actif
    ├── field.field.node.article.field_mon.yml               # Si node + type article existent
    ├── core.entity_form_display.node.article.default.yml   # Si article + field_mon existent
    └── core.entity_view_display.node.article.default.yml   # Pareil
```

**Règle fondamentale :** si une config dans `install/` dépend d'un autre module ou d'une entité qui pourrait ne pas exister, la déplacer dans `optional/`.

---

## Structure des dépendances dans config/optional/

```yaml
# views.view.mon_articles.yml — dans config/optional/
uuid: 12345678-1234-1234-1234-123456789012
langcode: fr
status: true
dependencies:
  module:
    - node      # Ce module doit être actif
    - views     # Views doit être actif
  config:
    - node.type.article   # Ce type de contenu doit exister
id: mon_articles
label: 'Mes articles'
# ... reste de la config Views
```

```yaml
# field.field.node.article.field_date_evenement.yml — dans config/optional/
uuid: abcdef12-abcd-abcd-abcd-abcdef123456
langcode: fr
status: true
dependencies:
  config:
    - field.storage.node.field_date_evenement   # Le storage doit exister (dans install/)
    - node.type.article                          # Le type article doit exister
  module:
    - node
    - datetime
entity_type: node
bundle: article
field_name: field_date_evenement
label: "Date de l'événement"
required: false
```

---

## Import manuel de config/optional/

```bash
# La config/optional/ est importée automatiquement lors de l'activation du module
# SI toutes les dépendances sont satisfaites à ce moment.

# Si les dépendances sont satisfaites après l'activation (ex. Views installé ensuite),
# importer manuellement :
docker compose exec php drush config:import \
  --source=web/modules/custom/mon_module/config/optional \
  --partial -y

# Vérifier ce qui serait importé (dry run) :
docker compose exec php drush config:import \
  --source=web/modules/custom/mon_module/config/optional \
  --partial \
  --preview -y
```

---

## hook_install() pour les cas complexes

```php
// mon_module.install

use Drupal\Core\Config\FileStorage;

/**
 * Implements hook_install().
 */
function mon_module_install(): void {
  // Importer une config optional spécifique si Views est actif
  if (\Drupal::moduleHandler()->moduleExists('views')) {
    _mon_module_import_optional_config('views.view.mon_articles');
  }

  // Importer si un type de contenu spécifique existe
  $node_types = \Drupal::entityTypeManager()
    ->getStorage('node_type')
    ->loadMultiple();

  if (isset($node_types['article'])) {
    _mon_module_import_optional_config('field.field.node.article.field_mon');
    _mon_module_import_optional_config('core.entity_view_display.node.article.default');
  }
}

/**
 * Importe une config optional spécifique.
 */
function _mon_module_import_optional_config(string $config_name): void {
  $config_path = \Drupal::service('extension.list.module')
    ->getPath('mon_module') . '/config/optional';

  $source = new FileStorage($config_path);
  $data   = $source->read($config_name);

  if ($data === FALSE) {
    \Drupal::logger('mon_module')->warning(
      'Config optional introuvable : @name',
      ['@name' => $config_name]
    );
    return;
  }

  \Drupal::service('config.installer')->installOptionalConfig(
    $source,
    ['config' => $config_name]
  );
}
```

---

## hook_update_N() pour la config optional en mise à jour

```php
// mon_module.install

/**
 * Importer la View des articles si Views a été activé depuis l'install.
 */
function mon_module_update_9001(): void {
  if (!\Drupal::moduleHandler()->moduleExists('views')) {
    // Views pas actif — noter dans les logs et passer
    \Drupal::logger('mon_module')->notice(
      'Views non actif — ignorer l\'import de views.view.mon_articles.'
    );
    return;
  }

  // Importer la config optional maintenant que Views est actif
  $config_path = \Drupal::service('extension.list.module')
    ->getPath('mon_module') . '/config/optional';

  $source = new FileStorage($config_path);

  \Drupal::service('config.installer')->installOptionalConfig(
    $source,
    ['config' => 'views.view.mon_articles']
  );
}
```

---

## Pipeline CI/CD — Ordre correct des opérations

```yaml
# .gitlab-ci.yml — stage deploy

deploy:
  stage: deploy
  script:
    # 1. Mettre à jour le code (pull / image Docker)
    - docker compose pull
    - docker compose up -d --no-deps php

    # 2. drush deploy = updb + cim + cr (ordre Drupal recommandé)
    # updb : mise à jour DB d'abord (les hooks peuvent dépendre de l'ancienne config)
    # cim  : import config (les modules sont déjà à jour)
    # cr   : rebuild final
    - docker compose exec php drush deploy -y

    # 3. Migrations APRÈS config
    # Les bundles et champs existent maintenant (créés par cim)
    - docker compose exec php drush migrate:import --tag=deploy -y || true

    # 4. Vider OPcache (validate_timestamps=0 en prod)
    - docker compose exec php drush php:eval "opcache_reset();"

    # 5. Cache final (au cas où les migrations auraient modifié la structure)
    - docker compose exec php drush cr
  only:
    - main
    - production
```

> `drush deploy` est équivalent à `drush updb -y && drush cim -y && drush cr` — mais dans le bon ordre avec vérifications d'erreurs.

---

## Cas d'usage fréquents

### View qui dépend d'un field custom

```
Problème : ma View référence field_date_pub — si la View est dans install/ et le field aussi,
           l'ordre d'import est non garanti.

Solution : mettre les DEUX dans optional/ avec les dépendances croisées déclarées.
           Drupal résoudra l'ordre automatiquement.
```

### Display mode conditionnel (jsonapi_extras)

```
Problème : core.entity_view_display.node.article.api dépend de jsonapi_extras.
           S'il est dans install/, l'erreur survient si jsonapi_extras n'est pas actif.

Solution : dans optional/ avec dependency: module: [jsonapi_extras, node]
```

### Anti-patterns config/optional/

| ❌ Erreur fréquente | ✅ Bonne pratique | Impact |
|--------------------|-----------------|--------|
| Config dans `install/` avec dépendance Views | Déplacer dans `optional/` | Erreur à l'activation si Views absent |
| Oublier la dépendance `config:` dans les YAML | Déclarer toutes les dépendances config | Import silencieusement ignoré |
| `drush cim` sans `--partial` pour l'optional | Utiliser `--partial` ou `config.installer` | Suppression de config non gérée |
| `drush deploy` avant `composer install` | `composer install` → `drush deploy` | Les modules nouveaux ne sont pas présents |
| Ignorer les erreurs migrate:import en CI | Logger les erreurs, alerter sur les critiques | Migrations silencieusement ratées |
