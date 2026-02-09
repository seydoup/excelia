# CONSTITUTION — ExcelAI Platform

> Plateforme de traitement intelligent de fichiers Excel via des agents IA, avec paiement adaptatif par tâche.

**Version** : 1.0.0
**Date** : 2026-02-08
**Auteur** : TAPSAID
**Cadre de développement** : spec-kit / open-spec

---

## 1. Vision et Raison d'Être

ExcelAI est une plateforme SaaS destinée au marché burkinabè (et ouest-africain) qui permet à tout utilisateur — sans compétences techniques — de résoudre des problèmes complexes de traitement de données Excel en dialoguant avec des agents IA. Le service génère dynamiquement des scripts adaptés au besoin, les exécute de manière sécurisée, et retourne le résultat traité. Le paiement s'effectue par tâche via Orange Money ou Visa.

### 1.1 Problème Résolu

Les PME, ONG, administrations et professionnels indépendants au Burkina Faso manipulent quotidiennement des fichiers Excel pour des traitements récurrents (nettoyage, consolidation, transformation, reporting) sans disposer de développeurs ou d'outils adaptés. Les solutions existantes (freelances, formations) sont lentes, coûteuses ou inadaptées au contexte local.

### 1.2 Proposition de Valeur

La plateforme offre un accès instantané à un "analyste de données virtuel" capable de comprendre un besoin exprimé en langage naturel (français, mooré à terme), de générer et exécuter un script de traitement, et de livrer le résultat — le tout en quelques minutes et pour un coût proportionnel à la complexité.

---

## 2. Architecture Fonctionnelle

Le traitement d'une tâche suit un pipeline en trois phases séquentielles.

### 2.1 Phase 1 — Maîtrise d'Ouvrage Conversationnelle (MOA)

**Objectif** : Transformer un besoin vague en une spécification exploitable par un agent technique.

**Acteurs** : Client ↔ Agent MOA (LLM)

**Fonctionnement** :
L'agent MOA engage un dialogue structuré avec le client pour clarifier le besoin. Il pose des questions ciblées, reformule, propose des exemples et converge vers une spécification formalisée. La spécification produite contient la description du traitement attendu, le format d'entrée et de sortie, les règles métier et les cas limites identifiés.

**Livrables** : Un objet `TaskSpec` (JSON) validé par le client avant passage en phase 2.

**Contraintes** :
- Le nombre de tours de dialogue est plafonné (configurable, défaut : 20 tours) pour maîtriser les coûts LLM.
- L'agent doit être capable de refuser un besoin hors périmètre (pas de traitement Excel) et de rediriger.
- La spécification doit être compréhensible par un humain et par l'agent technique.

### 2.2 Phase 2 — Collecte et Calibration

**Objectif** : Recevoir les fichiers sources et, le cas échéant, des exemples de résultats attendus pour affiner la solution.

**Acteurs** : Client → Système (upload) → Agent Technique (LLM)

**Fonctionnement** :
Le client téléverse son ou ses fichiers Excel. L'agent technique analyse la structure (colonnes, types, volume, encodage), la compare à la spécification MOA et identifie les écarts. Si des exemples de sortie sont fournis, ils servent de tests de validation. L'agent peut poser des questions de clarification complémentaires.

**Livrables** : Un objet `TaskContext` enrichi (spec + métadonnées fichiers + exemples de validation).

**Contraintes** :
- Taille maximale par fichier : 50 Mo (configurable).
- Formats acceptés : `.xlsx`, `.xls`, `.csv`, `.tsv`.
- Les fichiers sont analysés en sandbox sans exécution de macros VBA.
- Les données client ne sont jamais utilisées pour l'entraînement de modèles.

### 2.3 Phase 3 — Génération, Exécution et Livraison

**Objectif** : Produire un script, l'exécuter sur les données, valider le résultat et le livrer au client.

**Acteurs** : Agent Technique (LLM) → Sandbox d'exécution → Client

**Fonctionnement** :
L'agent génère un script Python (bibliothèques autorisées : `openpyxl`, `pandas`, `xlsxwriter`, liste blanche extensible). Le script est exécuté dans un environnement sandboxé avec des limites de temps (60s défaut), de mémoire (512 Mo défaut) et sans accès réseau. Si des exemples de validation existent, le résultat est comparé automatiquement. En cas d'échec, l'agent peut itérer (3 tentatives max).

**Livrables** : Le fichier résultat téléchargeable + un résumé du traitement effectué.

**Contraintes** :
- Aucune exécution de code arbitraire côté client.
- Le script généré est auditable par le client (transparence).
- Le résultat est disponible au téléchargement pendant 72 heures puis supprimé.

---

## 3. Architecture Technique

### 3.1 Stack Technologique

