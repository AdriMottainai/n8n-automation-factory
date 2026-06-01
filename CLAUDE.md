# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

Automation factory built on the self-hosted AI starter kit. Every new workflow/agent starts here, gets built and tested locally via n8n-mcp, then can be extracted to its own project when mature.

## Factory structure

Ce repo est la **factory** : infra + conventions + leçons capitalisées. Les workflows vivent en Postgres (n8n DB), pas dans le repo. Chaque projet actif a un couloir dédié, déménagé hors du repo une fois mature.

```
self-hosted-ai-starter-kit/             ← FACTORY
├── docker-compose.yml                  ← infra (n8n, postgres, qdrant, ollama)
├── CLAUDE.md / README.md               ← conventions + porte d'entrée
├── .env / .env.example                 ← config non-sensible (secrets en ~/.zshrc)
├── .mcp.json / .claude/settings.json   ← MCP + permissions Claude
├── n8n/                                ← config & assets n8n partagés (sub-workflows à venir)
├── shared/                             ← bind mount → /data/shared (container n8n)
│   └── <nom-projet>/                   ← inputs + outputs projet (gitignored)
└── projets/                            ← métadonnées projets actifs
    └── <nom-projet>/
        ├── README.md                   ← 1-pager projet
        ├── analysis.md                 ← analyses & audits projet
        └── workflows/                  ← snapshots JSON optionnels
```

**Pourquoi cette séparation** : `shared/<projet>/` (que n8n voit comme `/data/shared/<projet>/`) porte les données ; `projets/<projet>/` porte les documents humains. À la graduation, on déplace les deux couloirs ensemble vers `~/projets/n8n/<projet>/`.

### Démarrer un nouveau projet

1. `mkdir projets/<nom>/ shared/<nom>/` (le second uniquement si le projet manipule des fichiers)
2. Créer `projets/<nom>/README.md` (mission, IDs n8n, état)
3. Créer une project memory `project_<nom>_workflow.md` dans `~/.claude/projects/.../memory/` + ajout dans `MEMORY.md`
4. Créer les workflows dans n8n UI ou via n8n-mcp ; tous les paths utilisateur dans les nodes pointent vers `/data/shared/<nom>/...`

### Graduation (workflow mature, libérer la factory)

Quand un projet est stable depuis ~1 mois :

```bash
mkdir -p ~/projets/n8n/<nom>/{workflows,qdrant,memory}

# Snapshots optionnels (workflow JSON + Qdrant collections)
for id in <wf-ids>; do
  curl -s -H "X-N8N-API-KEY: $N8N_API_KEY" \
    "http://localhost:5678/api/v1/workflows/$id" \
    > ~/projets/n8n/<nom>/workflows/$id.json
done

# Déménagement des couloirs projet
mv shared/<nom>     ~/projets/n8n/<nom>/shared
mv projets/<nom>    ~/projets/n8n/<nom>/projets

# Déménagement memory projet (les feedbacks transférables RESTENT en factory)
mv ~/.claude/projects/-Users-adrien-projets-self-hosted-ai-starter-kit/memory/project_<nom>_workflow.md \
   ~/projets/n8n/<nom>/memory/
# + retirer la ligne correspondante de MEMORY.md
```

Les workflows continuent à tourner dans la même instance n8n (Postgres garde tout). Pour désactiver : `POST /api/v1/workflows/{id}/deactivate`.

**Avant de graduer** : vérifier que toutes les leçons transférables (gotchas n8n, patterns réutilisables) sont déjà promues en `feedback_*.md` factory-level — sinon elles partent avec le projet et sont perdues pour les projets suivants.

### Conventions n8n production

`n8n/conventions.md` contient les conventions factory pour construire des workflows robustes, debuggables, maintenables et exportables — **à lire avant de démarrer un nouveau workflow ou refactorer**. Couvre architecture/maintenabilité (naming, Code node limits, sub-workflows), robustesse (errorWorkflow, onError, timeouts, idempotency, OAuth refresh), observabilité (logs JSON, EXECUTIONS_DATA_SAVE_*, pinData, test_workflow MCP, staticData, healthz), et exportabilité ($env discipline, magic strings, CLI export/import, N8N_ENCRYPTION_KEY). Inclut checklist nouveau workflow et anti-patterns à fuir.

## Stack

