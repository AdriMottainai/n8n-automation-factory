# n8n Automation Factory

**Une « factory » d'automatisations IA auto-hébergées, local-first** : n8n + Ollama + Qdrant + PostgreSQL, orchestrés en Docker Compose, avec un jeu de conventions d'ingénierie pour construire des workflows **robustes, debuggables, maintenables et exportables**.

Construit au-dessus du [self-hosted AI starter kit de n8n](https://github.com/n8n-io/self-hosted-ai-starter-kit) (Apache 2.0), enrichi d'une structure « factory », de conventions de production, et d'un cas d'usage de bout en bout.

> Stack 100 % auto-hébergée, aucune API LLM payante. L'intérêt est dans les **choix d'architecture** (routage LLM, configuration externalisée, isolation des données) et un jeu de **conventions** réutilisables d'un projet à l'autre.

---

## Stack

```
Host macOS (Apple Silicon M1, 8 GB RAM)
├── Ollama natif ─── localhost:11434 (GPU Metal)
│   ├── llama3.2:3b          génération / classification / extraction
│   └── bge-m3 / nomic-embed embeddings (RAG)
└── Docker Compose
    ├── n8n ──────── localhost:5678   orchestrateur (UI + API)
    ├── postgres ─── interne           backend n8n (workflows + exécutions)
    └── qdrant ───── localhost:6333    vector store
```

| Cible | URL depuis un node n8n (Docker) | URL depuis le host |
|---|---|---|
| Ollama | `http://host.docker.internal:11434` | `http://localhost:11434` |
| Qdrant | `http://qdrant:6333` | `http://localhost:6333` |
| Fichiers partagés | `/data/shared` | `./shared/` |

## Politique de routage LLM : LOCAL-FIRST, CLOUD-FALLBACK

**Aucune API payante.** Tout passe par Ollama.
- **Niveau 1 — local (`llama3.2:3b`)** : classification, extraction JSON, résumé court, RAG Q&A, scoring.
- **Niveau 2 — Ollama Cloud (free tier)** : raisonnement multi-étapes, génération longue, gros documents, code.

Le routage se fait par un node Switch selon la complexité — pas par le credential.

## Structure du repo

```
.
├── docker-compose.yml      infra (n8n, postgres, qdrant) + config via env vars
├── CLAUDE.md               porte d'entrée + conventions de travail
├── n8n/
│   └── conventions.md      conventions factory : archi, robustesse, observabilité, exportabilité
├── shared/                 bind mount → /data/shared (gitignored : données & I/O projets)
└── projets/                métadonnées des projets actifs (1-pagers + analyses)
    └── veille-emploi/       cas d'usage de démonstration ↓
```

Les **workflows vivent en base** (Postgres n8n), pas dans le repo — c'est l'instance n8n qui en est la source de vérité. Les `projets/<nom>/` portent la documentation humaine ; `shared/<nom>/` porte les données (gitignored).

## Cas d'usage de démonstration — « Veille Emploi »

Pipeline d'automatisation quotidienne de veille (illustré avec une persona générique *Sales / RevOps*) :

- **W1 — feeds** : ingestion multi-source (Remotive, Jobicy, WeWorkRemotely, France Travail) → normalisation + filtrage configurable → déduplication vectorielle (Qdrant) → matching CV par similarité cosine → **scoring LLM local** → digest Markdown + email.
- **W2 — alertes email** : lecture d'alertes Gmail → **extraction LLM (Ollama Cloud)** → filtrage → scoring local → digest + email.

Détail technique, audit d'anti-patterns et stratégie de refacto : [`projets/veille-emploi/`](projets/veille-emploi/).

## Choix architecturaux & techniques

- **Local-first, cloud-fallback** — un node Switch route selon la complexité : modèle local (`llama3.2:3b`) par défaut, Ollama Cloud (free tier) seulement pour le raisonnement long. Aucune API payante.
- **Workflows en base, pas dans le repo** — l'instance n8n (Postgres) est la source de vérité ; le repo porte l'infra, les conventions et la doc. Les données restent sous `shared/<projet>/`, hors versionnement.
- **Configuration externalisée** — paths, URLs, modèles et seuils passent par des variables d'environnement, pas en dur dans les workflows.
- **RAG local** — embeddings `bge-m3` + Qdrant pour dédoublonner les offres et les matcher au CV par similarité, sans service externe.

## Quick start

```bash
git clone <this-repo>
cd n8n-automation-factory
cp .env.example .env        # renseigne secrets + VEILLE_TO_EMAIL
docker compose up -d        # CPU par défaut ; profils gpu-nvidia / gpu-amd disponibles
```

n8n : <http://localhost:5678/> · Qdrant : <http://localhost:6333/dashboard>

Mac / Apple Silicon : pour de meilleures perfs, faire tourner Ollama **nativement** (pas dans Docker) et pointer `OLLAMA_HOST=host.docker.internal:11434`. Voir [`CLAUDE.md`](CLAUDE.md).

## Crédits & licence

Basé sur le [self-hosted AI starter kit](https://github.com/n8n-io/self-hosted-ai-starter-kit) de n8n. Sous licence **Apache 2.0** — voir [LICENSE](LICENSE).
