# Cas d'étude — D'un logiciel on-premise à un SaaS multi-tenant prêt pour la production

## Contexte

SmartPayroll a démarré comme un logiciel de paie **on-premise** : un binaire distribué au client (via `pkg`), une base MySQL portable installée localement, et une licence chiffrée hors ligne (AES-256-CBC) contrôlant l'organisation active et la durée de validité. Ce modèle fonctionnait pour un client à la fois, mais ne pouvait pas scaler : chaque nouveau client exigeait une installation manuelle, une mise à jour manuelle, et le code métier voyageait physiquement chez le client (surface d'attaque pour le reverse engineering).

L'objectif est devenu de transformer ce produit en **plateforme SaaS multi-tenant hébergée**, avec facturation par mobile money (le mode de paiement dominant en RDC, fonctionnant en paiement *poussé* plutôt qu'en prélèvement automatique). Ce chantier a nécessité de repenser en profondeur la résolution du tenant, la facturation, la sécurité, et l'observabilité — pas seulement d'ajouter des fonctionnalités par-dessus l'existant.

## La démarche : un audit avant la mise en production

Avant tout déploiement multi-tenant réel, le projet a fait l'objet d'un **audit de production-readiness** structuré, avec la posture suivante : *« Staff Engineer chargé du feu vert avant mise en production d'un SaaS multi-tenant, potentiellement des milliers d'utilisateurs, maintenance pluriannuelle. »*

