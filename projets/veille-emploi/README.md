# Veille Emploi — projet en cours

Automatisation quotidienne de veille emploi : ingestion multi-source (feeds API + alertes email), filtrage par profil, scoring LLM, matching CV vectoriel, digest markdown + email.

## Workflows n8n

| ID | Nom | État | Nodes | Tier |
|---|---|---|---|---|
| `q2crCIIaJhSsXfRx` | Veille Emploi (daily) | **active=true** (prod) | 24 | 1 — feeds API/RSS |
| `b8erSgFcfJ32W7vx` | Veille Emploi Emails (daily) | **active=true** (prod) | 14 | 2 — alertes plateformes |
| `y3fd7SBf6dhU261q` | _global_error_handler | **active=true** | 4 | util — filet d'erreur (W1+W2) |

Snapshots JSON figés : `projets/veille-emploi/workflows/<id>.json`.

UI : http://localhost:5678/workflow/q2crCIIaJhSsXfRx · http://localhost:5678/workflow/b8erSgFcfJ32W7vx

## Pipeline résumé

**W1 — feeds** : 7 sources (Remotive, Jobicy, WWR, France Travail, Himalayas, The Muse, RemoteOK) + recherche ciblée par rôle → normalize + filter (titre+desc) → dedup vectoriel (Qdrant `job_listings`, embed `bge-m3` local) → match CV (Qdrant `cv_profile`) → **score LLM Groq `llama-3.3-70b`** → digest markdown + email.

**W2 — emails** : Gmail label `job-alerts` (returnAll) → **extract LLM Groq `llama-4-scout`** (ctx 131K) → filter → **score LLM Groq** → digest markdown + email.

> État détaillé (audit + P0/P1/P2 Groq, 2026-07-01) : project memory `[[project_veille_emploi_workflow]]`, bloc « ÉTAT COURANT ».

## Fichiers & dossiers projet

| Chemin | Rôle |
|---|---|
| `projets/veille-emploi/README.md` | ce fichier — 1-pager projet |
| `projets/veille-emploi/analysis.md` | analyse factory & extraction (audit n8n + plan refacto) |
| `shared/veille-emploi/cv/` | CV PDF (input) |
| `shared/veille-emploi/jobs/YYYY-MM-DD.md` | digest journalier W1 (output) |
| `shared/veille-emploi/emails/YYYY-MM-DD.md` | digest journalier W2 (output) |

_Namespacing par projet (`shared/veille-emploi/`) + refacto P0 env vars : **faits le 2026-06-01** (bloc B). Tous les paths/URLs/modèles des workflows passent par `$env.*` (voir `docker-compose.yml` section `x-n8n`)._

## État au 2026-06-07 (hardening prod)

Session de diagnostic + durcissement. Les 2 workflows étaient fonctionnels mais échouaient **en silence** (2 plantages OAuth Gmail le 04-06 non alertés). Corrigé :
- **Filet d'erreur** : `_global_error_handler` (Error Trigger → fichier `_errors/<ts>.json` + email) attaché à W1+W2.
- **OAuth** : app Google publiée en **Production** (fin de l'expiry 7j du mode Testing) + token régénéré.
- **Idempotence W1** : l'upsert Qdrant (marquage « vu ») déplacé **après** l'envoi de l'email (node `Index New Jobs`) → plus de perte d'offre sur échec d'envoi. Validé live (Qdrant 40→41 post-envoi).
- **Coût W2** : node `Dedup New Emails` (message-id) → fin de la ré-extraction LLM cloud de 7 jours d'emails à chaque run + des digests en double.
- **Robustesse** : `retryOnFail` (3×) sur fetch+LLM, `timeout` HTTP sources, `executionTimeout` (W1 1200s / W2 900s).
- **Observabilité sources W1** : 7 sources en `continueErrorOutput` → `Log Failed Sources` (visibilité d'une source morte).
- **Scoring W2** : température 0 (déterministe).
- **Outillage** : n8n-mcp bloquait localhost (SSRF strict) → `WEBHOOK_SECURITY_MODE=moderate` dans `.mcp.json` (effet au prochain lancement de `claude`).

Détail complet + items à reprendre : project memory `[[project_veille_emploi_workflow]]` (bloc « ÉTAT COURANT »).

⚠️ **Schedule décoratif** : les crons 9h ne se déclenchent pas seuls (Mac en veille → VM Docker suspendue, pas de rattrapage). Run manuel, ou host always-on pour du 24/7.

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
