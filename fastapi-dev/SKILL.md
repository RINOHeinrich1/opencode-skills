---
name: fastapi-dev
description: >-
  Use when working on a FastAPI / Pydantic / Python backend: routes, Pydantic
  models, dependency injection (Depends), async, lifespan, background tasks,
  test client, or 422 validation. Trigger on words like "fastapi", "pydantic",
  "Depends", "APIRouter", "uvicorn", "async def", "lifespan", "TestClient",
  "response_model", "BaseModel". Compensates a known model weakness: never mix
  sync/async wrongly nor trust request bodies without a Pydantic model.
---

# FastAPI — travailler juste

## Modèles & validation (Pydantic v2)
- Tout corps de requête passe par un modèle `BaseModel` (jamais un `dict` brut).
- Config v2 : `model_config = ConfigDict(from_attributes=True)` (remplace `orm_mode`).
- Sérialiser : `model.model_dump()` (v2), pas `.dict()` (v1).
- Séparer les modèles : entrée (`Create`), sortie (`Read`) — `response_model=` sur la route
  pour ne pas fuiter des champs internes (ex. mot de passe, hash).

## Routes & dépendances
- Regrouper par `APIRouter(prefix="/x", tags=["x"])` et inclure via `app.include_router`.
- Injection : `def route(dep = Depends(get_dep))` — la dépendance peut elle-même en
  injecter d'autres (DB, user courant, pagination).
- Statut explicite : `status_code=201` pour une création, `raise HTTPException(404, ...)`.
- Erreurs de validation → **422** automatique ; ne pas réinventer.

## Async — le piège n°1
- `async def` qui appelle une bibliothèque **bloquante** (ex. client DB sync) bloque la
  boucle : soit tout est async (driver async), soit la route est `def` (FastAPI l'exécute
  dans un threadpool). **Ne pas mélanger arbitrairement.**
- Ne pas faire d'appel réseau bloquant dans un `async def`.

## Cycle de vie & background
- Startup/shutdown : utiliser le `lifespan` context manager (les `@app.on_event` sont
  dépréciés).
- Tâche détachée : `BackgroundTasks` (inclus) ; pour du lourd → vrai worker (Celery/RQ).

## Structure recommandée
- `main.py` (app + include_router), `routers/`, `schemas/` (Pydantic), `services/`,
  `deps.py` (dépendances partagées), `core/` (config, sécurité).
- Config via `pydantic-settings` (`Settings(BaseSettings)`), pas de constantes en dur.

## Vérification obligatoire
1. Lire `pyproject.toml`/`requirements.txt` **avant** de citer une version ou un outil.
2. Lint/format : `ruff check .` / `ruff format .` (ou `black`/`flake8` selon le repo).
3. Types : `mypy` si configuré.
4. Tests : `python -m pytest` (utiliser `TestClient` ou `httpx.AsyncClient`) — ne jamais
   affirmer qu'une route marche sans l'exécuter.
