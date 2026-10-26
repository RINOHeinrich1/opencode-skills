---
name: supabase-dev
description: >-
  Use when working with Supabase / Postgres: SQL migrations, Row Level Security
  (RLS) policies, auth, RPC/edge functions, storage, realtime, or the JS client.
  Trigger on words like "supabase", "RLS", "policy", "migration SQL", "auth.uid",
  "service_role", "edge function", "realtime", "bucket storage", "gen types",
  "postgres", "pgbouncer". Compensates a known model weakness: never guess the
  schema or policy — inspect it.
---

# Supabase — travailler juste (anti-approximation)

## Règle n°1 : ne JAMAIS deviner le schéma
Avant toute requête/policy/migration, **inspecter la réalité** (MCP outils réels,
CLI ou SQL) au lieu d'inventer des noms de tables/colonnes :
- `supabase migration list` / lire `supabase/migrations/*.sql`
- introspection SQL : `select table_name, column_name, data_type from information_schema.columns where table_schema='public' order by table_name;`
- index : `select indexname, indexdef from pg_indexes where schemaname='public';`
- policies : `select * from pg_policies where schemaname='public';`
- types TS du client : `supabase gen types typescript --linked > src/types/database.types.ts`

## RLS (Row Level Security)
- RLS **activée par défaut** sur les tables du dashboard : sans policy → **tout est refusé**.
- Activer : `alter table public.<t> enable row level security;`
- Policy type : `create policy "<nom>" on public.<t> for <select|insert|update|delete> to authenticated using (auth.uid() = user_id) with check (auth.uid() = user_id);`
- `using` = lignes visibles/éditables ; `with check` = lignes autorisées en écriture (obligatoire pour insert/update).
- `service_role` **contourne RLS** → jamais côté client/navigateur.
- `anon` = non authentifié.
- Fonctions `security definer` : **toujours** fixer `set search_path = ''` sinon faille.
- Indexer les colonnes utilisées dans les policies (sinon full scan).

## Migrations (source de vérité = fichiers, pas le dashboard)
- Nouvelle migration : `supabase migration new <nom>` → éditer `supabase/migrations/<ts>_<nom>.sql`.
- Appliquer : `supabase db push` (distant) / `supabase migration up` (local).
- Vérifier le drift : `supabase db diff`.
- Ne jamais modifier une migration déjà appliquée en prod → en créer une nouvelle.
- Migrations idempotentes quand possible (`if not exists`, `create or replace`).

## Client JS/TS
- Sélectionner explicitement les colonnes (`select('id, name')`) — jamais `*` en prod.
- Joins imbriqués : `select('*, author:profiles(name)')`.
- `.single()` exige pile 1 ligne (erreur sinon) ; `.maybeSingle()` tolère 0.
- Toujours gérer `{ data, error }` — Supabase ne lève pas d'exception.
- Serverless : utiliser le pooler transaction (port **6543**), `?pgbouncer=true` ; port direct **5432** pour migrations.
- Client navigateur = clé `anon` ; serveur privilégié = `service_role` (jamais exposée).

## Realtime
- Publier une table : `alter publication supabase_realtime add table public.<t>;`
- RLS s'applique aussi au realtime → la policy doit laisser voir la ligne.

## Storage
- Bucket privé : créer des policies sur `storage.objects` (même logique RLS). URL signée : `createSignedUrl`.

## Checklist avant de livrer une migration
1. `enable row level security` + policies `using`/`with check` pour TOUS les accès.
2. Index sur FK et colonnes de policy.
3. Testé localement (`supabase start`, `supabase db reset`) ou en branche/preprod.
4. Types régénérés (`gen types`) si le client TS est typé.
