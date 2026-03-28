# Database Concepts & API Design in PostgreSQL

## 1. Core Database Concepts
* **Persistence:** Allowing data to survive sessions. Databases typically run on persistent disk storage (SSD/HDD), which is cheaper and offers higher capacity but is slower compared to primary RAM (which is reserved for speed-focused caches like Redis).
* **DBMS (Database Management System):** The software responsible for data organization, CRUD access (Create, Read, Update, Delete), data integrity validation, and setting security constraints.
* **Why not text files?** Storing data in plain text files suffers from slow sequential parsing, massive concurrency conflicts (race conditions on overwriting), and a complete lack of format structure.

## 2. Choosing a Database
* **Relational (SQL):** Enforces a strict schema prior to insertion. Data lives in tables, rows, and columns. Offers unmatched data integrity. Examples: **PostgreSQL**, MySQL. Great for structured needs (e.g., CRM systems).
* **Non-Relational (NoSQL):** Schemaless, documents-collections model. Fast prototype speed, highly flexible for mixed content. Examples: MongoDB. Great for mixed or fluid structures (e.g., Content Management Systems).
* **Why PostgreSQL?** Open-source, strictest adherence to SQL standards natively, incredibly extensible, and features robust, searchable `JSON/JSONB` support (which practically eliminates the need for NoSQL databases when handling dynamic data requirements).

## 3. Postgres Data Types & Best Practices
* **IDs & Serials:** `UUID` is best practice for IDs to prevent malicious ID guessing in APIs via standard sequential counting (vs `bigserial`).
* **Strings (`text` vs `varchar` vs `char`):**
    * Avoid `char(n)` unless the string is guaranteed to be universally constant in size.
    * **Always default to `text`**. In Postgres, `varchar(255)` is an arbitrary legacy convention imported from MySQL. Under the hood, `varchar` and `text` perform identically. Hard-limiting length through DB types causes messy, risky database migrations later; prefer dynamic length validations in backend application code instead.
* **Numbers:**
    * Use `decimal`/`numeric` for absolute, precise accuracy (crucial for pricing & calculating math).
    * Use floating-point types (`float`, `real`) only where slight marginal precision loss is acceptable in exchange for incredibly fast calculations.
* **JSON:** Default to `JSONB`. `JSONB` is parsed and stored in binary formats underneath, drastically optimizing query performance and allowing granular indexing natively.

## 4. Modeling & Relationships
* **Primary Keys:** Uniquely identifies rows. Implies strictly `UNIQUE` and `NOT NULL`.
* **Referential Integrity Constraints:** Protecting data chains when declaring Foreign Keys:
    * `ON DELETE RESTRICT`: Blocks the deletion of a parent record if connected child records exist.
    * `ON DELETE CASCADE`: Deleting the parent implicitly and automatically deletes all chained children.
* **Relationship Architectures:**
    * *One-to-One (1:1)*: (e.g., `users` -> `user_profiles`) Child table uses its Foreign Key identically as its Primary Key.
    * *One-to-Many (1:N)*: (e.g., `projects` -> `tasks`) The `tasks` table individually stores `project_id`.
    * *Many-to-Many (N:N)*: (e.g., `projects` <-> `users`) Implemented through a distinct Linking/Join table (e.g., `project_members`) utilizing a *Composite Primary Key* formed by tying both targeted foreign IDs together.

## 5. Migrations & Seeding
* **Migrations:** Reproducible files that track database structure changes over time (often via tools like `dbmate`).
    * **Up migrations:** Executes the schema change (e.g. creating tables/indexes). 
    * **Down migrations:** Perfectly reverts the respective Up action (used to safely rollback production emergencies).
* **Seeding:** Injecting localized dummy test records into databases immediately so that developers can test real API structures.

## 6. Optimization, Indexing & Triggers
* **Indexes:** A database lookup table (similar to a literal book index by page numbers) optimizing query speed for frequently targeted columns. This blocks highly inefficient "sequential scans" (searching block-by-block).
    * *When to Index?* Index columns routinely spotted in `WHERE` clauses, `JOIN ... ON` arguments, or `ORDER BY` directions. 
    * *Trade-off:* Adds lightweight insertion overhead on edits. Evaluate frequency before blindly indexing.
* **Triggers / PL/pgSQL Functions:** Programs that script DB-level logic automatically in response to events. 
    * *Classic use case:* Updating an `updated_at` column timestamp exactly whenever its row receives an `UPDATE`, keeping backend code completely agnostic.
    * **How to test triggers:** To test if an `updated_at` trigger is working:
        1. **Check current state:** `SELECT id, updated_at FROM users LIMIT 1;` Note the exact timestamp.
        2. **Perform an arbitrary update:** `UPDATE users SET full_name = 'Trigger Test' WHERE id = '<id-from-step-1>';` Let the database process it.
        3. **Verify modification:** `SELECT id, updated_at FROM users WHERE id = '<id-from-step-1>';`
        *If the trigger fired correctly, the `updated_at` time will have advanced to the exact moment you ran step 2 securely inside Postgres, without you manually declaring `SET updated_at = ...` in the query itself.*

---

## Demonstrated API SQL Queries & Full API Endpoint Map

Below is a complete mapping of the API endpoints from the discussed design, showing the underlying parameterized SQL queries (binding dynamic runtime inputs like `:id`, `:email` as `$1`, `$2`, etc. to prevent SQL injection).

