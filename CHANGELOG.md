# Changelog — drupal-config

---

## v1.2 — 2026-05-16

**Bug critique corrigé :**
- `agents/config-doctor.md` Étape 2 — Fix UUID inversé : la commande `sed -i` écrasait les fichiers YAML git avec l'UUID de la DB prod. Corrigé dans la bonne direction : l'UUID du YAML git est mis dans la DB (`drush config:set system.site uuid "$YAML_UUID"`). Ajout d'un commentaire explicit `# ❌ NE PAS FAIRE`.

**Description frontmatter étendue :**
- Ajout : `config_split`, `drupal recipes`, `config_translation`, `config/optional/`, `multisite avec config_split`
- Garantit le déclenchement du skill sur ces cas d'usage couverts

**Quick Decision Table :**
- Nouvelle entrée : `ConfigEvents::SAVE / DELETE / IMPORT` via EventSubscriber (→ `config-hooks.md`)

---

## v1.1 — 2026-05-14

**Bugs corrigés :**
- `config.manager->getConfigFactory()` → `\Drupal::configFactory()` (méthode inexistante)
- `getOriginal('name')` → `getOriginal('name', FALSE)` (sans FALSE, les overrides s'appliquent encore)
- `\Drupal::entityQuery('node')` dépréciée D9+ → `getStorage('node')->getQuery()` (deux occurrences)
- `drush deploy` : "entity-updates + locale-update" retiré du commentaire (non inclus par défaut)

**Incohérences corrigées :**
- SKILL.md Quick Decision Table : ordre drush deploy "cim + updb" → "updb → cim → cr"
- `--no` présenté comme simulation → `--simulate` pour le dry-run, `--no` pour annuler
- `config_ignore.settings:` comme clé racine YAML → structure correcte avec `langcode:` au niveau racine
- Leçon `getOriginal()` dans lessons.md corrigée avec `FALSE` obligatoire

**Ajouts :**
- `hook_deploy_N` → `.deploy.php` recommandé en priorité (vs `.install`)
- Config Split : exemple YAML complet de `config_split.config_split.dev.yml`
- Leçon UUID d'objet config (distinct du UUID de site) dans lessons.md

---

## v1.0 — 2026-05-14

**Création initiale**

### Couverture

**`SKILL.md`**
- Les 3 Règles d'Or (Git source de vérité, jamais cim sans cex, UUID unique)
- Quick Decision Table (20 entrées)
- Anti-patterns critiques (8 entrées)
- Table versioning D8→D11 (`$config_directories` supprimé D9, `hook_deploy_N` D9.3+)
- Section Auto-Amélioration

**`config-fundamentals.md`**
- Schéma du cycle de vie Active → Staged → Active
- Configuration du sync directory (`$settings['config_sync_directory']`)
- Ancienne syntaxe D8 supprimée en D9 (avec avertissement)
- Lecture (immutable) / Écriture (mutable) avec injection de `ConfigFactoryInterface`
- Les 3 storages : DatabaseStorage, FileStorage, CachedStorage
- Portée de la config (global, module, entity)
- `listAll()` — lister les configs
- Override `$config` vs valeur DB (`getOriginal()`)

**`workflow-commands.md`**
- Référence complète : `cex`, `cim`, `config:status`, `config:get/set/delete/diff/edit/list`
- `drush deploy` — ordre d'exécution et avantages vs updb seul
- Lecture du résultat de `drush config:status` (Only in active/staging/Different)
- Workflow Git complet (développeur A → commit → développeur B → prod)
- Script de déploiement en production
- Import partiel `--partial` avec mise en garde
- Import programmatique depuis un module
- Commandes de diagnostic (diff entre environnements)

**`yaml-anatomy.md`**
- Anatomie complète d'un Simple Config YAML (`system.site.yml`)
- Anatomie complète d'un Config Entity YAML (`node.type.article.yml`, `field.field.node.article.field_image.yml`)
- Les deux UUIDs : UUID du SITE vs UUID de l'OBJET (tableau comparatif)
- Problème UUID du site : symptôme, 3 résolutions, script de setup
- Lire un diff — grille de décision (valider ou non)
- Bloc `dependencies:` — comportement à l'import et à la suppression
- Conventions de nommage des fichiers YAML (catalog complet)

**`environment-overrides.md`**
- `$config` dans settings.php — ce que ça fait / ne fait pas (tableau comparatif)
- Pattern `settings.local.php` — cache null, Twig debug, overrides locaux
- Module `config_split` — setup 3 splits (dev/prod/local), workflow, structure des répertoires
- Module `config_ignore` — configuration, cas d'usage, wildcards, mises en garde
- Module `config_readonly` — activation, enforcement en prod
- Pattern complet multi-environnements (local/staging/prod)

**`config-entities.md`**
- Tableau comparatif Simple Config vs Config Entity (API, UUID, YAML, CRUD)
- Catalogue complet des Config Entities par catégorie
- CRUD programmatique : load, loadMultiple, listAll, create, save
- Cascade de suppression : comment ça fonctionne, comment la vérifier
- Désinstaller un module sans perdre la config (procédure sécurisée)
- `config/install/` vs `config/optional/` — différence et usage
- Tableau des storage IDs par type d'entité

**`config-hooks.md`**
- `hook_update_N` vs `hook_deploy_N` — tableau décisionnel avec raison
- Exemples `hook_update_N` : ajout de clé, renommage, modification rôle, modification View
- Exemples `hook_deploy_N` : réindexation Search API, migration de données en batch
- Config Schema (`config/schema/mon_module.schema.yml`) — anatomie complète
- Tous les types de données du schéma (string, label, text, uri, email, sequence, mapping, etc.)
- Config Events : `SAVE`, `DELETE`, `IMPORT` avec EventSubscriber complet
- `hook_config_schema_info_alter` — étendre le schéma d'un autre module

**`lessons.md`**
- 8 leçons pré-remplies avec symptôme/cause/correct/prévention
- UUID "Site UUID does not match" — 3 résolutions
- `$config_directories` supprimé D9
- `hook_update_N` vs `hook_deploy_N` confusion
- Config Ignore trop large
- Config Split mal configuré
- Cascade de suppression inopinée
- `drush config:get` retourne l'override, pas la valeur DB
- Config ignorée silencieusement (module désactivé)

---

## Compatibilité Drupal

| Skill version | Drupal testé | Notes clés |
|--------------|-------------|------------|
| v1.0 | D9, D10, D11 | `$config_directories` supprimé D9, `hook_deploy_N` D9.3+, `drush deploy` D9+ |
