# slopcheck

Agent instructions for catching and fixing the performance slop AI-generated ("vibe coded") sites ship with — auditing, fixing, and verifying web/webapp performance. Triggers whenever the user asks to "optimize performance", "speed up the site/app", "run a perf audit", "clean up the slop", or pastes a checklist like the one below.

20-item checklist, grouped by layer. For every item: **Detect** current state → **Fix** if missing → **Verify** with a concrete, re-runnable check (not a guess). Never mark an item done without evidence (grep hit, header value, `EXPLAIN` output, bundle size number, Lighthouse score). Unverifiable = not done, never assumed done.

## Workflow

1. **Scope**: confirm target (repo path, deployed URL if live verification is wanted, stack — Node/Next/Django/Rails/etc). If a live URL exists and browser automation is available (Playwright, Chrome DevTools, `lighthouse` CLI), use it for real measurements. If only source is available, verify statically (grep, config inspection, migration files).
2. **Audit pass**: run Detect for all 20 items below. If your environment supports parallel/sub-tasked tool calls, use them for independent checks — the items don't depend on each other.
3. **Fix pass**: for every item found missing, implement the Fix. Cross-cutting items (load balancer, CDN — usually infra config, not code) may need infra access outside the repo; flag those explicitly instead of faking a fix.
4. **Verify pass**: re-run Detect for everything touched. Only mark an item done on a passing re-check.
5. **Report**: emit the checklist back with one line of evidence per completed item, e.g. `Index the Database — idx_orders_user_id on orders(user_id), EXPLAIN shows Index Scan`.

---

## A. Database

### 1. Index the Database
- **Detect**: grep migrations/schema for `CREATE INDEX` / `index: true` / Prisma `@@index`. Cross-check against slow query log or `EXPLAIN ANALYZE` on the app's hottest queries (find via ORM call sites for WHERE/JOIN/ORDER BY on foreign keys).
- **Fix**: add index on columns used in WHERE, JOIN, ORDER BY, and foreign keys lacking one. Migration file, not manual `ALTER TABLE` on prod.
- **Verify**: `EXPLAIN ANALYZE` on the target query shows Index Scan/Index Only Scan, not Seq Scan; query time drops.

### 2. Database connection pooling
- **Detect**: check DB client init for a pool config (`pg.Pool`, `mysql2.createPool`, Prisma `connection_limit`, SQLAlchemy `pool_size`, PgBouncer/RDS Proxy in infra). Absence = raw per-request connections.
- **Fix**: wrap client in a pool with sane min/max sized to DB max_connections / concurrent workers. Prefer external pooler (PgBouncer, RDS Proxy) for serverless/multi-instance apps.
- **Verify**: connection count under load stays bounded (`SELECT count(*) FROM pg_stat_activity` or equivalent) instead of climbing with request rate.

### 3. Remove N+1 database queries
- **Detect**: find loops that call the ORM/DB inside a `for`/`map` over a parent list (classic pattern: fetch list, then per-item fetch related data). Confirm via query log count during one request — N+1 shows as 1 + N queries instead of 1-2.
- **Fix**: batch with `include`/`with`/`joinedload`/`select_related`/`prefetch_related`/GraphQL DataLoader, or a single JOIN/`WHERE id IN (...)` query.
- **Verify**: query log for the same request now shows constant query count regardless of list size.

### 4. Cache expensive queries
- **Detect**: identify queries with high cost (aggregations, large joins, reporting endpoints) via `EXPLAIN ANALYZE` cost/time. Check if results are cached (Redis/Memcached wrapper, materialized view, in-memory cache) or re-run every request.
- **Fix**: cache result behind Redis/Memcached with TTL matched to data staleness tolerance, or materialized view refreshed on schedule/trigger.
- **Verify**: second identical request served from cache (cache hit log/metric, or query log shows DB not hit on repeat call within TTL).

### 5. Paginate large lists
- **Detect**: grep endpoints/queries returning lists for missing `LIMIT`/`OFFSET`, cursor param, or `take`/`skip`. Check response size on an endpoint backed by a large table.
- **Fix**: add limit/offset or cursor-based pagination (cursor preferred for large/changing datasets); cap max page size server-side.
- **Verify**: endpoint response size bounded regardless of underlying table size; response includes pagination metadata (next cursor / total pages).

---

## B. Backend / API