### Users API
Endpoints handling the core user demographic and relationship mappings.

*   `GET /api/users` (With Sorting & Pagination)
    ```sql
    SELECT u.*, to_jsonb(up.*) as profile 
    FROM users u
    LEFT JOIN user_profiles up ON u.id = up.user_id
    ORDER BY u.created_at DESC
    LIMIT $1 OFFSET $2;
    ```
*   `GET /api/users/:id`
    ```sql
    SELECT u.*, to_jsonb(up.*) as profile FROM users u
    LEFT JOIN user_profiles up ON u.id = up.user_id
    WHERE u.id = $1;
    ```
*   `POST /api/users`
    ```sql
    INSERT INTO users (email, full_name, password_hash)
    VALUES ($1, $2, $3)
    RETURNING *;
    ```
*   `PATCH /api/users/:id` (Dynamic update of profile data)
    ```sql
    UPDATE user_profiles 
    SET bio = COALESCE($1, bio), phone = COALESCE($2, phone), avatar_url = COALESCE($3, avatar_url)
    WHERE user_id = $4
    RETURNING *;
    ```
*   `DELETE /api/users/:id`
    ```sql
    DELETE FROM users WHERE id = $1; 
    -- Handled cascaded deletions via ON DELETE CASCADE in dependent tables seamlessly.
    ```
*   `GET /api/users/:id/stats`
    ```sql
    SELECT 
        (SELECT count(*) FROM projects WHERE owner_id = $1) as owned_projects,
        (SELECT count(*) FROM tasks WHERE assigned_to = $1) as assigned_tasks;
    ```

### Projects API
Endpoints for manipulating core project nodes and team structures.

*   `GET /api/projects`
    ```sql
    SELECT * FROM projects ORDER BY created_at DESC;
    ```
*   `GET /api/projects/:id`
    ```sql
    SELECT * FROM projects WHERE id = $1;
    ```
*   `POST /api/projects`
    ```sql
    INSERT INTO projects (name, description, status, owner_id)
    VALUES ($1, $2, $3, $4)
    RETURNING *;
    ```
*   `PATCH /api/projects/:id`
    ```sql
    UPDATE projects 
    SET name = COALESCE($1, name), description = COALESCE($2, description), status = COALESCE($3, status)
    WHERE id = $4 RETURNING *;
    ```
*   `DELETE /api/projects/:id`
    ```sql
    DELETE FROM projects WHERE id = $1;
    ```
*   `GET /api/projects/:id/tasks`
    ```sql
    SELECT * FROM tasks WHERE project_id = $1 ORDER BY due_date ASC;
    ```
*   `GET /api/projects/:id/members`
    ```sql
    SELECT u.id, u.email, u.full_name, pm.role 
    FROM users u
    JOIN project_members pm ON u.id = pm.user_id
    WHERE pm.project_id = $1;
    ```
*   `POST /api/projects/:id/members` (Add user to project)
    ```sql
    INSERT INTO project_members (project_id, user_id, role)
    VALUES ($1, $2, $3)
    RETURNING *;
    ```
*   `PUT /api/projects/:id/members/:userId` (Modify user access/role)
    ```sql
    UPDATE project_members 
    SET role = $1 
    WHERE project_id = $2 AND user_id = $3
    RETURNING *;
    ```
*   `DELETE /api/projects/:id/members/:userId`
    ```sql
    DELETE FROM project_members 
    WHERE project_id = $1 AND user_id = $2;
    ```

### Tasks API
Managing discrete sub-tasks linked to specific projects and individual assignments.

*   `GET /api/tasks/:id`
    ```sql
    SELECT * FROM tasks WHERE id = $1;
    ```
*   `POST /api/tasks`
    ```sql
    INSERT INTO tasks (project_id, title, description, priority, due_date, assigned_to)
    VALUES ($1, $2, $3, $4, $5, $6)
    RETURNING *;
    ```
*   `PUT /api/tasks/:id`
    ```sql
    UPDATE tasks 
    SET title = COALESCE($1, title), priority = COALESCE($2, priority), status = COALESCE($3, status)
    WHERE id = $4
    RETURNING *;
    ```
*   `DELETE /api/tasks/:id`
    ```sql
    DELETE FROM tasks WHERE id = $1;
    ```
*   `GET /api/tasks/overdue`
    ```sql
    SELECT * FROM tasks 
    WHERE due_date < CURRENT_DATE 
      AND status NOT IN ('completed', 'cancelled')
    ORDER BY due_date ASC;
    ```
*   `GET /api/users/:id/tasks` (User's specific assignments)
    ```sql
    SELECT * FROM tasks WHERE assigned_to = $1 ORDER BY current_timestamp DESC;
    ```

### Search API
Fuzzy searching across core entities using `ILIKE` pattern-matching. Assuming the parameter passed to `$1` has already been wildcarded (e.g., `%query%`).

*   `GET /api/search/users?q=query`
    ```sql
    SELECT * FROM users 
    WHERE full_name ILIKE $1 OR email ILIKE $1;
    ```
*   `GET /api/search/projects?q=query`
    ```sql
    SELECT * FROM projects 
    WHERE name ILIKE $1 OR description ILIKE $1;
    ```
*   `GET /api/search/tasks?q=query`
    ```sql
    SELECT * FROM tasks 
    WHERE title ILIKE $1;
    ```
