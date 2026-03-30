# Notes Organizer DB Schema (PostgreSQL)

This container uses PostgreSQL and is reachable via the CLI command stored in:

- `notes_db/db_connection.txt` (example: `psql postgresql://appuser:dbuser123@localhost:5000/myapp`)

## Entities

### `users`
Represents an authenticated user account.

Columns:
- `id` (bigserial, PK)
- `email` (citext, unique, required)
- `password_hash` (text, required) — stored hash (implementation decided by backend)
- `display_name` (text, optional)
- `created_at` (timestamptz, default now)
- `updated_at` (timestamptz, default now; maintained by trigger)

### `notes`
Represents a note owned by a user.

Columns:
- `id` (bigserial, PK)
- `user_id` (bigint, FK -> users.id, on delete cascade)
- `title` (text, default '')
- `content` (text, default '')
- `is_pinned` (boolean, default false)
- `is_favorite` (boolean, default false)
- `created_at` (timestamptz, default now)
- `updated_at` (timestamptz, default now; maintained by trigger)

### `tags`
Represents a tag owned by a user. Tag names are unique per user (case-insensitive).

Columns:
- `id` (bigserial, PK)
- `user_id` (bigint, FK -> users.id, on delete cascade)
- `name` (citext, required)
- `created_at` (timestamptz, default now)

Constraints:
- `UNIQUE(user_id, name)`

### `note_tags`
Join table for many-to-many between `notes` and `tags`.

Columns:
- `note_id` (bigint, FK -> notes.id, on delete cascade)
- `tag_id` (bigint, FK -> tags.id, on delete cascade)
- `created_at` (timestamptz, default now)

Constraints:
- `PRIMARY KEY(note_id, tag_id)`

## Extensions Used

- `citext` for case-insensitive email/tag name uniqueness.
- `pg_trgm` for trigram search indexes on notes.

## Indexes

- `idx_notes_user_updated_at` on `notes(user_id, updated_at desc)` — listing by recency
- `idx_notes_user_pinned_updated_at` on `notes(user_id, is_pinned desc, updated_at desc)` — pinned-first ordering
- `idx_notes_user_favorite_updated_at` on `notes(user_id, is_favorite desc, updated_at desc)` — favorite filtering/sorting
- `idx_tags_user_name` on `tags(user_id, name)` — tag lookup
- `idx_note_tags_tag_id` on `note_tags(tag_id)` — notes-by-tag lookup
- `idx_notes_title_trgm` GIN trigram on `notes.title`
- `idx_notes_content_trgm` GIN trigram on `notes.content`

## updated_at Triggers

A single trigger function is used:

- `set_updated_at()` — sets `NEW.updated_at = now()` on update

Triggers:
- `trg_users_updated_at` on `users`
- `trg_notes_updated_at` on `notes`

## Minimal Seed Data (for local verification)

The following are inserted (idempotently):
- user: `demo@example.com`
- tags: `work`, `personal`
- note: `Welcome` (pinned + favorite)
- relationship: `Welcome` tagged with `work`

## Applying the Schema Manually (Idempotent)

All statements are designed to be safe to run multiple times.

From `notes_db/`:

