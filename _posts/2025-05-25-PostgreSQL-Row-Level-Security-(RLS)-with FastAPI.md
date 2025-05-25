---
title: Introduction to Row-Level Security (RLS)

author: thrinath
date: 2020-05-25 00:34:00 +0800
categories: [PostgreSQL, PostgreSQL RLS]
tags: [Row-Level Security (RLS), FastApi]
---

Row-Level Security (RLS) is a built-in feature of PostgreSQL that enforces access control policies on individual table rows.Rather than granting or revoking privileges at the table or column level, RLS enables fine-grained control on Row Level access by associating policies with `SELECT`, `INSERT`, `UPDATE`, and `DELETE` operations. Policies are evaluated by the database engine for each row, ensuring access restrictions cannot be circumvented by application-side logic alone.


# Why RLS Matters
In traditional applications, access control logic (e.g., "show only the current user's records") is often implemented in the application layer.

 **Example**

 ```python
    async def get_user_info_by_id(user_id: int, session: AsyncSession):
        # Perform a query to select user info where ID matches the user_id
        stmt = select(UserInfo).where(UserInfo.id == user_id)  # Using WHERE clause
        result = await session.execute(stmt)
        user_info = result.scalars().first()  # Fetch the first result
        
        return user_info

 ```

 This can lead to.:

* Security vulnerabilities (e.g., developer error can expose sensitive rows).

* Duplicated logic across different services.

* Inconsistent enforcement of access policies.

**RLS solves this by enforcing policies inside the database**:

* Guarantees consistent policy enforcement.

* Prevents data leakage due to application bugs.

* Simplifies application code by moving access logic to the database.



## 1.1 Conceptual Overview

* **Policy**: A named rule defined on a table specifying conditions under which an operation is permitted. Policies are written in SQL expressions and attached to tables.
* **Session Variables**: PostgreSQL allows custom configuration parameters (e.g., `app.current_user_id`) to hold contextual data. RLS policies commonly reference these variables to determine row visibility or modifiability.
* **FORCE RLS**: An option that prevents superusers and table owners from bypassing RLS policies, ensuring uniform enforcement across all roles.

### 1.2 Access Control Models

Traditional access control relies on:

1. **Table/Column Privileges**: GRANT/REVOKE on an entire table or column set.
2. **Application-Level Filtering**: Adding `WHERE` clauses in application queries to restrict data.

Limitations of these approaches:

* Application must include correct filters in every query, susceptible to developer error.
* Privileged database roles (e.g., superusers) bypass table-level grants.
* Harder to maintain as the codebase grows.

**RLS** centralizes policy enforcement at the database layer, reducing risk and maintenance overhead.

# 2. RLS vs. WHERE-Clause Filtering

| Aspect               | WHERE-Clause Approach                 | RLS Approach                                |
| -------------------- | ------------------------------------- | ------------------------------------------- |
| Enforcement Location | Application code                      | Database engine                             |
| Bypass Risk          | High (missing filter introduces leak) | Low (policies apply to all sessions)        |
| Policy Granularity   | Per-query                             | Per-table, per-operation                    |
| Superuser Bypass     | Possible                              | Preventable with `FORCE ROW LEVEL SECURITY` |
| Maintenance          | High (update all queries)             | Low (update policy definitions)             |

# 3. Core Components of RLS

## 3.1 Enabling RLS on a Table

```sql
ALTER TABLE posts ENABLE ROW LEVEL SECURITY;
```

This command activates the RLS subsystem for `posts`. Until you create policies, no rows are returned by operations governed by RLS—unless default permissive behavior is allowed.

## 3.2 Policy Definition Syntax

A policy has the form:

```sql
CREATE POLICY policy_name
  ON table_name
  [ FOR { SELECT | INSERT | UPDATE | DELETE } ]
  [ TO role_name [, ...] ]
  [ USING ( using_expression ) ]
  [ WITH CHECK ( check_expression ) ];
```

* **FOR**: Specifies which operations the policy applies to.
* **TO**: Restricts policy to specific roles (optional).
* **USING**: A Boolean expression that rows must satisfy to be visible for `SELECT` or `DELETE`.
* **WITH CHECK**: A Boolean expression that new or updated rows must satisfy for `INSERT` or `UPDATE`.

## 3.3 Policy Evaluation Order

