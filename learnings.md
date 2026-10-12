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

# Module in python

## 1. How the Import Works in Your Code

- **`from app.database import engine`**
- **`app`** is the folder (`app/`) inside your project root.
- **`database`** is the file (`database.py`) inside that folder.
- **`engine`** is the specific variable/object you are importing out of `database.py`.

- **`from sqlalchemy import text`**
- This imports a package (`sqlalchemy`) installed globally or in your virtual environment (`.venv`), similar to `require('sqlalchemy')` or an npm package.

---

## 2. Python vs. Node.js (Mental Model Shift)

| Feature            | Node.js (`require` / `import`)                                      | Python (`import`)                                                                    |
| ------------------ | ------------------------------------------------------------------- | ------------------------------------------------------------------------------------ |
| **Path Style**     | File-system relative strings (e.g., `./database`, `../models/user`) | Dot-separated module hierarchy starting from the project root (e.g., `app.database`) |
| **Context**        | Relative to the _current file's location_                           | Relative to where the script is run / project root (**PYTHONPATH**)                  |
| **File Extension** | Optional or omitted (`.js`, `.ts`)                                  | Omitted entirely (Python looks for `.py` files or packages)                          |

---

## 3. Why `app.database` works

Python determines import paths based on the directory where you run your command, or where your project root is configured. Because your `app` folder is at the root of your workspace, Python recognizes `app` as a package (thanks to the `__init__.py` file inside it).

If you were inside `app/database.py` and wanted to import something else in the same folder, you _could_ use a relative import with a dot (e.g., `from .models import User`), but for cross-folder or absolute-style workspace imports, Python prefers starting from the root package name (`app.something`).

Coming from Node.js, `__init__.py` is one of the biggest quirks to get used to. Think of it as Python's way of explicitly declaring: **"Hey, treat this folder as a package/module container."**

Here is what it does and why it matters:

## 1. The Marker for a "Package"

In older versions of Python (Python 2 and early Python 3), a folder **had** to contain an `__init__.py` file (even if it was completely empty) for Python to recognize that folder as a package you could import from.

- Without `__init__.py`, Python would just see `app/` as a random folder on your hard drive, and `from app.database import engine` would throw a `ModuleNotFoundError`.
- With `__init__.py`, Python knows `app` is a Python package, allowing you to use dot-notation to access files inside it.

_(Note: In modern Python 3, "implicit namespace packages" exist, meaning folders without `__init__.py` can sometimes work, but best practice is still to include `__init__.py`—or leave it empty—so your packages behave predictably across all environments and tools)._

---

## 2. What Else Can You Do Inside `__init__.py`?

While it can be empty, `__init__.py` is actually executed the moment any file inside that folder is imported. This makes it a handy place for a few things:

- **Initialization Code:** Running setup code or database checks when the package loads.
- **Flattening Imports:** If you have deep nesting, you can expose inner functions/classes right at the package level inside `__init__.py`.
  For example, if your `__init__.py` inside `app/` has:

```python
from app.database import engine

```

Then elsewhere in your project, you can do:

```python
from app import engine  # Instead of from app.database import engine

```

---

### Node.js Comparison

In Node.js, any folder can act like a module if it has a `package.json` with a `main` field, or simply by pointing to an `index.js` file inside that folder (`require('./app')` automatically looks for `app/index.js`).

In Python, **`__init__.py` is essentially Python's version of `index.js` or `package.json**` for directories—it defines the entry point and identity of a package.

# Fast API basics

```python
engine = create_engine(DATABASE_URL)
SessionLocal = sessionmaker(bind=engine, autoflush=False, autocommit=False)
Base = declarative_base()
```

---

## 1. `engine = create_engine(DATABASE_URL)`

```python
engine = create_engine(DATABASE_URL)
```

This creates the **database engine**.

Think of the engine as the main “database connection manager” for your app.

- It knows the database URL, like:
  ```text
  postgresql://user:password@localhost/mydb
  ```
  or:
  ```text
  sqlite:///./test.db
  ```
- It manages a pool of connections.
- It does **not** actually connect to the database immediately.
- You usually create **one engine** for your whole app.

So:

```python
engine = create_engine(DATABASE_URL)
```

means: “Create the thing that knows how to connect to my database.”

---

## 2. `SessionLocal = sessionmaker(...)`

```python
SessionLocal = sessionmaker(bind=engine, autoflush=False, autocommit=False)
```

This creates a **session factory**.

Important: `SessionLocal` is not a session itself. It is a class/factory that creates sessions.

Later you do:

```python
db = SessionLocal()
```

Now `db` is an actual database session.

A session is like a temporary conversation/workspace with the database. You use it to:

- query data
- add new rows
- update rows
- delete rows
- commit changes
- rollback changes
- close the connection

The options:

```python
bind=engine
```

Means: sessions created by `SessionLocal` should use the `engine` we created above.

```python
autoflush=False
```

Means: SQLAlchemy should not automatically send pending changes to the database before every query. You control when things are flushed.

```python
autocommit=False
```

Means: SQLAlchemy should not automatically save changes. You must explicitly call:

```python
db.commit()
```

If you forget `db.commit()`, your changes usually will not be saved.

So this line:

```python
SessionLocal = sessionmaker(bind=engine, autoflush=False, autocommit=False)
```

means: “Create a factory that gives me database sessions using this engine, and I will manually control commits.”

---

## 3. `Base = declarative_base()`

```python
Base = declarative_base()
```

This creates a **base class** for your database models.

In SQLAlchemy, you define database tables as Python classes. Those classes inherit from `Base`.

Example:

```python
from sqlalchemy import Column, Integer, String

class User(Base):
    __tablename__ = "users"

    id = Column(Integer, primary_key=True, index=True)
    name = Column(String)
    email = Column(String, unique=True)
```

Here:

- `User` is a Python class.
- SQLAlchemy maps it to a database table called `users`.
- `id`, `name`, and `email` become columns.
- `Base` keeps track of all models that inherit from it.

You can later create the tables with:

```python
Base.metadata.create_all(bind=engine)
```

That tells SQLAlchemy: “Create all tables for the models that inherit from `Base`.”

So:

```python
Base = declarative_base()
```

means: “Create the parent class that all my database models will inherit from.”

---

## How they work together

The flow is roughly:

```text
DATABASE_URL
    |
    v
engine  ---> knows how to connect to the database
    |
    v
SessionLocal  ---> factory that creates database sessions
    |
    v
db = SessionLocal()  ---> actual session used in your FastAPI route
    |
    v
Base  ---> parent class for your models, like User, Item, Post
```

Typical FastAPI usage:

```python
from sqlalchemy.orm import Session

def get_db():
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()
```

Then in a route:

```python
@app.get("/users")
def read_users(db: Session = Depends(get_db)):
    return db.query(User).all()
```

And a model:

```python
class User(Base):
    __tablename__ = "users"

    id = Column(Integer, primary_key=True, index=True)
    name = Column(String)
```

---

## Simple summary

```python
engine = create_engine(DATABASE_URL)
```

Creates the database engine/connection manager.

```python
SessionLocal = sessionmaker(bind=engine, autoflush=False, autocommit=False)
```

Creates a factory that makes database sessions. You use `SessionLocal()` to get a session.

```python
Base = declarative_base()
```

Creates the base class for your database models. Your models like `User`, `Item`, etc. inherit from it.