```bash
CONN="$(cat db_connection.txt)"

# Extensions
$CONN -v ON_ERROR_STOP=1 -c "CREATE EXTENSION IF NOT EXISTS citext;"
$CONN -v ON_ERROR_STOP=1 -c "CREATE EXTENSION IF NOT EXISTS pg_trgm;"

# Tables
$CONN -v ON_ERROR_STOP=1 -c "CREATE TABLE IF NOT EXISTS users (id BIGSERIAL PRIMARY KEY, email CITEXT NOT NULL UNIQUE, password_hash TEXT NOT NULL, display_name TEXT, created_at TIMESTAMPTZ NOT NULL DEFAULT now(), updated_at TIMESTAMPTZ NOT NULL DEFAULT now());"
$CONN -v ON_ERROR_STOP=1 -c "CREATE TABLE IF NOT EXISTS notes (id BIGSERIAL PRIMARY KEY, user_id BIGINT NOT NULL REFERENCES users(id) ON DELETE CASCADE, title TEXT NOT NULL DEFAULT '', content TEXT NOT NULL DEFAULT '', is_pinned BOOLEAN NOT NULL DEFAULT FALSE, is_favorite BOOLEAN NOT NULL DEFAULT FALSE, created_at TIMESTAMPTZ NOT NULL DEFAULT now(), updated_at TIMESTAMPTZ NOT NULL DEFAULT now());"
$CONN -v ON_ERROR_STOP=1 -c "CREATE TABLE IF NOT EXISTS tags (id BIGSERIAL PRIMARY KEY, user_id BIGINT NOT NULL REFERENCES users(id) ON DELETE CASCADE, name CITEXT NOT NULL, created_at TIMESTAMPTZ NOT NULL DEFAULT now(), UNIQUE(user_id, name));"
$CONN -v ON_ERROR_STOP=1 -c "CREATE TABLE IF NOT EXISTS note_tags (note_id BIGINT NOT NULL REFERENCES notes(id) ON DELETE CASCADE, tag_id BIGINT NOT NULL REFERENCES tags(id) ON DELETE CASCADE, created_at TIMESTAMPTZ NOT NULL DEFAULT now(), PRIMARY KEY(note_id, tag_id));"

# Indexes
$CONN -v ON_ERROR_STOP=1 -c "CREATE INDEX IF NOT EXISTS idx_notes_user_updated_at ON notes(user_id, updated_at DESC);"
$CONN -v ON_ERROR_STOP=1 -c "CREATE INDEX IF NOT EXISTS idx_notes_user_pinned_updated_at ON notes(user_id, is_pinned DESC, updated_at DESC);"
$CONN -v ON_ERROR_STOP=1 -c "CREATE INDEX IF NOT EXISTS idx_notes_user_favorite_updated_at ON notes(user_id, is_favorite DESC, updated_at DESC);"
$CONN -v ON_ERROR_STOP=1 -c "CREATE INDEX IF NOT EXISTS idx_tags_user_name ON tags(user_id, name);"
$CONN -v ON_ERROR_STOP=1 -c "CREATE INDEX IF NOT EXISTS idx_note_tags_tag_id ON note_tags(tag_id);"
$CONN -v ON_ERROR_STOP=1 -c "CREATE INDEX IF NOT EXISTS idx_notes_title_trgm ON notes USING GIN (title gin_trgm_ops);"
$CONN -v ON_ERROR_STOP=1 -c "CREATE INDEX IF NOT EXISTS idx_notes_content_trgm ON notes USING GIN (content gin_trgm_ops);"

# Trigger function (note the escaped $$ in bash)
$CONN -v ON_ERROR_STOP=1 -c "CREATE OR REPLACE FUNCTION set_updated_at() RETURNS TRIGGER AS \$\$ BEGIN NEW.updated_at = now(); RETURN NEW; END; \$\$ LANGUAGE plpgsql;"

# Triggers (drop first to keep idempotent)
$CONN -v ON_ERROR_STOP=1 -c "DROP TRIGGER IF EXISTS trg_users_updated_at ON users;"
$CONN -v ON_ERROR_STOP=1 -c "DROP TRIGGER IF EXISTS trg_notes_updated_at ON notes;"
$CONN -v ON_ERROR_STOP=1 -c "CREATE TRIGGER trg_users_updated_at BEFORE UPDATE ON users FOR EACH ROW EXECUTE FUNCTION set_updated_at();"
$CONN -v ON_ERROR_STOP=1 -c "CREATE TRIGGER trg_notes_updated_at BEFORE UPDATE ON notes FOR EACH ROW EXECUTE FUNCTION set_updated_at();"

# Seed
$CONN -v ON_ERROR_STOP=1 -c "INSERT INTO users (email, password_hash, display_name) VALUES ('demo@example.com', 'demo-password-hash', 'Demo User') ON CONFLICT (email) DO NOTHING;"
$CONN -v ON_ERROR_STOP=1 -c "INSERT INTO tags (user_id, name) SELECT id, 'work' FROM users WHERE email='demo@example.com' ON CONFLICT (user_id, name) DO NOTHING;"
$CONN -v ON_ERROR_STOP=1 -c "INSERT INTO tags (user_id, name) SELECT id, 'personal' FROM users WHERE email='demo@example.com' ON CONFLICT (user_id, name) DO NOTHING;"
$CONN -v ON_ERROR_STOP=1 -c "INSERT INTO notes (user_id, title, content, is_pinned, is_favorite) SELECT id, 'Welcome', 'This is a seeded note for local verification.', TRUE, TRUE FROM users WHERE email='demo@example.com' ON CONFLICT DO NOTHING;"
$CONN -v ON_ERROR_STOP=1 -c "INSERT INTO note_tags (note_id, tag_id) SELECT n.id, t.id FROM notes n JOIN users u ON u.id=n.user_id JOIN tags t ON t.user_id=u.id WHERE u.email='demo@example.com' AND n.title='Welcome' AND t.name='work' ON CONFLICT DO NOTHING;"
```
