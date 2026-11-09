# Architecture — Document QA Assistant

> Statut : **validée** (v1, avant implémentation).

## Objectif

Déposer des PDF (normes, politiques internes, documentation technique), puis poser des questions en langage naturel. Les réponses sont générées par RAG, diffusées en direct (streaming) et **citent les pages sources exactes**.

Stack : Angular · ASP.NET Core (.NET 10) · PostgreSQL + pgvector · API Claude.

## 1. Vue d'ensemble

```
 Angular ──HTTP──▶ ASP.NET Core API ──────────▶ PostgreSQL + pgvector
    ▲                │  │                          ▲
    │ SSE (stream)   │  └─▶ Worker d'ingestion ────┘  (BackgroundService)
    └────────────────┘         │
                               ├─▶ Extraction PDF (par page)
                               └─▶ Fournisseur d'embeddings
            API ──▶ API Claude (streaming + citations)
```

Principes :

- **Un seul processus backend** : l'API héberge le worker d'ingestion (`BackgroundService`).
- **PostgreSQL sert aussi de file d'attente** : ni Redis ni RabbitMQ.
- **Démarrage en une commande** : `docker compose up`.

## 2. Structure du dépôt

```
backend/
  src/DocQa.Api/                  endpoints, SSE, configuration, worker
  src/DocQa.Core/                 domaine, interfaces (IEmbeddingProvider, IAnswerGenerator, IFileStorage…)
  src/DocQa.Infrastructure/       EF Core + pgvector, PdfPig, client Claude, embeddings
  tests/DocQa.UnitTests/
  tests/DocQa.IntegrationTests/   Testcontainers (image pgvector/pgvector)
frontend/                         Angular (standalone components, signals)
docker-compose.yml                postgres+pgvector, api, front
.github/workflows/ci.yml
```

Trois projets suffisent : la séparation reste nette, sans couches CQRS ou Clean Architecture complètes, qui seraient de la sur-ingénierie à cette échelle.

## 3. Modèle de données

### `documents`

| Colonne       | Type          | Notes                                        |
|---------------|---------------|----------------------------------------------|
| `id`          | uuid          | PK                                           |
| `file_name`   | text          |                                              |
| `sha256`      | text          | unique, pour détecter les doublons           |
| `status`      | text          | `Pending` · `Processing` · `Ready` · `Failed` |
| `page_count`  | int           | renseigné après l'extraction                 |
| `error`       | text NULL     | message lisible si `Failed`                  |
| `locked_at`   | timestamptz   | réservation par le worker                    |
| `created_at`  | timestamptz   |                                              |

### `chunks`

| Colonne        | Type          | Notes                                           |
|----------------|---------------|-------------------------------------------------|
| `id`           | uuid          | PK                                              |
| `document_id`  | uuid          | FK → `documents`, `ON DELETE CASCADE`           |
| `page_number`  | int           | 1-indexé ; un chunk n'appartient qu'à une page  |
| `chunk_index`  | int           | ordre dans la page                              |
| `content`      | text          |                                                 |
| `embedding`    | vector(N)     | N dépend du fournisseur d'embeddings            |
| `search_vector`| tsvector      | colonne générée (config `french` / `simple`)    |

Index :

- **HNSW** (`vector_cosine_ops`) sur `embedding` ;
- **GIN** sur `search_vector`.

Les PDF sont stockés sur un volume, derrière `IFileStorage` (on pourra passer à S3 ou Azure Blob plus tard).

## 4. Pipeline d'ingestion (arrière-plan)

1. `POST /api/documents` : l'API enregistre le fichier, crée une ligne `Pending` et renvoie **`202 Accepted`**.
2. Le worker réserve un document avec `SELECT … FOR UPDATE SKIP LOCKED`. La file survit aux redémarrages ; un document bloqué en `Processing` au-delà d'un délai est remis en `Pending`.
3. **Extraction page par page** avec PdfPig (Apache 2.0, .NET pur).
4. **Découpage** en chunks d'environ 500 à 800 tokens avec chevauchement. **Un chunk ne déborde jamais sur la page suivante** : c'est ce qui garantit une citation à la page exacte.
5. **Embeddings** calculés par lots, puis insertion en une transaction.
6. Le statut passe à `Ready`, ou à `Failed` avec un message d'erreur.

Le frontend suit la progression par polling de `GET /api/documents` (un flux SSE pourra le remplacer plus tard).

Hors MVP : **OCR des PDF scannés**. Un PDF sans texte extractible est marqué `Failed` avec un message explicite.

## 5. Pipeline de question (RAG + streaming)

`POST /api/ask` → réponse en `text/event-stream`.

