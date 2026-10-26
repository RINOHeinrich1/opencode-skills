---
name: nextjs-dev
description: >-
  Use when working on a Next.js app (App Router or Pages): server/client
  components, data fetching, caching/revalidation, route handlers, server
  actions, middleware, env vars, or build errors. Trigger on words like
  "next.js", "nextjs", "app router", "use client", "server component",
  "server action", "route.ts", "revalidate", "getServerSideProps", "hydration",
  "middleware". Compensates a known model weakness: never put client-only code
  in a server component (or the reverse) without checking.
---

# Next.js — travailler juste

## Server vs Client Components (App Router)
- Par défaut, **tout est Server Component**. Ajouter `"use client"` **seulement** si le
  fichier utilise état/effets/événements du navigateur/hooks React.
- Un Server Component **ne peut pas** importer un module client dans du code serveur
  exclusif (`fs`, secrets) — et un Client Component **ne peut pas** être `async`.
- Les Server Actions (`"use server"`) tournent **côté serveur** : ne jamais y faire
  confiance aux entrées → valider (zod) et revérifier l'autorisation.
- Ne jamais exposer un secret via une prop passée à un Client Component.

## Data fetching & caching (pièges classiques)
- Fetch dans les **Server Components**, pas dans `useEffect` (legacy/Pages style).
- `fetch()` est mis en cache par défaut (Next) ; contrôle via :
  - `{ cache: 'no-store' }` → dynamique à chaque requête,
  - `{ next: { revalidate: 60 } }` → ISR,
  - `export const dynamic = 'force-dynamic'` / `revalidate = 0`.
- Après une mutation : `revalidatePath('/chemin')` ou `revalidateTag('tag')`.
- Ne pas mettre `no-store` partout « au cas où » → casse le cache et les perfs.

## Params / searchParams (versions récentes)
- Dans les pages/layouts récents, `params` et `searchParams` sont **asynchrones** :
  `const { id } = await params`. Les lire synchronement = erreur de build.

## Env vars
- `NEXT_PUBLIC_*` = exposée au navigateur (jamais un secret).
- Les autres ne sont lisibles que côté serveur.

## Route Handlers & Middleware
- API : `app/api/<route>/route.ts` exportant `GET`/`POST`/... (Web `Request`/`Response`).
- `middleware.ts` s'exécute sur l'edge : pas d'accès DB Node natif, garder léger.
- Toujours renvoyer les bons statuts (`NextResponse.json(body, { status: 400 })`).

## Vérification obligatoire avant de conclure
1. Types : `npx tsc --noEmit` (ou `npm run typecheck`).
2. Lint : `npm run lint`.
3. Build si le changement touche routing/config : `npm run build`.
4. Lire `package.json` et `next.config.*` **avant** de proposer une commande/option —
   ne pas supposer la version de Next ni les scripts disponibles.
