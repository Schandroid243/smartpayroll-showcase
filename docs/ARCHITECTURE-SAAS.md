# Architecture SaaS — Transformation multi-tenant de SmartPayroll

> Version condensée, à but de démonstration, du document d'architecture interne. Elle couvre les décisions structurantes de la transformation d'un logiciel on-premise mono-poste en plateforme SaaS multi-tenant hébergée.

## 1. Le point de départ et la contrainte de marché

SmartPayroll était distribué **on-premise** : un binaire par client, une base MySQL locale, une licence chiffrée hors ligne contrôlant l'organisation active. Faire évoluer ce modèle vers un SaaS hébergé posait une question centrale, propre au marché visé : **comment facturer un abonnement récurrent en RDC ?**

La réponse occidentale par défaut — carte bancaire + prélèvement automatique — ne s'applique pas : le mobile money y fonctionne en **paiement poussé** (le client confirme lui-même chaque transaction depuis son téléphone), sans mandat de prélèvement automatique fiable. Cette contrainte a orienté l'ensemble du modèle de facturation, détaillé en §4.

## 2. Décisions d'architecture actées

| Décision | Pourquoi | Conséquence |
|---|---|---|
| **Monolithe modulaire conservé** — aucune extraction en microservices | Une seule infrastructure à opérer, cohérent avec une équipe restreinte | Les nouveaux modules SaaS (abonnements, wallet, entitlements, admin plateforme) vivent dans le même dépôt, avec les mêmes conventions que le code existant |
| **Hébergement centralisé (Railway)**, abandon de la distribution de binaires | Le reverse engineering d'un binaire distribué au client est un risque impossible à maîtriser par la seule obfuscation | Le pointage GPS et toutes les fonctionnalités dépendent désormais d'un accès internet réel côté client, plus seulement d'un réseau local |
| **Facturation par wallet prépayé**, pas par prélèvement récurrent | Cohérent avec le paiement poussé du mobile money local | Le renouvellement d'abonnement débite un solde interne déjà crédité — jamais un compte ou une carte externe |
| **Stockage objet (S3-compatible)** plutôt que disque local | Un hébergeur PaaS ne garantit pas la persistance d'un disque local à travers les redéploiements ; les preuves de pointage ont une valeur d'audit | Les fichiers (documents RH, preuves de pointage) sont référencés par clé d'objet, jamais par chemin disque |
| **Séparation stricte Platform Admin / Tenant**, au niveau table, JWT et routes | Empêcher qu'un bug de contrôle d'accès transforme un admin d'organisation en admin de toute la plateforme | Toute route `/api/platform/*` exige un JWT distinct (audience et secret différents) — jamais le même token qu'un client |
| **Devise de référence CDF** pour le wallet, USD à titre indicatif commercial uniquement | Le mobile money en RDC transacte en CDF ; éviter la complexité d'un double grand-livre multi-devises dès le départ | Le taux de change n'est tracé que pour du reporting interne, jamais pour la logique de débit |
| **Retrait complet du système de licence locale** une fois confirmée l'absence de déploiement on-premise actif | Plus de code distribué au client = plus besoin de protéger un artefact hors-ligne | La vérification d'accès se fait désormais en base, en temps réel, à chaque requête — sans repli legacy |

## 3. Vue d'ensemble du système cible

```text
                         Internet (obligatoire désormais)
                                    │
                    ┌───────────────┴────────────────┐
                    │           Un seul projet         │
                    │                                  │
   ┌────────────────┼──────────────────────────────────┼───────────────┐
   │                │                                  │               │
   ▼                ▼                                  ▼               ▼
 web             worker                              MySQL           Redis
 (API Express   (BullMQ : paie,                     (managé)        (managé)
  + build       renouvellement                                          │
  React         d'abonnement,                                          ├── Cache
  statique)     webhooks billing,                                       ├── Rate limiting
                réconciliation)                                        └── File de jobs
   │
   ├── /api/*                        → routes tenant (paie, présences, RH…)
   ├── /api/wallet/*                 → top-up mobile money, solde, historique
   ├── /api/webhooks/billing/:provider → callbacks de l'agrégateur mobile money
   └── /api/platform/*               → réservé à l'opérateur de la plateforme
                    │
                    ▼
        Stockage objet externe (S3-compatible)
        documents employés, preuves de pointage
```

Un seul déploiement, plusieurs organisations. L'isolation est **logique** (modules, tables, routes, `organizationId` propagé depuis le JWT), pas infrastructurelle — un choix de simplicité opérationnelle assumé pour la taille d'équipe et de trafic visée, avec un chemin d'évolution clair si le besoin change (voir §7).

## 4. Facturation : wallet prépayé et cycle de vie de l'abonnement