1. **Embedding** de la question.
2. **Recherche hybride** :
   - similarité vectorielle (cosinus, top ~20) ;
   - plein texte sur `search_vector` (top ~20) ;
   - fusion par **RRF** (Reciprocal Rank Fusion) pour garder les ~8 meilleurs chunks.

   Le plein texte rattrape les identifiants précis (« ISO 27001 §5.2 », codes, numéros de clause) que les embeddings captent mal.
3. **Génération par Claude** : chaque chunk est envoyé comme bloc **`search_result`** avec `citations: { enabled: true }`.
   - `title` : nom du document ;
   - `source` : identifiant stable `doc:{documentId}:p{page}` ;
   - `content` : le texte du chunk.

   Claude renvoie des citations natives (passage cité + index du bloc), que l'on rattache au chunk, donc à la page. On n'a pas à analyser des `[1]` écrits par le modèle.
4. **Streaming** de Claude vers l'API, puis de l'API vers Angular en SSE (`TypedResults.ServerSentEvents`). Événements émis :

   | Événement  | Contenu                                           |
   |------------|---------------------------------------------------|
   | `text`     | delta de texte                                    |
   | `citation` | `documentId`, `fileName`, `page`, `citedText`     |
   | `done`     | usage (tokens), durée                             |
   | `error`    | message                                           |

5. **Frontend** : `fetch` + `ReadableStream` (`EventSource` ne gère pas le POST). La réponse s'affiche avec des pastilles de citation cliquables qui ouvrent le PDF à la bonne page (pdf.js).

Si aucun chunk n'est assez pertinent, le prompt système demande à Claude de dire que les documents ne contiennent pas la réponse plutôt que d'inventer.

### Configuration Claude

- Modèle configurable dans `appsettings` ; défaut : `claude-opus-5-5`, avec `output_config.effort` à `low` ou `medium` (suffisant pour du Q&A, à mesurer).
- Repli côté serveur activé en cas de refus (`fallbacks: "default"`).
- Toujours vérifier `stop_reason` (`refusal`, `max_tokens`) avant d'exploiter la réponse.
- Prompt système stable en tête de requête, pour profiter du cache de prompt.
- Version d'API `2026-10-21` : une citation renvoyée dans un message `user` provoque une erreur 400. Si le multi-tours est ajouté, l'historique stocke le texte **sans** ses citations.

## 6. Abstractions clés

| Interface            | Implémentation réelle        | Implémentation de test               |
|----------------------|------------------------------|--------------------------------------|
| `IEmbeddingProvider` | Voyage AI                    | embeddings factices déterministes    |
| `IAnswerGenerator`   | client Claude (SDK officiel) | réponse scriptée avec citations      |
| `IPdfTextExtractor`  | PdfPig                       | PDF de test versionnés               |
| `IFileStorage`       | système de fichiers (volume) | répertoire temporaire                |

La CI n'a ainsi besoin d'aucune clé API et ne coûte rien.

## 7. Qualité et outillage

- **Tests unitaires** (xUnit) : découpage, fusion RRF, rattachement des citations, machine à états des documents.
- **Tests d'intégration** : `WebApplicationFactory` + Testcontainers sur un vrai pgvector, de l'upload jusqu'à la réponse streamée.
- **Frontend** : tests de composants et de services (dont le parseur SSE), lint.
- **CI GitHub Actions** : build et tests du backend et du frontend, build des images Docker.
- **Jeu d'évaluation** : 10 à 20 questions annotées avec la page attendue, pour mesurer la qualité de la recherche (recall@k) et suivre les régressions.
- README avec capture animée, instructions `docker compose up` et licence MIT.

## 8. Décisions

| Sujet                    | Décision                               | Alternatives écartées (pour l'instant)             |
|--------------------------|----------------------------------------|----------------------------------------------------|
| Fournisseur d'embeddings | **Voyage AI** (multilingue, quota gratuit ; modèle exact et dimension N à confirmer à l'implémentation) | OpenAI `text-embedding-3-small` ; modèle local ONNX/Ollama |
| Authentification         | **Aucune dans le MVP** (un seul espace de documents) | ASP.NET Identity en v2                   |
| Conversation             | **Question unique dans le MVP**        | Multi-tours avec historique en v2                  |

Grâce à `IEmbeddingProvider`, changer de fournisseur reste possible ; cela impose seulement de recalculer les embeddings (et de migrer la colonne si la dimension change).

## 9. Hors périmètre du MVP

- OCR des PDF scannés
- Multi-utilisateurs, partage, droits d'accès
- Formats autres que PDF (DOCX, HTML…)
- Conversation multi-tours
- Déploiement cloud (la cible du MVP est `docker compose`)
