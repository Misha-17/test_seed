# LingoDeck Microservices — Setup Guide

## First-time setup

### 1. Create a `.env` file in the repo root

```env
DB_PASSWORD=lingo
SERVICE_SECRET=change_me_in_production
LLM_BACKEND=auto           # auto | anthropic | ollama | none
ANTHROPIC_API_KEY=          # optional — for cloud LLM
OLLAMA_HOST=http://localhost:11434
```

### 2. Start everything

```bash
docker compose up --build
```

Services start in this order (handled by depends_on):
  - Postgres (lingo_db) → quest, card, challenge services
  - All three Python services → game-backend

### 3. Seed the wordbank

```bash
# Seed Finnish words
curl -X POST http://localhost:4000/api/admin/quest/seed \
  -H "Authorization: Bearer <your_token>"

# Seed quest challenges
curl -X POST http://localhost:8001/admin/seed-challenges \
  -H "x-service-secret: change_me_in_production"
```

---

## API overview

| Service          | Port | Key routes |
|------------------|------|------------|
| Auth backend     | 3000 | POST /api/users, POST /api/login |
| game-backend     | 4000 | /api/quests/*, /api/cards/*, /api/challenges/* |
| quest-service    | 8001 | /quests/*, /admin/* |
| card-service     | 8002 | /cards/*, /scenarios/* |
| challenge-service| 8003 | /challenges/* |

## Game flow

### Quests
1. GET /api/quests/challenges/:userId  — show challenges & progress
2. POST /api/quests/generate           — get a question (optional scenario_tag)
3. POST /api/quests/submit             — score answer
   → game-backend grants card XP for correct answers
   → game-backend opens pack if challenge completed

### Big Battle
1. GET /api/cards/battle-ready/:userId/:scenario — check eligibility (≥6 cards at ≥2★)
2. POST /api/challenges/start                    — start fight (deck auto-fetched)
3. POST /api/challenges/:id/action               — submit answer each turn
   → game-backend grants card XP for each correct_words_this_turn
4. When status=won: POST /api/scenarios/unlock   — unlock next scenario

## Star / XP system
| Upgrade   | XP needed |
|-----------|-----------|
| 1★ → 2★   | 10        |
| 2★ → 3★   | 30        |
| 3★ → 4★   | 60        |

XP sources: quest correct (1), battle correct (2), duplicate card (5)

Battle gate: need ≥6 cards at ≥2★ for the target scenario.