### 6. Cache API responses
- **Detect**: `curl -I` the endpoint, check for `Cache-Control`/`ETag`/`Last-Modified` headers, or check for reverse-proxy/CDN cache config, or in-app cache middleware.
- **Fix**: set appropriate `Cache-Control` (public/private, max-age, stale-while-revalidate) on cacheable GET endpoints; add ETag for conditional requests; never cache authenticated/mutating responses without care.
- **Verify**: `curl -I` shows correct headers; repeat request returns `304` when conditional, or is served from CDN/proxy cache (check `X-Cache`/`CF-Cache-Status`/`Age` header).

### 7. Compress API payloads
- **Detect**: `curl -I -H "Accept-Encoding: gzip, br"` the endpoint, check `Content-Encoding` response header.
- **Fix**: enable gzip/brotli at server or reverse-proxy level (nginx `gzip on`, Express `compression()`, framework middleware). Also trim payload itself — avoid over-fetching (select only needed fields, use pagination from #5).
- **Verify**: response has `Content-Encoding: gzip` or `br`; payload size measurably smaller than uncompressed.

### 8. Server-side caching
- **Detect**: check for a cache layer (Redis/Memcached/in-memory LRU) in front of rendering or data-fetching logic — distinct from #4 (query-level) and #6 (HTTP-level); this is app-level (rendered fragments, computed views, SSR page cache).
- **Fix**: add cache layer for expensive computed/rendered output with sensible TTL and invalidation on write.
- **Verify**: cache hit ratio metric/log shows hits on repeat requests; latency drops on cached path vs cold path.

### 9. Load balancer
- **Detect**: check infra config (nginx upstream block, cloud LB resource — ALB/ELB/GCLB, Kubernetes Service/Ingress, `docker-compose` with multiple app replicas behind a proxy). Single app instance directly exposed = missing.
- **Fix**: this is infra, not app code — flag if it needs cloud/infra access outside this repo. If in-repo (nginx.conf, k8s manifests, terraform), add LB config across ≥2 app instances with a health check.
- **Verify**: LB health check endpoint returns 200; traffic distributes across instances (check access logs per-instance, or cloud console metrics); killing one instance doesn't drop the app.

### 10. Add CDN
- **Detect**: check static asset URLs/headers for a CDN (`CF-Ray`, `X-Cache`, `Via` headers, or asset domain pointing at Cloudflare/Fastly/CloudFront/Vercel Edge). Check framework config for CDN/asset-prefix setup.
- **Fix**: front static assets (images, JS, CSS, fonts) with a CDN; set `asset_prefix`/`CDN_URL` in build config; ensure long `max-age` + hashed filenames for cache-busting.
- **Verify**: asset response headers show CDN presence; repeat request from a different region is fast/cached (`Age` header increasing, `X-Cache: HIT`).

---

## C. Frontend build / delivery

### 11. Minify JS and CSS
- **Detect**: inspect build output (`dist`/`build`/`.next`) for unminified/readable JS-CSS, or check bundler config (Terser/esbuild/SWC minify flag, `mode: production`).
- **Fix**: ensure production build uses minification (default in most modern bundlers under `production` mode — check it's not accidentally disabled).
- **Verify**: built JS/CSS files are single-line/mangled, not formatted source; bundle size drops vs unminified build.

### 12. Split code into chunks
- **Detect**: check for dynamic `import()`, route-based code splitting, bundler config for chunk splitting (`splitChunks`, Next.js automatic per-route chunks). Run a bundle analyzer to see if it's one monolithic bundle.
- **Fix**: convert large/rarely-used modules (heavy libs, non-critical routes, modals) to dynamic imports; enable framework's route-level splitting.
- **Verify**: bundle analyzer shows multiple chunks instead of one giant bundle; initial page load JS size drops.

### 13. Add Lazy Loading
- **Detect**: check `<img>`/`<iframe>` tags for `loading="lazy"` or an intersection-observer-based lazy component/library; check below-fold images/components load eagerly.
- **Fix**: add `loading="lazy"` to below-fold images/iframes; lazy-load below-fold components/routes.
- **Verify**: network panel shows below-fold images not requested until scrolled into view.

### 14. Defer non-critical scripts
- **Detect**: check `<script>` tags for `defer`/`async`/`type="module"` on non-critical third-party/analytics scripts; check render-blocking scripts in `<head>`.
- **Fix**: add `defer` (needs DOM, order matters) or `async` (independent, order doesn't matter) to non-critical scripts; move to end of `<body>` or use a loader that injects post-load.
- **Verify**: Lighthouse "Eliminate render-blocking resources" no longer flags these scripts; First Contentful Paint improves.

### 15. Compress images
- **Detect**: check image file sizes/formats in assets — large PNG/JPG without WebP/AVIF alternative, no responsive `srcset`, no build-time image optimization pipeline (`next/image`, `sharp`, `imagemin`).
- **Fix**: convert to WebP/AVIF with fallback, compress with `sharp`/`imagemin`/framework image component, serve responsive sizes via `srcset`.
- **Verify**: file size drop (before/after byte count) with no visible quality loss; Lighthouse "Properly size/Serve images in next-gen formats" no longer flags them.

### 16. Debounce input handlers
- **Detect**: grep input `onChange`/`onKeyUp` handlers that trigger network calls, expensive filtering, or re-renders on every keystroke, with no debounce/throttle wrapper.
- **Fix**: wrap handler in debounce (lodash `debounce`, custom hook, or RxJS) with a reasonable delay (200-500ms typical for search-as-you-type).
- **Verify**: network panel/log shows one call after typing stops, not one per keystroke.

### 17. Loading skeletons
- **Detect**: check for skeleton/placeholder components on data-fetching views vs blank screen or spinner-only during load.
- **Fix**: add skeleton components matching final layout shape for async-loaded sections; show during fetch, swap on data arrival.
- **Verify**: visually confirm (screenshot/snapshot mid-load) skeleton renders instead of blank/spinner-only state; no layout shift when real content replaces it (check CLS in Lighthouse).

### 18. Remove unnecessary re-renders
- **Detect**: React/Vue/etc — check for missing `memo`/`useMemo`/`useCallback` on expensive components, inline object/array/function props causing new refs each render, missing key stability in lists. Use React DevTools Profiler or `why-did-you-render` if available to confirm actual re-render count, not just theoretical risk.
- **Fix**: memoize expensive components, stabilize prop references (`useCallback`/`useMemo`), fix unstable `key` props, split state to avoid unrelated re-renders.
- **Verify**: Profiler shows reduced re-render count/duration for the same interaction, before vs after.

### 19. Remove unused dependencies
- **Detect**: run `npx depcheck` (Node) or equivalent (`pip-autoremove --list`, `bundle-audit`, etc). Cross-check flagged deps aren't used dynamically (config files, CLI-only) before removing.
- **Fix**: remove unused packages from manifest and lockfile.
- **Verify**: `depcheck` clean (or remaining flags are confirmed false positives); build still passes; bundle size drops if dep was in the client bundle.

---

## D. Measurement

### 20. Lighthouse audit
- **Detect/Run**: if a live/local URL is reachable, run a Lighthouse audit — via the `lighthouse` CLI (`lighthouse <url> --output=json --quiet --chrome-flags='--headless'`), lighthouse-ci, or whatever browser automation tool your environment provides (Chrome DevTools, Playwright). Pull only the category scores + top 3-5 opportunities, don't dump the full JSON into context.
- **Fix**: address top opportunities — usually maps directly back to items 1-19 above (unminified JS, unoptimized images, render-blocking resources, missing cache headers, layout shift).
- **Verify**: re-run audit after fixes; Performance score improved; specific flagged audits (e.g. "Serve images in next-gen formats") no longer failing.
- Use a performance trace (Chrome DevTools Performance panel, `performance_start_trace`/`performance_stop_trace` if your tools expose it) for deeper diagnosis (LCP/CLS/TBT breakdown) when the score alone isn't enough to locate the cause.

---

## Report format

End with the checklist, evidence inline, nothing marked done without a check behind it:

```
DONE   Cache API responses — Cache-Control: public, max-age=3600 on GET /api/products, verified via curl -I
DONE   Index the Database — idx_orders_user_id added, EXPLAIN shows Index Scan (was Seq Scan, 340ms -> 4ms)
TODO   Load balancer — single app instance, no LB config in repo; needs infra decision (which cloud/provider?)
PARTIAL Remove N+1 database queries — fixed /api/orders, /api/users list still has N+1 on `.reviews` — not yet touched
...
```

Use PARTIAL for in-progress items — don't force everything into a binary done/not-done.
