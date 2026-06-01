# Veille Emploi — projet en cours

Automatisation quotidienne de veille emploi : ingestion multi-source (feeds API + alertes email), filtrage par profil, scoring LLM, matching CV vectoriel, digest markdown + email.

## Workflows n8n

| ID | Nom | État | Tier |
|---|---|---|---|
| `q2crCIIaJhSsXfRx` | Veille Emploi (daily) | **active=true** (prod) | 1 — feeds API/RSS |
| `b8erSgFcfJ32W7vx` | Veille Emploi Emails (daily) | **active=true** (prod depuis 2026-06-01) | 2 — alertes plateformes |

UI : http://localhost:5678/workflow/q2crCIIaJhSsXfRx · http://localhost:5678/workflow/b8erSgFcfJ32W7vx

## Pipeline résumé

**W1 — feeds** : Remotive + Jobicy + WeWorkRemotely + France Travail → normalize + filter → dedup vectoriel (Qdrant `job_listings`) → match CV (Qdrant `cv_profile`) → score LLM local → digest markdown + email.

**W2 — emails** : Gmail label `job-alerts` → extract LLM cloud (qwen3-coder:480b) → filter → score LLM local → digest markdown + email.

## Fichiers & dossiers projet

| Chemin | Rôle |
|---|---|
| `projets/veille-emploi/README.md` | ce fichier — 1-pager projet |
| `projets/veille-emploi/analysis.md` | analyse factory & extraction (audit n8n + plan refacto) |
| `shared/veille-emploi/cv/` | CV PDF (input) |
| `shared/veille-emploi/jobs/YYYY-MM-DD.md` | digest journalier W1 (output) |
| `shared/veille-emploi/emails/YYYY-MM-DD.md` | digest journalier W2 (output) |

_Namespacing par projet (`shared/veille-emploi/`) + refacto P0 env vars : **faits le 2026-06-01** (bloc B). Tous les paths/URLs/modèles des workflows passent par `$env.*` (voir `docker-compose.yml` section `x-n8n`)._

## État au 2026-06-01

- **W1** : prod stable. Refacto env vars + namespacing `shared/veille-emploi/` appliqués (bloc B).
- **W2** : **activée** (active=true). Refacto env vars appliqué.
- **Bloc B fait** : `docker-compose.yml` porte les env vars `VEILLE_*` / `OLLAMA_*_URL` / `QDRANT_URL` + `N8N_BLOCK_ENV_ACCESS_IN_NODE=false` (requis pour `$env` en Code node) + perf M1 (`N8N_CONCURRENCY_PRODUCTION_LIMIT=3`, `NODE_OPTIONS`, `mem_limit=2.5g`). Mécanisme `$env` validé via sonde.
- **À surveiller W2** : qualité d'extraction (location/url/desc souvent `"Not specified"` sur les emails LinkedIn). Si récurrent → patch prompt + préserver `<a href>` ou switcher modèle vers `qwen3.5:397b` / `gpt-oss:120b`.

## Reste à faire

Voir `projets/veille-emploi/analysis.md` § 1 (anti-patterns) et §3 (priorités P0/P1/P2), et la project memory `[[project_veille_emploi_workflow]]` pour l'état détaillé.

## Credentials n8n

Listées dans la project memory `[[project_veille_emploi_workflow]]` (6 creds : France Travail, Gmail, Ollama local, Qdrant, Ollama Cloud, Ollama Cloud HTTP).

## Graduation

Quand W2 est stable depuis ~1 mois, suivre la checklist `analysis.md § 4.3`. Destination : `~/projets/n8n/veille-emploi/`.