1. All relevant `USING` policies are combined with OR for `SELECT` and `DELETE`.
2. All relevant `WITH CHECK` policies are combined with OR for `INSERT` and `UPDATE`.
3. If no policy matches, the row is denied.

# 4. Detailed Example: `posts` Table

Consider the following schema:

| Column   | Type   | Description                 |
| -------- | ------ | --------------------------- |
| id       | SERIAL | Primary key                 |
| title    | TEXT   | Post title                  |
| content  | TEXT   | Post content                |
| user\_id | INT    | Owner of the post (foreign) |

### 4.1 Creating Table and Sample Data

```sql
CREATE TABLE posts (
  id SERIAL PRIMARY KEY,
  title TEXT NOT NULL,
  content TEXT NOT NULL,
  user_id INT NOT NULL
);

INSERT INTO posts (title, content, user_id) VALUES
  ('First Post',  'Hello World',      1),
  ('Second Post', 'Private post',     2),
  ('Third Post',  'My thoughts here', 1);
```

### 4.2 Enabling and Forcing RLS

```sql
ALTER TABLE posts ENABLE ROW LEVEL SECURITY;
ALTER TABLE posts FORCE ROW LEVEL SECURITY;
```

* **ENABLE**: Turns on RLS for the table.
* **FORCE**: Ensures even superusers cannot bypass policies.

### 4.3 Defining Policies

#### 4.3.1 SELECT Policy

```sql
CREATE POLICY user_select_policy
  ON posts
  FOR SELECT
  USING (
    -- Only allow rows owned by the current user
    user_id = current_setting('app.current_user_id')::int
  );
```

#### 4.3.2 INSERT Policy

```sql
CREATE POLICY user_insert_policy
  ON posts
  FOR INSERT
  WITH CHECK (
    -- New rows must be owned by the current user
    user_id = current_setting('app.current_user_id')::int
  );
```

#### 4.3.3 UPDATE Policy

```sql
CREATE POLICY user_update_policy
  ON posts
  FOR UPDATE
  USING (
    -- Can only update own rows
    user_id = current_setting('app.current_user_id')::int
  )
  WITH CHECK (
    -- Updated row must still belong to the current user
    user_id = current_setting('app.current_user_id')::int
  );
```

#### 4.3.4 DELETE Policy

```sql
CREATE POLICY user_delete_policy
  ON posts
  FOR DELETE
  USING (
    -- Can only delete own rows
    user_id = current_setting('app.current_user_id')::int
  );
```

# 5. Session Variables and `SET LOCAL`

## 5.1 Role of Session Variables

RLS policies often require knowledge of the current application user. PostgreSQL allows setting custom GUC (Grand Unified Configuration) variables:

```sql
SET app.current_user_id = '1';  -- Applies to the entire session
```

These variables are visible via `current_setting(name)`.

## 5.2 Challenges with `SET`

* **Persistence**: A `SET` command persists until the session ends or the variable is reset.
* **Connection Pooling**: In pooled environments, one session can serve many requests. Without cleanup, one user’s context may leak into another’s.

## 5.3 Benefits of `SET LOCAL`

```sql
BEGIN;
SET LOCAL app.current_user_id = '1';
-- ... run queries ...
COMMIT;
```

* **Transaction-Scoped**: Value lasts only for the duration of the transaction.
* **Automatic Cleanup**: On `COMMIT` or `ROLLBACK`, the previous value is restored.
* **Safe for Pooling**: Each transaction gets its own isolated variable setting.

# 6. Integrating with FastAPI and SQLAlchemy

Below is a detailed, commented FastAPI application illustrating RLS enforcement in practice.

