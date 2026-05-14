# Leçons — drupal-config

Bugs et pièges découverts en projet réel. Mis à jour après chaque incident.

---

## Comment ajouter une leçon

Après chaque incident lié à la config :
1. Identifier si la cause racine est documentée dans ce skill
2. Ajouter une entrée ci-dessous avec date + symptôme + cause + prévention
3. Corriger le fichier source si nécessaire
4. Ajouter une ligne dans `CHANGELOG.md`

---

## 2026-05-14 — Création du skill

Pièges courants documentés dès la v1 :

### UUID du site — "Site UUID does not match" à chaque réinstallation
- **Symptôme :** `drush cim` échoue avec "Site UUID in source storage does not match"
- **Cause :** UUID différent entre les fichiers YAML (site source) et la DB du site cible (réinstallation fraîche ou site différent)
- **Correct :** `ddev drush config:set system.site uuid "UUID-DU-YAML"` puis `drush cim`
- **Prévention :** Dans le README du projet, documenter cet UUID et l'étape de setup initial. Automatiser dans un script `make init` ou `ddev start`

### `$config_directories['sync']` supprimé D9 — Erreur silencieuse
- **Symptôme :** Le sync directory est ignoré, `drush cex` exporte ailleurs
- **Cause :** `$config_directories['sync']` a été supprimé en D9 — la syntaxe D8 est ignorée silencieusement
- **Correct :** `$settings['config_sync_directory'] = '../config/sync';`
- **Prévention :** À vérifier systématiquement lors d'un upgrade D8→D9

### `drush cim` sans `drush cex` — Perte silencieuse de modifications locales
- **Symptôme :** Des modifications faites en local via l'UI disparaissent après `drush cim`
- **Cause :** La config "Only in active" (modifiée en local mais non exportée) est écrasée par l'import
- **Correct :** Toujours `drush cst` avant `drush cim` — exporter ce qui est "Only in active" si voulu
- **Prévention :** Règle d'or n°2 — jamais de cim sans cex préalable

### `hook_update_N` pour du code qui dépend de la config fraîchement importée
- **Symptôme :** L'update_N lit une config qui n'existe pas encore (la config est importée après updb)
- **Cause :** Dans `drush deploy`, l'ordre est `updb` PUIS `cim` — les hook_update voient l'ancienne config
- **Correct :** Utiliser `hook_deploy_N` qui s'exécute après `drush cim`
- **Prévention :** Règle de décision : si le code accède à une config qui vient d'être ajoutée → `hook_deploy_N`

### Config Ignore trop large en production — Mises à jour bloquées
- **Symptôme :** Des mises à jour de config déployées en git ne s'appliquent jamais en prod
- **Cause :** `config_ignore` configuré avec un pattern trop large (ex: `system.*`) qui bloque toutes les configs du système
- **Correct :** Affiner les patterns — cibler exactement les configs éditables (ex: `system.menu.main`)
- **Prévention :** Tester `drush config:status` après chaque changement de config_ignore pour vérifier que les bonnes configs passent

### Config Split — Oublier d'exporter les deux splits
- **Symptôme :** En production, les modules dev s'activent parce que le split dev est dans le config/sync commun
- **Cause :** `drush cex` exporté avec le split dev actif mais non configuré — la config des modules dev atterrit dans config/sync/ au lieu de config/dev/
- **Correct :** Configurer config_split AVANT le premier cex. Vérifier que `core.extension.yml` dans config/sync/ ne contient pas `devel`, `kint`, etc.
- **Prévention :** Après configuration de config_split, inspecter git diff et vérifier que les modules dev sont bien dans le bon répertoire de split

### Suppression d'un module en cascade — Views supprimées inopinément
- **Symptôme :** Après `drush pm:uninstall mon_module`, plusieurs Views disparaissent
- **Cause :** Les Views avaient des dépendances `config:` vers des config du module désinstallé
- **Correct :** Avant toute désinstallation, lancer `drush pm:uninstall mon_module --no` pour voir l'impact
- **Prévention :** Toujours simuler avant de désinstaller. Exporter la config avant de désinstaller.

### `drush config:get` retourne l'override, pas la valeur DB réelle
- **Symptôme :** `drush config:get system.site name` retourne '[LOCAL]' alors que la DB contient autre chose
- **Cause :** `drush config:get` retourne la valeur avec les overrides `$config[]` de settings.php appliqués
- **Correct :** `\Drupal::config('system.site')->getOriginal('name', FALSE)` — le second paramètre `FALSE` est obligatoire pour court-circuiter les overrides. Sans `FALSE`, `getOriginal()` applique encore les overrides (comportement par défaut).
- **Prévention :** Signature complète : `getOriginal(string $key, bool $apply_overrides = TRUE)` — toujours passer `FALSE` pour la valeur DB pure

### UUID d'objet config différent entre envs — Conflit silencieux lors de `drush cim`
- **Symptôme :** `drush config:status` montre "Different" sur un bloc ou une View que personne n'a touché ; `drush cim` réimporte en boucle ou génère des erreurs
- **Cause :** L'objet a été recréé en production (via UI) et Drupal lui a attribué un nouvel UUID — différent de celui dans le YAML de git. C'est un conflit d'UUID d'OBJET (pas de site).
- **Correct :** 
  1. Récupérer l'UUID de l'objet en prod : `drush config:get block.block.mon_bloc uuid`
  2. Mettre à jour le YAML de git avec cet UUID : éditer `config/sync/block.block.mon_bloc.yml`
  3. Ou supprimer l'objet en prod et laisser `drush cim` le recréer avec le bon UUID
- **Prévention :** Ne jamais recréer un objet de config en prod — toujours passer par git + `drush cim`

### Chemin config/sync différent selon les projets — script d'extraction échoue
- **Symptôme :** `drush cex -y` exporte dans le mauvais dossier, ou le script drupal-obsidian ne trouve pas les YAML
- **Cause :** La convention `config/sync/` n'est pas universelle — certains projets utilisent `config/default/sync/`, `config/config/`, etc.
- **Correct :** Vérifier `settings.php` pour lire la valeur réelle de `$settings['config_sync_directory']`
- **Prévention :** Pour drupal-obsidian, toujours utiliser `--config $(grep config_sync_directory settings.php | awk -F"'" '{print $2}')` ou lire le settings.php avant de lancer le script

### Config importée mais module désactivé — Import silencieux ignoré
- **Symptôme :** Un fichier YAML existe dans config/sync mais la config n'est jamais importée
- **Cause :** Le module dont dépend cette config n'est pas activé — Drupal ignore silencieusement les configs dont les dépendances module sont absentes
- **Correct :** Activer le module PUIS `drush cim`
- **Prévention :** Vérifier `drush config:status` après chaque cim — les items absents du résultat sont souvent des configs ignorées pour dépendances manquantes