| Couche | Technologie | Justification |
|---|---|---|
| Frontend | Next.js (React) | SSR pour performances sur réseaux lents, PWA pour mobile |
| Backend API | Node.js (Express/Fastify) ou Python (FastAPI) | Écosystème riche, compatible LLM SDKs |
| Base de données | PostgreSQL | Transactions ACID pour paiements, JSONB pour specs |
| File d'attente | Redis + BullMQ | Gestion asynchrone des tâches de traitement |
| Sandbox d'exécution | Docker containers éphémères | Isolation complète, limites CPU/RAM/temps |
| LLM Provider | API Anthropic (Claude) — principal ; OpenAI — fallback | Qualité de raisonnement, support francophone |
| Stockage fichiers | S3-compatible (MinIO self-hosted ou AWS S3) | Coût maîtrisé, rétention configurable |
| Paiement | Orange Money API (OMAPI) + passerelle Visa (CinetPay/PayDunya) | Couverture marché burkinabè |

### 3.2 Diagramme de Flux Simplifié

```
Client → [Frontend PWA]
             ↓
         [API Gateway]
             ↓
    ┌────────┴────────┐
    ↓                 ↓
[Agent MOA]    [Agent Technique]
  (LLM)            (LLM)
    ↓                 ↓
[TaskSpec]     [Script Python]
                      ↓
              [Sandbox Docker]
                      ↓
              [Résultat Excel]
                      ↓
              [Stockage S3] → Client (téléchargement)
```

### 3.3 Modèle de Données Principal

```
User {id, phone, email?, name, created_at}
Task {id, user_id, status, spec_json, context_json, cost_estimate, cost_final, paid, created_at, completed_at}
TaskFile {id, task_id, type(input|output|example), s3_key, size_bytes, created_at, expires_at}
Payment {id, task_id, provider(orange_money|visa), amount_xof, status, reference, created_at}
LLMUsage {id, task_id, phase, model, tokens_in, tokens_out, cost_usd, created_at}
```

---

## 4. Modèle Économique et Tarification

### 4.1 Principes

La tarification est adaptative et transparente. Le coût d'une tâche dépend de trois facteurs : la complexité estimée du traitement, le volume de données, et le coût réel des appels LLM consommés. Le client reçoit une estimation avant de payer et ne paie que si le résultat est livré.

### 4.2 Grille Indicative (en FCFA)

| Niveau | Description | Fourchette |
|---|---|---|
| Simple | Nettoyage, filtrage, reformatage basique | 500 – 2 000 FCFA |
| Intermédiaire | Consolidation multi-feuilles, formules complexes, croisements | 2 000 – 7 500 FCFA |
| Avancé | Analyse statistique, génération de rapports, transformations complexes | 7 500 – 25 000 FCFA |

### 4.3 Formule de Calcul

```
Coût = max(COÛT_PLANCHER, COÛT_LLM × MARGE_LLM + COÛT_COMPUTE × MARGE_COMPUTE)
```

Où `COÛT_PLANCHER` est le minimum par tâche (ex: 500 FCFA), `MARGE_LLM` couvre les frais API + marge (ex: ×2.5), et `MARGE_COMPUTE` couvre l'infrastructure d'exécution.

### 4.4 Modes de Paiement

**Orange Money** : Le client initie le paiement via USSD ou l'app Orange Money. La plateforme utilise l'API Orange Money (ou un agrégateur comme PayDunya/CinetPay) pour la collecte. Le traitement démarre après confirmation du paiement.

**Carte Visa** : Via une passerelle compatible PCI-DSS (CinetPay, PayDunya, Flutterwave). Principalement pour les clients entreprises ou internationaux.

**Flux de paiement** : Estimation affichée → Client accepte → Paiement collecté → Tâche exécutée → Si échec après 3 tentatives, remboursement automatique.

---

## 5. Sécurité et Confidentialité des Données

### 5.1 Principes Fondamentaux

Les données client sont traitées comme confidentielles par défaut. Aucune donnée n'est utilisée pour entraîner des modèles IA. Les fichiers sont chiffrés au repos (AES-256) et en transit (TLS 1.3). Les fichiers sont supprimés automatiquement après la période de rétention (72h par défaut).

### 5.2 Sandbox d'Exécution

Chaque script s'exécute dans un conteneur Docker éphémère avec les restrictions suivantes : pas d'accès réseau, système de fichiers en lecture seule (sauf répertoire de travail), limites CPU/RAM/temps, liste blanche de bibliothèques Python, destruction automatique après exécution.

### 5.3 Audit et Traçabilité

Chaque tâche produit un journal d'audit comprenant la spécification validée, le script généré (consultable par le client), les métriques d'exécution, et le statut final. Les journaux sont conservés 90 jours.

