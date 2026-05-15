# Fondamentaux — Active vs Staged Config

## Le Cycle de Vie de la Configuration

```
[Développeur modifie via UI]
         ↓
  [ACTIVE CONFIG — DB]   ←── drush cim ──   [STAGED CONFIG — YAML]
  (table `config`)                           (config/sync/*.yml)
         ↓                                          ↑
     drush cex ──────────────────────────────────────
         ↓
    [git commit → push → déploiement → drush cim]
```

**Active Config** = ce que Drupal utilise en ce moment (base de données, table `config`)  
**Staged Config** = les fichiers YAML dans `config/sync/` — la source de vérité Git

---

## Configurer le Sync Directory

### Dans `settings.php` (D9+ — syntaxe obligatoire)

```php
// settings.php — chemin hors du webroot recommandé
$settings['config_sync_directory'] = '../config/sync';

// ⚠️ Chemin variable selon les projets — exemples réels rencontrés :
// Projet "simple" (recommandé)
$settings['config_sync_directory'] = '../config/sync';
// Projet avec sous-dossier "default" (ex: CCI Le Mans)
$settings['config_sync_directory'] = '../config/default/sync';
// Chemin absolu alternatif
$settings['config_sync_directory'] = DRUPAL_ROOT . '/../config/sync';

// Avec un chemin par environnement
if (getenv('APP_ENV') === 'local') {
  $settings['config_sync_directory'] = '../config/sync';
} else {
  $settings['config_sync_directory'] = '/var/www/config/sync';
}
```

```php
// ❌ Ancienne syntaxe D8 — supprimée en D9
$config_directories['sync'] = 'config/sync';
```

**Structure de projet recommandée :**
```
mon-projet/
├── web/                          # Webroot (DRUPAL_ROOT)
│   ├── core/
│   ├── modules/
│   └── ...
├── config/
│   ├── sync/                     # Config principale (version contrôlée)
│   ├── dev/                      # Config Split pour dev (si utilisé)
│   └── prod/                     # Config Split pour prod (si utilisé)
├── vendor/
└── composer.json
```

Le répertoire `config/sync/` doit être hors du webroot pour éviter l'accès HTTP direct.

---

## Lire et Écrire la Config depuis PHP

### Lecture — Config Immutable (recommandé)

```php
// Service : config.factory — injectez ConfigFactoryInterface
$config = \Drupal::config('system.site');          // Lecture seule

$name      = $config->get('name');                 // Valeur d'une clé
$page_404  = $config->get('page.404');             // Clé imbriquée (notation point)
$all       = $config->getRawData();                // Tout le contenu YAML

// Config d'un module custom
$config = \Drupal::config('mon_module.settings');
$limit  = $config->get('max_items') ?? 10;
```

### Écriture — Config Mutable (avec injection)

```php
use Drupal\Core\Config\ConfigFactoryInterface;

// Injection dans un service ou controller
public function __construct(
  private readonly ConfigFactoryInterface $configFactory,
) {}

// Modifier et sauvegarder
$config = $this->configFactory->getEditable('mon_module.settings');
$config
  ->set('max_items', 25)
  ->set('mode', 'advanced')
  ->clear('deprecated_key')         // Supprimer une clé
  ->save();                         // OBLIGATOIRE — sans save() rien n'est persisté

// Supprimer entièrement une config
$this->configFactory->getEditable('mon_module.settings')->delete();
```

### Dans un hook (fichier `.module` — `\Drupal::` acceptable)

```php
function mon_module_install(): void {
  // Initialiser la config au premier install
  \Drupal::configFactory()
    ->getEditable('mon_module.settings')
    ->set('max_items', 10)
    ->set('mode', 'basic')
    ->save();
}
```

---

## Les Types de Stockage

| Stockage | Service | Où | Usage |
|----------|---------|-----|-------|
| `DatabaseStorage` | `config.storage` | Table `config` | Active config — lecture runtime |
| `FileStorage` | `config.storage.sync` | `config/sync/*.yml` | Staged config — source vérité |
| `CachedStorage` | Wrapper | RAM + DB | Cache de la config active |

**Le Config Factory** (`config.factory`) lit depuis `DatabaseStorage` avec cache.  
**Ne jamais écrire directement dans le FileStorage** — passer par `drush cex` ou l'API PHP.

---

## Portée de la Configuration

```php
// Config globale au site
\Drupal::config('system.site')           // Nom du site, email admin
\Drupal::config('system.performance')    // Agrégation CSS/JS, cache

// Config par module
\Drupal::config('node.settings')         // Paramètres du module node
\Drupal::config('user.settings')         // Paramètres des comptes

// Config par entité (Config Entities)
\Drupal::config('node.type.article')     // Le Content Type "article"
\Drupal::config('views.view.frontpage')  // La View "frontpage"

// Ce qui N'est PAS dans la config :
// → State API : données runtime volatiles (timestamps, cron)
// → Variables par utilisateur : user.data
// → Content : nodes, taxonomies, users
```

---

## Lister les Configs Disponibles

```php
// Lister toutes les configs d'un namespace
$node_types = \Drupal::configFactory()->listAll('node.type.');
// → ['node.type.article', 'node.type.page', 'node.type.blog']

// Lister toutes les configs
$all = \Drupal::configFactory()->listAll();

// Depuis Drush
// docker compose exec php drush config:list
// docker compose exec php drush config:list --prefix=node.type
```

---

## Active Config vs Override

Drupal applique les overrides de `settings.php` **à la lecture** — la valeur en DB reste inchangée :

```php
// settings.php
$config['system.site']['name'] = 'Mon Site [DEV]';

// En PHP
\Drupal::config('system.site')->get('name');                   // → 'Mon Site [DEV]' (override appliqué)
\Drupal::config('system.site')->getOriginal('name', FALSE);    // → 'Mon Site' (valeur DB pure, sans override)
\Drupal::config('system.site')->getOriginal('name');           // → 'Mon Site [DEV]' (override encore appliqué !)
// getOriginal($key, $apply_overrides = TRUE) — passer FALSE pour ignorer les overrides settings.php
```

`drush cex` exporte la valeur **originale** de la DB, pas l'override.  
`drush config:get system.site name` retourne la valeur **avec l'override** appliqué.  
`\Drupal::config('system.site')->getOriginal('name', FALSE)` retourne la valeur **pure DB** (sans override).
