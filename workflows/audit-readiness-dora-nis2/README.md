🛡️ Agent Audit Readiness DORA + NIS 2
> Agent IA n8n qui génère automatiquement des kits d'audit réglementaires complets (DORA + NIS 2) en moins de 30 secondes, avec mapping ITIL/COBIT intégré.
---
Le problème
Préparer un audit de conformité DORA ou NIS 2 prend aujourd'hui plusieurs jours par article :
Identifier les preuves à collecter
Structurer un plan de test d'audit
Préparer les questions d'entretien (RSSI, DPO, équipes IT)
Rédiger les templates de constats d'audit
Multiplié par 20+ articles DORA et 8+ articles NIS 2 pour une banque ou une entité critique... c'est une charge colossale pour les équipes conformité et audit interne.
Cet agent réduit ce travail de quelques jours à 30 secondes.
---
Démo rapide
L'auditeur remplit un formulaire web (article ciblé, framework, secteur, périmètre)
L'agent RAG retrouve les chunks réglementaires pertinents dans le vector store
Claude génère un kit d'audit structuré
Le kit est envoyé par email et enregistré dans Google Sheets
---
Architecture
Workflow A — Indexation (one-time setup)
```
Manual Trigger
    ↓
Code "Texte DORA"     → 8 articles avec metadata (framework: "DORA")
    ↓
Code "Texte NIS 2"    → 8 articles avec metadata (framework: "NIS2")
    ↓
Vector Store Insert   → memory key: "eu-regulations"
+ Default Data Loader → propagation 6 metadata par chunk
+ Text Splitter       → chunk size: 1000 / overlap: 200
    ↓
Embeddings OpenAI     → text-embedding-3-small
```
Résultat : 29 chunks indexés (16 articles DORA + NIS 2) avec metadata complètes
Workflow B — Agent (par requête)
```
Form Trigger (6 champs utilisateur)
    ↓
Vector Store Get Many   → top 15 chunks, prompt dynamique
+ Embeddings OpenAI     → même modèle que l'indexation
    ↓
Code "Preparer contexte audit"
  → filtrage metadata.framework (DORA ou NIS2)
  → top 8 chunks pertinents
  → diagnostic retrieval
    ↓
Agent IA "Audit Kit Generator" (Claude Sonnet)
  → plan de test + preuves + interview + findings + ITIL/COBIT
    ↓
Sheets append + Gmail send
```
---
Articles réglementaires indexés
DORA — Règlement (UE) 2022/2554
Article	Titre	Pilier DORA
Art. 5	Cadre de gestion des risques TIC	ICT Risk Management
Art. 8	Identification des fonctions critiques	ICT Risk Management
Art. 9	Protection et prévention	ICT Risk Management
Art. 10	Détection des incidents et anomalies	ICT Risk Management
Art. 11	Réponse et rétablissement	ICT Risk Management
Art. 17	Processus de gestion des incidents TIC	Incident Reporting
Art. 24	Programme de tests de résilience	Resilience Testing
Art. 28	Gestion des risques tiers TIC	Third-Party Risk
NIS 2 — Directive (UE) 2022/2555
Article	Titre	Domaine
Art. 3	Entités essentielles et importantes	Scope
Art. 20	Gouvernance et responsabilité du management	Governance
Art. 21	Mesures de gestion des risques cybersécurité (10 domaines)	Risk Management
Art. 23	Obligations de notification d'incident (24h/72h/1 mois)	Incident Reporting
Art. 24	Schémas européens de certification	Standards
Art. 25	Standardisation (ISO 27001, 22301, 27005)	Standards
Art. 27	Enregistrement des entités	Registration
Art. 32	Supervision et sanctions (jusqu'à 10M€ / 2% CA)	Enforcement
---
Kit d'audit généré
Pour chaque soumission de formulaire, l'agent produit un kit structuré contenant :
Section	Contenu
Plan de test	Procédures d'audit concrètes pour évaluer la conformité à l'article
Preuves attendues	Liste des documents, logs, configs, politiques à demander
Questions d'entretien	Questions à poser au RSSI, DPO, équipes IT/OPS
Template de constats	Format standard pour documenter les non-conformités
Mapping ITIL/COBIT	Correspondances vers les processus ITIL et contrôles COBIT
---
Tech Stack
Composant	Outil	Rôle
Orchestration	n8n (self-hosted)	Workflow automation
LLM	Claude Sonnet (Anthropic)	Génération du kit d'audit
Embeddings	text-embedding-3-small (OpenAI)	Vectorisation RAG
Vector Store	n8n Simple Vector Store	Stockage in-memory
Sortie 1	Google Sheets	Registre des kits générés
Sortie 2	Gmail	Livraison par email
---
Pattern clé : RAG avec metadata filtering
Ce projet introduit un pattern RAG avancé : indexation multi-corpus avec filtrage par metadata.
```javascript
// Chaque chunk est tagué avec son framework d'origine
metadata: {
  framework: "DORA",          // ou "NIS2"
  article_number: "9",
  article_title: "Protection et prévention",
  pillar: "ICT risk management",
  source: "Règlement (UE) 2022/2554",
  document_id: "dora-art-9"
}

// Au retrieval : filtre strict par framework
const filteredChunks = retrievedChunks.filter(item =>
  item.json.metadata.framework === targetFramework
);
```
Résultat : DORA et NIS 2 cohabitent dans le même vector store (`eu-regulations`) mais ne se mélangent jamais au retrieval.
---
Prérequis
n8n (cloud ou self-hosted) — version récente recommandée
Clé API Anthropic (Claude Sonnet)
Clé API OpenAI (embeddings uniquement — coût négligeable)
Compte Google avec accès Sheets + Gmail
---
Installation
Cloner ce repo
Importer `workflow-A-indexation-dora-nis2.json` dans n8n
Importer `workflow-B-audit-kit-generator.json` dans n8n
Configurer les credentials dans n8n (Anthropic, OpenAI, Google)
Exécuter Workflow A une fois (indexation ~3-5 secondes)
Activer Workflow B — le formulaire est disponible via l'URL n8n
> ⚠️ Le Simple Vector Store est in-memory : re-exécuter Workflow A après redémarrage de n8n.
> Pour une persistence en production, migrer vers Pinecone ou pgvector.
---
Résultats observés
Métrique	Valeur
Temps de génération d'un kit	~25-35 secondes
Chunks indexés	29 (16 articles)
Chunks retrieval par requête	15 → filtre → 8
Précision du filtrage framework	100% (zéro contamination cross-framework)
Coût par requête (tokens Claude)	~0.05-0.10 €
---
Roadmap V2
[ ] Migration vers Pinecone (persistence vector store)
[ ] Ajout ISO 42001 (AI Management System) dans le corpus
[ ] Ajout NIS 2 actes d'exécution (ENISA guidelines)
[ ] Authentification du formulaire (Basic Auth ou Cloudflare Access)
[ ] Export PDF du kit d'audit
[ ] Interface dashboard Retool ou Notion
---
Auteur
Zubair — Senior Infrastructure & Cybersecurity Project Manager  
12 ans d'expérience IT | PMP · Agile · Azure · AWS · ITIL · COBIT  
En transition vers l'AI Governance et l'AI Risk Management
🔗 LinkedIn | GitHub
---
Licence
MIT — libre d'utilisation, de modification et de distribution.
