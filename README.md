# n8n Automation Factory

**Une « factory » d'automatisations IA auto-hébergées** : n8n + Ollama (embeddings) + Qdrant + PostgreSQL orchestrés en Docker Compose, inférence LLM sur API gratuite (Groq free tier), avec un jeu de conventions d'ingénierie pour construire des workflows **robustes, debuggables, maintenables et exportables**.

Construit au-dessus du [self-hosted AI starter kit de n8n](https://github.com/n8n-io/self-hosted-ai-starter-kit) (Apache 2.0), enrichi d'une structure « factory », de conventions de production, et d'un cas d'usage de bout en bout.

> Stack auto-hébergée (n8n + Postgres + Qdrant + embeddings Ollama), **inférence LLM sur API gratuite** (Groq free tier, OpenAI-compatible). Aucune API payante. L'intérêt est dans les **choix d'architecture** (routage LLM adaptatif par tâche, configuration externalisée, isolation des données) et un jeu de **conventions** réutilisables d'un projet à l'autre.

---

## Stack

```
Host macOS (Apple Silicon M1, 8 GB RAM)
├── Ollama natif ─── localhost:11434 (GPU Metal)
│   └── bge-m3               embeddings (RAG : dédup Qdrant + matching CV)
└── Docker Compose
    ├── n8n ──────── localhost:5678   orchestrateur (UI + API)
    ├── postgres ─── interne           backend n8n (workflows + exécutions)
    └── qdrant ───── localhost:6333    vector store

Inférence LLM (scoring + extraction) ─ Groq API (free tier, OpenAI-compatible)
    ├── llama-3.3-70b-versatile        scoring des offres
    └── llama-4-scout (ctx 131K)       extraction des offres (emails)
```

| Cible | URL depuis un node n8n (Docker) | URL depuis le host |
|---|---|---|
| Ollama | `http://host.docker.internal:11434` | `http://localhost:11434` |
| Qdrant | `http://qdrant:6333` | `http://localhost:6333` |
| Fichiers partagés | `/data/shared` | `./shared/` |

## Routage LLM — adaptatif, par tâche

Le modèle et l'hébergement sont choisis **par tâche** (coût / qualité / confidentialité / latence), **pas** par un node Switch dynamique. **Aucune API payante** (Groq free tier + Ollama local).
- **Embeddings & RAG** (dédup Qdrant, matching CV) : **Ollama local** (`bge-m3`, 1024-dim) — les données ne quittent pas la machine.
- **Scoring & extraction** (scorer les offres, extraire les offres des emails) : **Groq API** (free tier, OpenAI-compatible) — `llama-3.3-70b-versatile` (scoring), `llama-4-scout` (extraction, contexte 131K).

Le choix est **fixé par étape**, paramétré par variables d'environnement (`VEILLE_*_MODEL_*`, `GROQ_*`). Historique : le pipeline tournait 100 % local (`llama3.2:3b`) ; migré vers Groq pour la qualité de scoring et la fiabilité d'extraction, **sans coût**.

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

- **W1 — feeds** : ingestion multi-source (7 sources : Remotive, Jobicy, WeWorkRemotely, France Travail, Himalayas, The Muse, RemoteOK, + recherche ciblée par rôle) → normalisation + filtrage configurable → déduplication vectorielle (Qdrant) → matching CV par similarité cosine → **scoring LLM (Groq `llama-3.3-70b`)** → digest Markdown + email.
- **W2 — alertes email** : lecture d'alertes Gmail → **extraction LLM (Groq `llama-4-scout`)** → filtrage → scoring (Groq) → digest + email.

Détail technique, audit d'anti-patterns et stratégie de refacto : [`projets/veille-emploi/`](projets/veille-emploi/).

## Choix architecturaux & techniques

- **Routage LLM adaptatif par tâche** — embeddings en local (`bge-m3`), scoring + extraction sur **Groq** (free tier, `llama-3.3-70b` / `llama-4-scout`). Choix **fixé par étape** (variables d'env), pas un node Switch. Aucune API payante.
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