### 5.4 Conformité

La plateforme respecte la réglementation burkinabè sur la protection des données personnelles (loi n°010-2004/AN relative à la protection des données à caractère personnel) et les exigences de l'ARCEP pour les services numériques.

---

## 6. Agents IA — Spécifications

### 6.1 Agent MOA (Maîtrise d'Ouvrage)

**Rôle** : Analyste métier conversationnel.

**Comportement** : L'agent accueille le client, identifie la nature du besoin, pose des questions de clarification structurées, propose des reformulations pour validation, et produit une spécification formelle. Il refuse poliment les demandes hors périmètre. Il communique en français simple, adapté au contexte local.

**Prompt System** : Défini dans `/specs/agents/moa-agent.prompt.md` — versionné et auditable.

**Modèle** : Claude Sonnet (rapport qualité/coût) pour la majorité des cas. Escalade vers Claude Opus pour les cas complexes détectés automatiquement.

### 6.2 Agent Technique

**Rôle** : Développeur Python spécialisé en traitement de données tabulaires.

**Comportement** : L'agent reçoit la `TaskSpec` et le `TaskContext`, génère un script Python conforme aux contraintes de la sandbox, le teste mentalement contre les exemples fournis, et itère en cas d'échec d'exécution.

**Prompt System** : Défini dans `/specs/agents/tech-agent.prompt.md`.

**Modèle** : Claude Sonnet pour la génération standard. Claude Opus si la première génération échoue ou si la complexité estimée dépasse un seuil.

### 6.3 Agent Estimateur

**Rôle** : Évaluer la complexité d'une tâche et produire une estimation de coût.

**Comportement** : À partir de la `TaskSpec`, l'agent évalue le nombre probable de tours LLM nécessaires, le modèle requis, et le temps de calcul. Il produit une estimation en FCFA avec une fourchette (min/max).

**Modèle** : Claude Haiku (rapide et économique, suffisant pour l'estimation).

---

## 7. Structure du Projet (spec-kit / open-spec)

```
excelai-platform/
├── CONSTITUTION.md                  # Ce document
├── specs/
│   ├── overview.md                  # Vue d'ensemble technique
│   ├── agents/
│   │   ├── moa-agent.prompt.md      # Prompt system agent MOA
│   │   ├── tech-agent.prompt.md     # Prompt system agent technique
│   │   └── estimator-agent.prompt.md
│   ├── api/
│   │   ├── tasks.spec.md            # Endpoints gestion des tâches
│   │   ├── payments.spec.md         # Endpoints paiement
│   │   └── auth.spec.md             # Authentification (OTP SMS)
│   ├── models/
│   │   └── data-model.spec.md       # Schéma BDD détaillé
│   ├── sandbox/
│   │   └── execution.spec.md        # Specs sandbox Docker
│   └── integrations/
│       ├── orange-money.spec.md     # Intégration OMAPI
│       └── cinetpay.spec.md         # Intégration passerelle Visa
├── src/
│   ├── frontend/                    # Next.js PWA
│   ├── backend/                     # API (FastAPI ou Express)
│   ├── workers/                     # BullMQ workers (exécution tâches)
│   └── sandbox/                     # Dockerfile + scripts d'exécution
├── tests/
│   ├── unit/
│   ├── integration/
│   └── e2e/
├── infra/
│   ├── docker-compose.yml
│   ├── terraform/                   # Si déploiement cloud
│   └── nginx/                       # Reverse proxy config
└── docs/
    ├── onboarding.md
    ├── api-reference.md
    └── deployment.md
```

---

## 8. Contraintes Techniques Contextuelles (Burkina Faso)

### 8.1 Connectivité

La bande passante est limitée et instable dans de nombreuses zones. Le frontend doit être une PWA capable de fonctionner en mode dégradé. Les uploads doivent supporter la reprise (chunked upload avec résumabilité). Les fichiers résultats doivent être aussi légers que possible.

### 8.2 Appareils

La majorité des utilisateurs accèdent via smartphone Android moyen/bas de gamme. Le frontend doit être mobile-first, léger (bundle < 200 Ko gzippé), et fonctionnel sur Android 8+.

### 8.3 Paiement

Orange Money est le mode de paiement dominant. L'intégration doit supporter le flux USSD classique (pas uniquement l'app). Le montant minimum viable est 500 FCFA (contrainte psychologique et économique).

### 8.4 Langue

Interface en français. Les agents IA communiquent en français. Le support du mooré et du dioula est un objectif à moyen terme (Phase 2 du produit).

### 8.5 Hébergement

L'hébergement initial peut être sur un VPS en Europe ou en Afrique (OVH, Hetzner, ou fournisseur local si la latence est acceptable). La migration vers un cloud africain (par ex. Africa's Talking, MainOne) est envisageable à moyen terme.

