# Song Recommender with LangGraph Memory

A command-line music recommender built with LangGraph. It uses OpenRouter for structured conversational responses, PostgreSQL for graph checkpoints and store data, and SQLite for persistent user preferences.

## Features

- Interactive Portuguese-language chat with an OpenRouter-hosted model.
- Separate conversation threads, with a persistent user ID for saved preferences.
- Structured preference extraction for names, genres, artists, and other context.
- Conversation summarization to retain useful context while reducing older message history.
- PostgreSQL-backed LangGraph checkpointing and store.
- SQLite-backed preference records managed by Knex.

## How It Works

```text
START
  |
  v
 chat -------------------------------> END
  |                 |
  |                 +--> summarize --> END
  +--> savePreferences --> summarize
             |
             +-----------------------> END
```

The chat node generates a response and may extract preferences. The graph saves extracted preferences, and summarizes longer conversations when needed. Conversation state is checkpointed in PostgreSQL. User preference summaries are stored separately in `preferences.db` and can be reused by later conversations for the same user ID.

The CLI creates a new thread ID each time it starts. Passing the same `--user` value across runs reuses that user's saved preferences, but starts a fresh conversation thread.

## Requirements

- Node.js 22 or newer
- pnpm
- Docker Compose, for the local PostgreSQL service
- An OpenRouter API key

## Setup

Install the declared dependencies:

```bash
pnpm install
```

Create a local environment file and add your OpenRouter key:

```bash
cp .env.example .env
```

Edit `.env` and set `OPENROUTER_API_KEY`. The optional LangSmith variables in the example enable tracing when configured.

Start PostgreSQL:

```bash
docker compose up -d postgres
```

The current database connection string is hard-coded in `src/config.ts` and matches the local Compose service defaults. Those credentials are for local development only; change the configuration before using a non-local database.

## Run the Chat

The chat script loads `.env` and starts the CLI:

```bash
pnpm run chat -- --user alex
```

The `--user` argument is optional. If omitted, the CLI uses `anonymous`. Type `exit` to end the chat.

## Tests

The end-to-end test calls the configured model and requires a running PostgreSQL service and a valid OpenRouter key. Run it with:

```bash
pnpm test
```

## Configuration

| Setting | Source | Purpose |
| --- | --- | --- |
| `OPENROUTER_API_KEY` | Environment | Authenticates model requests. |
| `OPENROUTER_MODEL` | Environment, optional | Overrides the default free model (`baidu/ernie-5.5-flash-preview:free`). Must end in `:free`; check OpenRouter's current free model catalog if the default is unavailable. |
| LangSmith tracing variables | Environment | Optional LangSmith tracing, as shown in `.env.example`. |
| Model, routing, temperature, title, and summary threshold | `src/config.ts` | Controls OpenRouter requests and summarization behavior. |
| PostgreSQL connection string | `src/config.ts` | Used by the LangGraph PostgreSQL checkpointer and store. |
| SQLite preference database path | `src/graph/factory.ts` | Defaults to `./preferences.db`. |

`OPENROUTER_HTTP_REFERER` and `OPENROUTER_X_TITLE` appear in `.env.example`, but the current configuration does not read those environment variables; the corresponding values are set in `src/config.ts`.

## Project Structure

```text
src/
  config.ts                      # Model and database configuration
  index.ts                       # Interactive CLI
  graph/
    factory.ts                   # Builds services and the graph
    graph.ts                     # State schema and graph routing
    nodes/
      chatNode.ts                # Response generation and preference extraction
      edgeConditions.ts          # Conditional graph routing
      savePreferencesNode.ts     # Persists extracted preferences
      summarizationNode.ts       # Summarizes conversation history
  prompts/v1/                    # Structured response and summary prompts
  services/
    memoryService.ts             # PostgreSQL checkpointer and store
    openrouterService.ts         # OpenRouter-compatible LangChain client
    preferencesService.ts        # SQLite persistence through Knex
tests/
  chat.e2e.test.ts               # End-to-end conversation test
```

`langgraph.json` exports the graph factory as `song_highlights` for LangGraph tooling.

## License

MIT