Cet audit a couvert : l'architecture, le code, la sécurité, le modèle de données, la performance, les tests, le CI/CD, l'observabilité, le frontend, les workflows métier critiques, et l'historique Git — avec une règle explicite : **ne jamais présumer qu'un point non vérifiable (absence de base MySQL réelle disponible au moment de l'audit, par exemple) était correct.**

### Constat initial

> **Verdict : NOT PRODUCTION READY** pour le SaaS multi-tenant ciblé.

- **47 éléments de dette technique identifiés** : 6 critiques, 17 majeurs, 19 moyens, 5 mineurs.
- Une suite de **663 tests, tous verts** — mais entièrement basée sur des mocks, ne couvrant pas le scénario qui se produit réellement en environnement multi-tenant (organisation A qui accède aux données de l'organisation B).

**Top constats critiques :**

1. **Isolation multi-tenant non garantie** — l'organisation active était résolue depuis un fichier de licence global, pas depuis l'identité de l'utilisateur authentifié ; les filtres de données étaient conditionnels (`if (req.organizationId)`), donc *fail-open* en son absence.
2. **Fichiers RH exposés sans authentification** — dossier d'upload servi statiquement, accessible par URL directe, y compris entre employés de la même organisation.
3. **Perte irréversible de données de paie** — un cron de purge automatique supprimait en cascade des paiements après 30 jours d'inactivité d'un compte.
4. **Facturation non appliquée** — le statut d'abonnement (`SUSPENDED`, `CANCELED`) n'avait aucun effet réel sur l'accès à l'application.
5. **Schéma de base non reproductible** — `sequelize.sync()` activé par défaut au démarrage, chaîne de migrations cassée, aucune stratégie de sauvegarde vérifiée.

## La remédiation, phase par phase

Le plan de remédiation a été découpé en phases avec une règle simple : **ne jamais démarrer une phase avant que la précédente ne soit intégralement close.**

| Phase | Objectif | Statut |
|---|---|---|
| **Phase 0** | Blocages absolus avant toute mise en production : isolation tenant réelle (JWT, jamais un paramètre client), fermeture des fichiers publics, désactivation du hard-delete sur la paie, enforcement de l'abonnement, schéma de base reproductible, backups vérifiés | ✅ Fait |
| **Phase 1** | Durcissement production : health checks, arrêt propre (`graceful shutdown`), Redis non bloquant avec repli DB, CSP/HSTS, rate-limiting distribué, logs structurés avec `requestId`, Node LTS | ✅ Fait |
| **Phase 2** | Architecture : service worker BullMQ séparé du process web, unification des identifiants d'organisation (trois identifiants distincts coexistaient auparavant sans clé étrangère les reliant), extraction de services métier dédiés | ✅ Fait |
| **Phase 3** | Performance : workflow de correction/régularisation de paie, archivage applicatif des tables à forte croissance | ✅ Fait |
| **Phase 4** | Developer experience : CI complète (toutes branches, migrations rejouées sur base vierge, tests contre une vraie base MySQL/Redis, tests client, `npm audit`, détection de secrets, build Docker), réactivation progressive du linting strict, documentation à jour | ✅ Fait |

## Ce qu'une CI qui va enfin au bout révèle

Le point le plus instructif de la Phase 4 mérite d'être raconté en détail, parce qu'il illustre une réalité peu discutée : **une suite de tests verte ne garantit rien tant qu'elle n'a jamais tourné dans les conditions réelles qu'elle prétend vérifier.**

Avant cette phase, la CI ne tournait que sur la branche principale, sans base MySQL réelle ni Redis — les suites d'intégration destinées à s'exécuter contre une vraie base restaient inertes (`describe.skip`) faute d'environnement. Élargir la CI à toutes les branches/PR avec un vrai service MySQL a immédiatement révélé, **en une seule après-midi**, une série de bugs latents jamais rencontrés en pratique :

- Des migrations Sequelize antérieures au schéma de référence tentaient de recréer des colonnes/index déjà existants sur une base rejouée depuis zéro — jamais un problème en production (base jamais reconstruite entièrement), mais bloquant pour toute CI reproductible.
- Un ordre de migration incorrect entre deux tables liées par une clé étrangère, dû à un tri lexical de noms de fichiers trompeur (`wallet-` triait avant `wallets`).
- Une variable de session MySQL (`FOREIGN_KEY_CHECKS`) réinitialisée sur une connexion différente de celle utilisée pour l'opération suivante, à cause du pool de connexions — corrigé en épinglant explicitement une transaction unique.
- Un index Sequelize référençant le nom d'attribut JavaScript plutôt que le nom réel de la colonne en base — invisible tant que `sync()` n'avait jamais reconstruit le schéma correspondant contre un vrai moteur SQL.
- Une requête `INSERT ... SELECT *` entre deux tables construites indépendamment (l'une via les migrations, l'autre via le modèle Sequelize) supposant implicitement le même ordre de colonnes — corrigée avec une liste de colonnes explicite, ce qui élimine aussi un risque de régression future si une colonne est ajoutée d'un côté sans l'autre.
- Des `TRUNCATE TABLE` de test échouant systématiquement sur des tables référencées par une clé étrangère — un comportement MySQL structurel (bloqué dès que la contrainte existe, indépendamment du contenu des tables), jamais rencontré tant que le test correspondant n'avait jamais tourné contre un vrai moteur.

Chacun de ces bugs a été diagnostiqué à partir des logs bruts de CI, corrigé, couvert par un test de non-régression, puis vérifié par un nouveau passage de CI — jusqu'au vert.

## Résultat

- Isolation multi-tenant **prouvée** par une matrice de tests d'autorisation (organisation A / B / aucune) exécutée contre une vraie base MySQL, plutôt que par relecture de code ou mocks.
- Pipeline CI en 8 étapes : lint + vérification de types progressive, tests unitaires et d'intégration (dont contre une vraie base MySQL/Redis), tests client, audit de dépendances, détection de secrets dans l'historique Git, build de l'image Docker de production.
- Documentation d'architecture, runbooks de déploiement et de sauvegarde, `CHANGELOG` et politique de versionnement sémantique tenus à jour au fil des chantiers plutôt qu'en rattrapage ponctuel.
- Les 47 éléments de dette technique identifiés par l'audit initial (Phases 0 à 4) sont clos.

## Ce que cette démarche illustre

- **Auditer avant de faire confiance à une suite de tests verte** — 663 tests qui passent ne disent rien sur ce qu'ils ne couvrent pas.
- **Prioriser par risque réel, pas par facilité** — l'isolation multi-tenant (le chantier le plus lourd) a été traitée en Phase 0, avant les correctifs plus simples mais moins critiques.
- **Ne jamais fusionner une découverte de CI sans la comprendre.** Chaque bug révélé par l'élargissement de la CI a été root-causé (pas seulement contourné) et documenté avec la raison pour laquelle il n'avait jamais été détecté avant.
- **Une architecture SaaS se conçoit à partir des contraintes réelles du marché ciblé**, pas d'un modèle générique — le choix du wallet prépayé plutôt que du prélèvement automatique en est l'exemple le plus concret.