---

## 9. Phases de Développement

### Phase 0 — MVP (8-12 semaines)

Périmètre : Un seul flux complet (MOA → Upload → Traitement → Livraison) avec paiement Orange Money uniquement. Un seul type de traitement (nettoyage/transformation basique). Interface web mobile-first minimale. Déploiement sur un VPS unique.

Objectif : Valider le concept avec 50-100 utilisateurs pilotes.

### Phase 1 — Consolidation (3-6 mois post-MVP)

Ajout du paiement Visa. Élargissement des types de traitement supportés. Amélioration de l'agent MOA (meilleure gestion de l'ambiguïté). Tableau de bord client (historique des tâches). Monitoring et alerting.

### Phase 2 — Croissance (6-12 mois post-MVP)

Support multilingue (mooré, dioula). API publique pour intégrations tierces. Abonnements pour clients réguliers (forfaits mensuels). Extension géographique (Mali, Côte d'Ivoire, Niger).

---

## 10. Principes de Développement

### 10.1 Standards de Code

Le code suit les conventions du framework spec-kit/open-spec. Chaque composant majeur dispose d'une spécification dans `/specs/` avant implémentation. Les revues de code sont obligatoires. La couverture de tests minimale est de 70% pour le backend et 50% pour le frontend.

### 10.2 Design Patterns Retenus

**Pipeline Pattern** pour le flux de traitement des tâches (Phase 1 → 2 → 3 avec transitions explicites et rollback possible).

**Strategy Pattern** pour les providers de paiement (Orange Money, Visa, futurs providers) afin de permettre l'ajout de nouveaux moyens de paiement sans modifier le code existant.

**Repository Pattern** pour l'accès aux données, séparant la logique métier de la persistance.

**Event-Driven Architecture** pour la communication entre services (création de tâche → estimation → paiement → exécution → notification), permettant le découplage et la scalabilité.

### 10.3 Principes Directeurs

**Simplicité d'abord** : Chaque décision technique privilégie la solution la plus simple qui résout le problème. La complexité n'est ajoutée que lorsqu'elle est justifiée par un besoin concret, pas par anticipation.

**Transparence** : Le client voit le coût estimé avant de payer, le script généré après traitement, et le détail de la consommation LLM. Aucune boîte noire.

**Résilience** : Le système gère gracieusement les échecs (LLM indisponible, script en erreur, paiement échoué) avec des mécanismes de retry, fallback et remboursement automatique.

**Frugalité** : Le choix du modèle LLM s'adapte à la complexité réelle de chaque tâche. On ne mobilise pas Claude Opus pour une tâche que Claude Haiku peut résoudre.

---

## 11. Métriques de Succès

| Métrique | Cible MVP | Cible Phase 1 |
|---|---|---|
| Taux de complétion des tâches | > 75% | > 90% |
| Temps moyen de traitement (bout en bout) | < 10 min | < 5 min |
| Satisfaction client (enquête post-tâche) | > 3.5/5 | > 4/5 |
| Coût LLM moyen par tâche | < 0.15 USD | < 0.10 USD |
| Taux de remboursement | < 20% | < 5% |
| Utilisateurs actifs mensuels | 50 | 500 |

---

## 12. Risques et Mitigations

| Risque | Impact | Probabilité | Mitigation |
|---|---|---|---|
| Coûts LLM supérieurs aux estimations | Élevé | Moyenne | Seuils stricts par tâche, modèle adaptatif, cache de patterns récurrents |
| Qualité insuffisante des scripts générés | Élevé | Moyenne | Itération automatique (3 essais), escalade de modèle, validation par exemples |
| Adoption faible (méconnaissance IA) | Moyen | Élevée | Campagne de démonstration ciblée, partenariats avec incubateurs locaux (CTIC, 2IE) |
| Indisponibilité API Orange Money | Moyen | Moyenne | File d'attente avec retry, mode Visa en fallback, gestion gracieuse de l'attente |
| Fuite de données client | Critique | Faible | Sandbox isolée, chiffrement, suppression automatique, audit |

---

## 13. Gouvernance du Projet

**Propriétaire** : TAPSAID

**Décisions techniques** : Documentées dans `/specs/` via le processus spec-kit (proposition → revue → validation → implémentation).

**Versioning** : SemVer pour l'API et les releases. Chaque modification de cette constitution incrémente sa version.

**Licence** : Propriétaire (TAPSAID). Les composants open-source utilisés sont listés dans `DEPENDENCIES.md` avec leurs licences respectives.

---

*Ce document est la source de vérité pour la vision, l'architecture et les principes directeurs de la plateforme ExcelAI. Toute décision de conception doit être cohérente avec cette constitution ou proposer un amendement formel.*