Chaque organisation dispose d'un **wallet** (solde interne en CDF) et d'un **abonnement** rattaché à un plan. Le wallet a un **point d'entrée unique** pour toute modification de solde — aucune autre partie du code n'a le droit d'écrire directement dessus, ce qui garantit que chaque mouvement est tracé et cohérent.

**Machine à états de l'abonnement :**

```text
TRIAL ──────────► ACTIVE       (premier crédit suffisant, débit réussi)
TRIAL ──────────► CANCELED     (fin d'essai sans paiement, ou résiliation)

ACTIVE ─────────► PAST_DUE     (solde insuffisant au renouvellement)
PAST_DUE ───────► ACTIVE       (recharge reçue avant la fin de la période de retard)
PAST_DUE ───────► GRACE_PERIOD (après un délai de retard non résolu)
GRACE_PERIOD ───► SUSPENDED    (après un délai de grâce cumulé non résolu)
SUSPENDED ──────► ACTIVE       (recharge + débit réussi)

(tout état) ────► CANCELED     (résiliation volontaire)
```

Un accès en lecture seule est maintenu même en `SUSPENDED` — suspendre une organisation ne bloque jamais la consultation de ses propres données, seulement les opérations de création/modification.

La logique de débit et de réactivation est centralisée dans une **fonction unique** qui verrouille l'abonnement puis le wallet, toujours dans cet ordre — garantissant *structurellement* l'absence d'interblocage, plutôt que de compter sur une discipline respectée à chaque site d'appel. Un top-up ordinaire (le cas le plus fréquent) ne verrouille jamais l'abonnement.

L'intégration avec l'agrégateur mobile money passe par une **interface commune** (`PaymentProvider`), découplant la logique métier du fournisseur choisi. Chaque webhook entrant est vérifié par signature puis dédupliqué par un identifiant d'événement unique avant tout traitement métier — traité de façon asynchrone, jamais en ligne dans le handler HTTP. Un job de réconciliation périodique complète le dispositif : la fiabilité des webhooks entrants depuis un agrégateur mobile money vers un serveur hébergé hors RDC n'est pas garantie à 100 %, donc le système ne dépend jamais du webhook seul.

## 5. Séparation Platform Admin / Tenant

L'opérateur de la plateforme (super-admin) dispose d'un compte, d'un JWT et d'un secret de signature **entièrement distincts** de ceux des organisations clientes — pas seulement un rôle différent sur le même compte. L'authentification plateforme exige un second facteur (TOTP) dès la création du compte, sans exception. Toute action de modification via les routes plateforme écrit une ligne d'audit **avant** de répondre au client, jamais après coup en tâche de fond.

## 6. Pointage de présence hors ligne

Le pointage se fait depuis le terrain, où la connectivité est intermittente. Le principe : enregistrer l'action **immédiatement côté client** (horodatage capturé à l'instant réel), puis synchroniser en différé sans perte ni doublon.

- **Idempotence** : chaque pointage en attente porte un identifiant généré côté client, dédupliqué côté serveur.
- **Horodatage à paliers** : l'écart entre l'horloge du client et celle du serveur est traité en trois niveaux (accepté silencieusement / marqué à valider manuellement / exclu du calcul de paie tant que non validé) plutôt qu'un seuil binaire — pour ne jamais faire perdre une journée de salaire à un employé réellement présent à cause d'un simple bug d'horloge.
- **Contrôle géographique côté client strictement indicatif** : jamais bloquant, le serveur reste l'autorité finale à la synchronisation.
- **Distinction erreur transitoire / rejet métier** lors de la synchronisation : une erreur réseau déclenche un nouvel essai avec délai croissant ; un rejet métier (hors zone, déjà pointé) passe immédiatement en état définitif, sans nouvel essai inutile.

## 7. Conformité et rétention des données

La plateforme traite des données personnelles d'employés sous l'angle du cadre légal RDC applicable à la protection des données. Principe appliqué : toute donnée personnelle est conservée le temps nécessaire à sa finalité, puis anonymisée ou supprimée — jamais indéfiniment sous forme identifiante. Un job récurrent audite les organisations résiliées et applique le cycle export → anonymisation → purge définitive selon des délais paramétrables, avec traçabilité complète de chaque étape.

## 8. Ce qui est délibérément hors périmètre pour l'instant

Certaines évolutions ont été identifiées mais volontairement repoussées, faute de besoin réel démontré à l'échelle actuelle : agrégats journaliers matérialisés pour le tableau de bord plateforme, traçage distribué (les métriques agrégées suffisent en l'état), présence en temps réel par organisation. Une architecture SaaS ne se juge pas seulement à ce qu'elle construit, mais aussi à ce qu'elle choisit consciemment de ne pas construire avant d'en avoir la preuve du besoin.

---

*Ce document est une version condensée, à but de démonstration, de la documentation d'architecture interne du projet.*