```
Host macOS (Mac M1, 8 GB RAM)
├── Ollama natif ─── localhost:11434 (GPU Metal)
│   └── llama3.2:3b (2 GB)
└── Docker Desktop
    ├── n8n ──────── localhost:5678 (UI + API)
    ├── postgres ─── interne (DB backend n8n)
    └── qdrant ───── localhost:6333 (vector store)
```

### Connexions depuis les nodes n8n (Docker)

| Cible | URL depuis n8n | URL depuis le host |
|---|---|---|
| Ollama | `http://host.docker.internal:11434` | `http://localhost:11434` |
| Qdrant | `http://qdrant:6333` | `http://localhost:6333` |
| Postgres | `postgres:5432` (env vars) | `localhost:5432` |
| Fichiers partages | `/data/shared` | `./shared/` |

### Docker lifecycle

| Commande | Effet |
|---|---|
| `docker compose stop` / `start` | Pause/reprise, conserve containers + volumes |
| `docker compose down` | Detruit containers, **conserve** volumes (donnees OK) |
| `docker compose down -v` | **Destructif** — detruit volumes, reset complet |
| `docker compose up -d` | Recree containers, **necessite** env vars dans le shell |

## n8n Credentials (premiere utilisation)

- **Ollama local** : type "Ollama", base URL = `http://host.docker.internal:11434/`
- **Ollama Cloud** : type "Ollama", base URL = `https://ollama.com`, API key = `$OLLAMA_API_KEY`
- **Qdrant** : type "Qdrant", URL = `http://qdrant:6333` (pas d'API key)

## Modeles

### Ollama local (gratuit, illimite)

| Modele | Taille | Usage |
|---|---|---|
| `llama3.2:3b` | 2 GB | Generation, classification, extraction, resume court |
| `nomic-embed-text` | 274 MB | Embeddings pour RAG/Qdrant (**a pull** : `ollama pull nomic-embed-text`) |

Contrainte M1 8 GB : **un seul modele charge a la fois**. Decharge auto apres 5 min.

### Ollama Cloud (free tier)

Meme API, endpoint `https://ollama.com`. Modeles avec suffixe `:cloud` (liste : ollama.com/search?c=cloud).

| Modele | Parametres | Force |
|---|---|---|
| `deepseek-v3.1:cloud` | 671B | Raisonnement complexe, long contexte |
| `qwen3-coder:cloud` | 480B | Generation et analyse de code |
| `qwen3.5:cloud` | variable | Generaliste puissant |

Free tier : 1 modele concurrent, usage "light" (resets 5h/7j).

## Politique de routage : LOCAL-FIRST, CLOUD-FALLBACK

**Aucune API payante** (pas de Claude API, pas d'OpenAI). Tout passe par Ollama.

**Niveau 1 — local (llama3.2:3b) par defaut** : classification, resume court, extraction JSON, RAG Q&A, templates, triage.

**Niveau 2 — Cloud quand le local ne suffit pas** : raisonnement multi-etapes, generation longue (> 1000 mots), code (`qwen3-coder:cloud`), gros documents (> 4K tokens).

**Dans n8n** : deux credentials Ollama (local + Cloud) + Switch node qui route selon la complexite. Le routage se fait par le Switch, pas par le credential.

```
[Trigger] → [Switch: complexite?]
              ├── simple → Ollama Chat Model (local)
              └── complexe → Ollama Chat Model (Cloud, :cloud)
            → [Output]
```

## MCP

### n8n-mcp (pilotage n8n)

| Outil | Usage |
|---|---|
| `n8n_health_check` | Sante de l'instance |
| `n8n_list_workflows` / `n8n_get_workflow` | Lister / inspecter |
| `n8n_create_workflow` / `n8n_generate_workflow` | Creer / generer depuis description |
| `n8n_update_full_workflow` / `n8n_update_partial_workflow` | Modifier |
| `n8n_validate_workflow` / `n8n_autofix_workflow` | Valider / corriger |
| `n8n_test_workflow` / `n8n_executions` | Tester / voir executions |
| `n8n_manage_credentials` | Gerer credentials |
| `n8n_deploy_template` | Deployer un template |
| `search_nodes` / `search_templates` | Chercher nodes ou templates |
| `get_node` / `validate_node` | Info / validation node |

### Context7 (documentation a jour)

`resolve-library-id` puis `query-docs`. Utiliser des qu'on touche a une lib/SDK/framework.

### Skills n8n (`/n8n-*`)

`/n8n-workflow-patterns`, `/n8n-node-configuration`, `/n8n-expression-syntax`, `/n8n-code-python`, `/n8n-code-javascript`, `/n8n-mcp-tools-expert`, `/n8n-validation-expert`

### Configuration MCP

n8n-mcp est en scope **project** (`.mcp.json`, gitignoré). `${N8N_API_KEY}` est une reference resolue au runtime depuis l'env shell. Pour que ca marche : la variable doit etre exportee dans `~/.zshrc` AVANT de lancer `claude`.

Context7 est en scope **user** (`~/.claude.json`), disponible dans tous les projets.

Si MCP ne se charge pas : verifier le trust dialog au premier lancement, que `$N8N_API_KEY` est dans l'env, que n8n tourne (`docker compose ps`).

## Construire une automatisation

1. **Decrire** en langage naturel (trigger, traitement, sortie) et utiliser le skill deep-planning pour structurer la requête, et le skill apex pour structurer les étapes d'implémentation de la meilleure automatisation.
2. **Chercher** un template : `search_templates` → `n8n_deploy_template`
3. **Generer** si pas de template : `n8n_generate_workflow`
4. **Valider** : `n8n_validate_workflow`
5. **Credentials** : `n8n_manage_credentials` (Ollama local, Ollama Cloud, Qdrant, etc.)
6. **Tester** : `n8n_test_workflow` ou UI n8n
7. **Iterer** : `n8n_autofix_workflow` / `n8n_update_partial_workflow`
8. **Activer** : mode "Active" pour les triggers automatiques

### Patterns courants

**RAG local** : Local File Trigger (`/data/shared`) → chunk → embed (nomic-embed-text) → Qdrant. Chat Trigger → query Qdrant → llama3.2:3b.

**Classification/triage** : Trigger → llama3.2:3b classifie en JSON → Switch → actions par categorie → Postgres.

**Veille web** : Schedule Trigger → HTTP Request (RSS) → llama3.2:3b resume → `shared/veille/YYYY-MM-DD.md`.

**Pipeline docs** : fichier dans `shared/` → extraction → llama3.2:3b → `shared/output/`.

## Commandes

```bash
# Stack
docker compose ps                     # Statut
docker compose logs -f n8n            # Logs n8n
docker compose restart n8n            # Redemarrer n8n
docker compose up -d                  # Demarrer (env vars requises)
docker compose stop                   # Pause

# Ollama
ollama list                           # Modeles installes
ollama pull nomic-embed-text          # Modele d'embedding
curl http://localhost:11434/api/tags  # Check API

# Sante
curl http://localhost:5678/healthz    # n8n
curl http://localhost:6333/           # Qdrant
```

## Securite

**Secrets** dans `~/.zshrc` (jamais dans le repo) : `POSTGRES_USER`, `POSTGRES_PASSWORD`, `N8N_ENCRYPTION_KEY`, `N8N_USER_MANAGEMENT_JWT_SECRET`, `N8N_API_KEY`, `OLLAMA_API_KEY`.

**`.env`** : uniquement config non-sensible (`POSTGRES_DB`, `OLLAMA_HOST`, `N8N_DEFAULT_BINARY_DATA_MODE`).

**Permissions Claude Code** (`.claude/settings.json`) : whitelist stricte pour Docker/git/curl localhost. Deny sur `.env`, destructif, exfil API. Ask sur commit/push/install.

**`N8N_BLOCK_ENV_ACCESS_IN_NODE=true`** (defaut n8n v2) : Code nodes ne lisent pas les env vars.

**Backup critique** : `N8N_ENCRYPTION_KEY` — si perdue, tous les credentials n8n deviennent illisibles. Sauvegarder hors machine (password manager).

### Hardening a faire avant d'exposer des webhooks

- Verifier la protection SSRF (`host.docker.internal:11434` peut etre bloque)
- Signature HMAC-SHA256 sur webhooks entrants (node Crypto)
- Reverse proxy (Traefik/nginx/Caddy) pour rate limiting si exposition publique

## Extraire une automatisation

1. `n8n_get_workflow` → JSON dans `shared/exports/<nom>.json`
2. Documenter les dependances (credentials, modeles, services)
3. README minimal (quoi, comment configurer, comment lancer)
4. Si standalone : extraire un `docker-compose.yml` minimal
5. Partager : galerie templates n8n ou repo independant
