# NEON VOID — Global Leaderboard (Supabase) Setup

The game works out of the box with a **local** leaderboard (saved in the
browser via `localStorage`). To make the ranking **global** (shared by everyone),
connect a free Supabase project. ~5 minutes, no credit card.

## 1. Create a project
1. Go to <https://supabase.com> → sign in (GitHub login works) → **New project**.
2. Pick a name + region, set a database password, wait for it to provision.

## 2. Create the table + security rules
Open **SQL Editor** → **New query**, paste this, and **Run**:

```sql
create table public.neon_void_scores (
  id         bigint generated always as identity primary key,
  name       text    not null check (char_length(name) between 1 and 14),
  score      integer not null check (score >= 0 and score < 100000000),
  wave       integer not null default 0 check (wave between 0 and 100),
  created_at timestamptz not null default now()
);

-- Row Level Security: anyone can read and insert, nobody can edit/delete.
alter table public.neon_void_scores enable row level security;

create policy "public read"
  on public.neon_void_scores for select to anon using (true);

create policy "public insert"
  on public.neon_void_scores for insert to anon
  with check (
    char_length(name) between 1 and 14
    and score >= 0 and score < 100000000
    and wave between 0 and 100
  );

create index neon_void_scores_score_idx on public.neon_void_scores (score desc);
```

The `anon` key is **meant to be public** — Row Level Security above is what keeps
the table safe (reads + inserts only, with value limits; no updates or deletes).

## 3. Get your keys
**Project Settings → API**, copy:
- **Project URL** → e.g. `https://abcdefgh.supabase.co`
- **Project API keys → `anon` `public`**

## 4. Plug them into the game
Open `neon-void/index.html`, find `LB_CONFIG` near the top of the `<script>`,
and fill in:

```js
const LB_CONFIG = {
  SUPABASE_URL: "https://abcdefgh.supabase.co",
  SUPABASE_ANON_KEY: "eyJhbGciOi...your-anon-key...",
  TABLE: "neon_void_scores",
  TOP_N: 10,
};
```

Commit & push. The game-over screen badge will switch from **LOCAL** to
**GLOBAL** and scores will be shared by all players.

## Notes
- To wipe the board: Supabase → Table Editor → delete rows (or `truncate public.neon_void_scores;`).
- Abuse hardening later (optional): rate-limit by IP with an Edge Function, or
  add a server-side score sanity check. The RLS check constraints already block
  malformed/out-of-range submissions.