```python
from fastapi import FastAPI, Depends, HTTPException
from sqlalchemy import create_engine, Column, Integer, String, text
from sqlalchemy.orm import sessionmaker, Session, declarative_base
from sqlalchemy.future import select
from pydantic import BaseModel
from typing import List

# 6.1 Database Configuration
DATABASE_URL = "postgresql://user:password@localhost/your_db"
engine = create_engine(DATABASE_URL, future=True)
SessionLocal = sessionmaker(
    bind=engine,
    autoflush=False,
    autocommit=False,
    future=True
)
Base = declarative_base()

# 6.2 ORM Model Definition
class Post(Base):
    __tablename__ = "posts"

    # Primary key column
    id = Column(Integer, primary_key=True)
    # Post title
    title = Column(String, nullable=False)
    # Post content
    content = Column(String, nullable=False)
    # Owner user ID
    user_id = Column(Integer, nullable=False)

# 6.3 Pydantic Schema for Validation
class PostSchema(BaseModel):
    id: int
    title: str
    content: str
    user_id: int

    class Config:
        orm_mode = True  # Accept ORM objects directly

# 6.4 Dependency: DB Session per Request

def get_db():
    db: Session = SessionLocal()
    try:
        yield db
    finally:
        db.close()

app = FastAPI()

# 6.5 Startup Event: Initialize RLS Policies
@app.on_event("startup")
def setup_rls():
    with engine.connect() as conn:
        # Enable and enforce RLS
        conn.execute(text("ALTER TABLE posts ENABLE ROW LEVEL SECURITY"))
        conn.execute(text("ALTER TABLE posts FORCE ROW LEVEL SECURITY"))

        # Define policy specifications
        policies = [
            # (policy_name, operation, condition_clause)
            ("user_select_policy", "FOR SELECT",
             "USING (user_id = current_setting('app.current_user_id')::int)"),
            ("user_insert_policy", "FOR INSERT",
             "WITH CHECK (user_id = current_setting('app.current_user_id')::int)"),
            ("user_update_policy", "FOR UPDATE",
             "USING (user_id = current_setting('app.current_user_id')::int) "
             "WITH CHECK (user_id = current_setting('app.current_user_id')::int)"),
            ("user_delete_policy", "FOR DELETE",
             "USING (user_id = current_setting('app.current_user_id')::int)")
        ]

        # Drop existing policies and recreate
        for name, action, condition in policies:
            conn.execute(text(f"DROP POLICY IF EXISTS {name} ON posts"))
            conn.execute(text(f"CREATE POLICY {name} ON posts {action} {condition}"))

# 6.6 GET Endpoint: Fetch User Posts
@app.get("/posts/{user_id}", response_model=List[PostSchema])
def get_posts(user_id: int, db: Session = Depends(get_db)):
    """
    Retrieve all posts for a specific user. The RLS SELECT policy
    automatically filters rows to those owned by user_id.
    """
    try:
        # Begin a transaction block
        with db.begin():
            # Set the session variable with SET LOCAL which is transaction scoped only
            db.execute(
                text("SET LOCAL app.current_user_id = :uid"),
                {"uid": user_id}
            )
            # SELECT query; RLS enforces row-level filtering
            result = db.scalars(select(Post)).all()
            return result
    except Exception:
        # Errors automatically rollback the transaction scope
        raise HTTPException(500, "Failed to fetch posts")

# 6.7 POST Endpoint: Create a New Post
@app.post("/posts/{user_id}", response_model=PostSchema)
def create_post(user_id: int, post: PostSchema, db: Session = Depends(get_db)):
    """
    Insert a new post for the given user. The RLS INSERT policy
    ensures the row’s user_id matches the session variable.
    """
    try:
        with db.begin():
            # Transaction-scoped user context
            db.execute(
                text("SET LOCAL app.current_user_id = :uid"),
                {"uid": user_id}
            )
            # Build new Post object
            new_post = Post(
                title=post.title,
                content=post.content,
                user_id=user_id
            )# get only those rows where uid = user_id 
            db.add(new_post)
            # Flush sends INSERT to DB to generate the ID
            db.flush()
            return new_post
    except Exception:
        raise HTTPException(500, "Failed to create post")
```

# 7. Summary and Best Practices

* **Enable and Force RLS**: Use `ALTER TABLE ... ENABLE` and `FORCE` to activate and lock down policies.
* **Define Specific Policies**: Craft separate policies for each CRUD operation, specifying both `USING` and `WITH CHECK` as needed.
* **Leverage `SET LOCAL`**: Isolate session variables to transactions, preventing cross-request leakage in pooled connections.
* **Centralized Security**: RLS moves access control into the database tier, reducing reliance on application code.
* **Maintainability**: Updating or extending policies is easier than auditing dispersed `WHERE` clauses.

This textbook-style guide provides both theoretical foundations and practical examples for implementing robust, scalable row-level security in PostgreSQL-powered FastAPI applications.
