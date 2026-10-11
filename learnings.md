# Important commands

```
sudo docker exec -it postgres psql -U flash -d postgres
```

Open a psql session

# Install the packages we'll need

```bash
pip install "fastapi[standard]" sqlalchemy psycopg[binary] python-dotenv
```

What each one is:

| Package             | Why                                                             |
| ------------------- | --------------------------------------------------------------- |
| `fastapi[standard]` | FastAPI + uvicorn (the server) + nice dev tools                 |
| `sqlalchemy`        | The ORM — Python classes ↔ SQL tables                           |
| `psycopg[binary]`   | Postgres driver (the thing SQLAlchemy uses to talk to Postgres) |
| `python-dotenv`     | Loads secrets/config from a `.env` file                         |

We're **not** installing an async DB driver yet. Let's keep the first version simple with sync SQLAlchemy — you already understand async; you can convert later once the app works. Simpler to debug.
