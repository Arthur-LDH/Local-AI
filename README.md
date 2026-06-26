# Local LLM Gateway

Stack self-hosted : **Ollama** (inférence locale) + **LiteLLM Proxy** (gateway OpenAI-compatible).

```
Application ──► http://localhost:4000/v1/  (LiteLLM)
                        │
              ┌─────────┴──────────┐
              ▼                    ▼
        ollama:11434          API cloud
        (modèles locaux)   (claude, gemini…)
```

Les modèles et leurs clés API sont gérés via l'UI LiteLLM et stockés en base PostgreSQL — aucune clé en clair dans les fichiers du projet.

---

## Prérequis

| Logiciel | Version min | Lien |
|---|---|---|
| Docker Engine | 24.x | https://docs.docker.com/engine/install/ |
| Docker Compose plugin | 2.20 | intégré depuis Docker 24 |
| PostgreSQL | externe | base et utilisateur créés à l'avance |
| nvidia-container-toolkit | dernière | https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/install-guide.html |

> **Sans GPU NVIDIA** : ajoutez le bloc suivant au service `ollama` dans `docker-compose.yml` pour utiliser le CPU uniquement — aucune autre modification nécessaire. Les performances seront réduites.

---

## Démarrage rapide

```bash
# 1. Remplir les deux variables
cp .env.example .env
$EDITOR .env            # DATABASE_URL et LITELLM_MASTER_KEY

# 2. Lancer la stack
docker compose up -d

# 3. Télécharger un modèle Ollama (première fois, ~8 Go pour gemma3:12b)
docker exec -it ollama ollama pull gemma3:12b

# 4. Ouvrir l'UI d'administration
#    http://localhost:4000/ui  →  se connecter avec LITELLM_MASTER_KEY

# 5. Vérifier que tout tourne
docker compose ps
```

---

## Ajouter un modèle

### Modèle local Ollama

```bash
# 1. Télécharger le modèle
docker exec -it ollama ollama pull <nom-du-modele>
# Exemples :
#   docker exec -it ollama ollama pull mistral:7b
#   docker exec -it ollama ollama pull llama3.2:3b

# 2. Vérifier qu'il est disponible
docker exec ollama ollama list
```

Puis dans l'UI LiteLLM (`/ui`) → **Models** → **Add Model** :
- Model name (alias) : `mistral`
- LiteLLM model : `ollama/mistral:7b`
- API Base : `http://ollama:11434`

Ou via l'API :

```bash
curl -X POST http://localhost:4000/model/new \
  -H "Authorization: Bearer $LITELLM_MASTER_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model_name": "mistral",
    "litellm_params": {
      "model": "ollama/mistral:7b",
      "api_base": "http://ollama:11434"
    }
  }'
```

### Modèle cloud (Anthropic, Gemini…)

Via l'UI ou l'API — la clé est stockée chiffrée en base, jamais dans les fichiers :

```bash
curl -X POST http://localhost:4000/model/new \
  -H "Authorization: Bearer $LITELLM_MASTER_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model_name": "claude",
    "litellm_params": {
      "model": "anthropic/claude-sonnet-4-5",
      "api_key": "sk-ant-..."
    }
  }'
```

---

## Tests rapides

Remplacez `sk-local-changeme` par la valeur de `LITELLM_MASTER_KEY`.

```bash
# Modèle local
curl http://localhost:4000/v1/chat/completions \
  -H "Authorization: Bearer sk-local-changeme" \
  -H "Content-Type: application/json" \
  -d '{"model": "gemma4", "messages": [{"role": "user", "content": "Dis bonjour."}]}'

# Modèle cloud (après ajout via UI/API)
curl http://localhost:4000/v1/chat/completions \
  -H "Authorization: Bearer sk-local-changeme" \
  -H "Content-Type: application/json" \
  -d '{"model": "claude", "messages": [{"role": "user", "content": "Dis bonjour."}]}'
```

---

## Opérations courantes

```bash
# Logs en temps réel
docker compose logs -f litellm
docker compose logs -f ollama

# Lister les modèles Ollama présents
docker exec ollama ollama list

# Arrêter la stack
docker compose down

# Arrêter et supprimer les volumes (efface les modèles téléchargés !)
docker compose down -v
```
