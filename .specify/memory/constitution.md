<!--
  Sync Impact Report
  ===================
  Version change: 0.0.0 (unfilled template) → 1.0.0
  Modified principles: N/A (initial population from CONSTITUTION.md)
  Added sections:
    - Core Principles (5 principles extracted from CONSTITUTION.md §10.3, §5, §4)
    - Technical Constraints (from §3.1, §8, §2)
    - Development Workflow (from §10, §7, §9)
    - Governance (from §13)
  Removed sections: None
  Templates requiring updates:
    - .specify/templates/plan-template.md — ✅ no update needed (Constitution Check section is generic)
    - .specify/templates/spec-template.md — ✅ no update needed (structure-agnostic)
    - .specify/templates/tasks-template.md — ✅ no update needed (task format is generic)
    - .specify/templates/checklist-template.md — ✅ no update needed (generic)
    - .specify/templates/agent-file-template.md — ✅ no update needed (generic)
  Follow-up TODOs: None
-->

# ExcelAI Platform Constitution

## Core Principles

### I. Simplicité d'Abord
Chaque décision technique DOIT privilégier la solution la plus simple
qui résout le problème. La complexité n'est ajoutée que lorsqu'elle
est justifiée par un besoin concret, jamais par anticipation (YAGNI).
Toute complexité introduite DOIT être justifiée dans le Complexity
Tracking du plan d'implémentation.

### II. Transparence Totale
Le client DOIT voir le coût estimé avant de payer, le script généré
après traitement, et le détail de la consommation LLM. Aucune boîte
noire. Le code généré est auditable. Les journaux d'audit sont
conservés 90 jours.

### III. Sécurité et Confidentialité des Données
Les données client sont confidentielles par défaut. Aucune donnée
n'est utilisée pour entraîner des modèles IA. Les fichiers DOIVENT
être chiffrés au repos (AES-256) et en transit (TLS 1.3). Les
fichiers sont supprimés automatiquement après 72h (configurable).
Chaque script s'exécute dans un conteneur Docker éphémère sans accès
réseau, avec système de fichiers en lecture seule (sauf répertoire
de travail), limites CPU/RAM/temps, et liste blanche de
bibliothèques Python.

### IV. Frugalité LLM
Le choix du modèle LLM DOIT s'adapter à la complexité réelle de
chaque tâche. Claude Haiku pour l'estimation, Claude Sonnet pour le
traitement standard, Claude Opus uniquement pour les cas complexes
détectés automatiquement. Les coûts LLM par tâche DOIVENT rester
sous contrôle avec des seuils stricts configurables.

### V. Résilience et Tolérance aux Pannes
Le système DOIT gérer gracieusement les échecs (LLM indisponible,
script en erreur, paiement échoué) avec des mécanismes de retry
(3 tentatives max pour les scripts), fallback (OpenAI si Anthropic
indisponible, Visa si Orange Money échoue), et remboursement
automatique en cas d'échec définitif.

## Contraintes Techniques

- **Stack**: Next.js (frontend PWA), FastAPI ou Express (backend),
  PostgreSQL, Redis + BullMQ, Docker (sandbox), S3-compatible
  (stockage fichiers)
- **LLM**: Anthropic Claude (principal), OpenAI (fallback)
- **Paiement**: Orange Money API (principal), CinetPay/PayDunya
  (Visa, fallback)
- **Contexte Burkina Faso**: PWA mobile-first, bundle < 200 Ko
  gzippé, Android 8+, uploads résumables (chunked), interface en
  français, support mooré/dioula prévu en Phase 2
- **Fichiers**: Max 50 Mo par fichier, formats .xlsx/.xls/.csv/.tsv,
  aucune exécution de macros VBA
- **Sandbox**: Timeout 60s, RAM 512 Mo, pas d'accès réseau,
  bibliothèques Python en liste blanche (openpyxl, pandas,
  xlsxwriter)
- **Tests**: Couverture minimale 70% backend, 50% frontend
- **Conformité**: Loi n°010-2004/AN (protection des données
  personnelles, Burkina Faso), exigences ARCEP

## Workflow de Développement

- **Pipeline en 3 phases**: MOA conversationnelle → Collecte et
  calibration → Génération, exécution et livraison
- **Design Patterns**: Pipeline Pattern (flux de tâches), Strategy
  Pattern (providers paiement), Repository Pattern (accès données),
  Event-Driven Architecture (communication inter-services)
- **Processus spec-kit**: Proposition → Revue → Validation →
  Implémentation. Chaque composant majeur dispose d'une
  spécification dans `/specs/` avant implémentation.
- **Revues de code obligatoires**
- **Versioning**: SemVer pour l'API et les releases
- **Structure projet**: Frontend (Next.js PWA) + Backend (API) +
  Workers (BullMQ) + Sandbox (Docker) — voir CONSTITUTION.md §7

## Governance

Cette constitution est la source de vérité pour la vision,
l'architecture et les principes directeurs de la plateforme ExcelAI.
Toute décision de conception DOIT être cohérente avec cette
constitution ou proposer un amendement formel.

Les amendements suivent le processus suivant :
1. Proposition documentée avec justification
2. Revue et validation par le propriétaire (TAPSAID)
3. Mise à jour de la constitution avec incrémentation de version
4. Propagation des changements aux templates et specs dépendants

Le versioning de la constitution suit SemVer :
- MAJOR : Changements incompatibles (suppression/redéfinition de
  principes)
- MINOR : Ajout de principe ou expansion significative
- PATCH : Clarifications, corrections de formulation

Propriétaire : TAPSAID
Licence : Propriétaire (TAPSAID)

**Version**: 1.0.0 | **Ratified**: 2026-02-08 | **Last Amended**: 2026-02-09
