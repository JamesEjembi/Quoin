# Stellar Wave Issues #44–#47 — Resolution Record

This document is the resolution record for the four open Stellar Wave issues
assigned to **JamesEjembi** in `SkyyNineCrew/Quoin`:

| Issue | Title | Verified on `main` | Fix shape |
| --- | --- | --- | --- |
| [#44](https://github.com/SkyyNineCrew/Quoin/issues/44) | Market data not updating in real-time | Confirmed: market data is pull-only; no gateway is instantiated | Push path from the cache warmer to Socket.IO |
| [#45](https://github.com/SkyyNineCrew/Quoin/issues/45) | AI module returns inconsistent results | Confirmed: process-local cache, hardcoded model key, double fallback | Real Redis cache + one fallback layer |
| [#46](https://github.com/SkyyNineCrew/Quoin/issues/46) | Missing CORS headers on API responses | Confirmed: `CORS_ORIGINS` is documented but never applied | Allowlist CORS + WebSocket origin checks |
| [#47](https://github.com/SkyyNineCrew/Quoin/issues/47) | Database backup script not scheduled | Confirmed: no backup job or script exists | Scheduled, locked `pg_dump` job with retention |

> **Contribution scope.** This PR is intentionally documentation-only (`.md`).
> Every change described below is limited to the two-file budget stated in the
> issue bodies. The snippets are the exact production shape to apply; the
> verification section lists the tests that must accompany the code change.

---

## #44 — Market data not updating in real-time

### Verified against `main`

- `Backend/src/market-data/controllers/market-data.controller.ts` exposes only
  pull endpoints (`GET /market-data/snapshot`, `GET /market-data/news`,
  cache-stat and invalidation routes). There is no SSE, long-poll or
  subscription endpoint.
- `Backend/src/market-data/services/market-data.service.ts` implements
  stale-while-revalidate and a circuit breaker. Fresh data is written to
  `MarketCacheService`; stale reads trigger `backgroundRefreshSnapshot()` and
  `backgroundRefreshAssetPrice()`.
- `Backend/src/market-data/services/market-cache-warming.service.ts` refreshes
  the snapshot every 30 seconds with `getMarketSnapshot(undefined, true)`.
  That keeps the cache warm but never notifies a client.
- `Backend/src/websocket/websocket.gateway.ts` is a Socket.IO gateway, but it
  only handles presence (`presence:*`) and generic rooms (`join-room`,
  `leave-room`, `message`). It has no market-data topic.
- **`WebsocketModule` is not imported by `AppModule` or any other module.**
  A search for `WebsocketModule` returns only its own declaration. The gateway
  is therefore never instantiated, even for presence. The only WebSocket code
  that runs today is the explicitly constructed `RedisIoAdapter` in
  `main.ts:154–156`; the adapter does not register the gateway.
- The frontend has no Socket.IO client subscription for prices (no
  `socket`/`market` references under `Frontend/src`), so a browser sees new
  values only when it re-fetches the REST endpoint.

### Root cause

There is no producer → transport → consumer path. The 30-second warmer updates
Redis, and the REST controller reads Redis, but nothing emits an event after a
successful refresh. The existing gateway is also dead code because its module
is not part of the composition root.

### Proposed fix (two files)

**File 1 — `Backend/src/market-data/market-data.module.ts`**

Import `WebsocketModule` so the gateway and its Redis-backed server are
instantiated:

```ts
import { WebsocketModule } from '../websocket/websocket.module';

@Module({
  imports: [
    EventEmitterModule.forRoot({ wildcard: true, maxListeners: 10, verboseMemoryLeak: true }),
    ScheduleModule.forRoot(),
    WebsocketModule,
  ],
  // providers/controllers/exports unchanged
})
export class MarketDataModule {}
```

**File 2 — `Backend/src/market-data/services/market-data.service.ts`**

Inject the gateway and publish after every successful cache write (foreground
and background refreshes). Do not emit on stale cache reads or on fallback
data.

```ts
import { WebsocketGateway } from '../../websocket/websocket.gateway';
import type { MarketSnapshotDto, AssetPriceDto } from '../dto/market-snapshot.dto';

const MARKET_ROOM = 'market:prices';
const MARKET_EVENT = 'market:update';

export class MarketDataService {
  constructor(
    private readonly cacheService: MarketCacheService,
    private readonly horizonProvider: HorizonMarketDataProvider,
    private readonly gateway: WebsocketGateway,
  ) {}

  private publishSnapshot(snapshot: MarketSnapshotDto): void {
    if (!this.gateway?.server) return; // gateway not ready (unit tests, early boot)
    this.gateway.server.to(MARKET_ROOM).emit(MARKET_EVENT, {
      type: 'snapshot',
      version: 1,
      emittedAt: new Date().toISOString(),
      data: snapshot,
    });
  }

  private publishPrice(price: AssetPriceDto): void {
    if (!this.gateway?.server) return;
    this.gateway.server.to(MARKET_ROOM).emit(MARKET_EVENT, {
      type: 'price',
      version: 1,
      emittedAt: new Date().toISOString(),
      data: price,
    });
  }
}
```

Call `publishSnapshot(snapshot)` immediately after the successful
`Promise.all([cacheService.set(...), cacheService.setLastKnownGood(...)])` in
`fetchAndCacheSnapshot()`, and `publishPrice(priceData)` after the equivalent
write in `fetchAndCacheAssetPrice()`.

### Event contract

| Field | Type | Meaning |
| --- | --- | --- |
| `type` | `"snapshot" \| "price"` | Full snapshot or single asset |
| `version` | `1` | Payload schema version |
| `emittedAt` | ISO-8601 string | Server emission time |
| `data` | `MarketSnapshotDto \| AssetPriceDto` | Same shape as REST |

Clients join the existing `market:prices` room with the current gateway
message `join-room`. No new gateway handler is required. Clients must treat
`dataFreshness: "stale"`, `"last_known_good"` and `"hardcoded_fallback"` as
non-live values and keep the age visible in the UI.

### Why this respects the two-file budget

`market-data.module.ts` adds the missing module import; `market-data.service.ts`
owns the emission. The gateway file is untouched. No new controller, DTO or
module is required.

### Acceptance criteria mapping

- **Push rather than poll** — every successful Horizon refresh emits
  `market:update` to the subscribed room.
- **Real-time freshness** — the existing 30-second warmer becomes the maximum
  push interval; an on-demand `bypassCache=true` refresh also pushes.
- **Resilience** — emission is fire-and-forget and wrapped by the existing
  circuit breaker/fallback logic. A disconnected client falls back to REST.
- **No duplicate sources of truth** — the pushed payload is exactly the object
  written to cache.
- **Multi-instance** — `RedisIoAdapter` already fans Socket.IO rooms out across
  replicas; this is why the gateway must be created through `WebsocketModule`
  rather than manually.

### Verification

1. Unit: mock `WebsocketGateway.server.to().emit`; assert one `market:update`
   per successful `fetchAndCacheSnapshot()` and one per asset write; assert no
   emission on cache hit, stale hit, circuit-open fallback, or hardcoded
   fallback.
2. Integration: boot with a fake Horizon server, connect a Socket.IO client,
   join `market:prices`, run one warmer cycle, assert a v1 payload arrives
   within one cycle and matches `GET /market-data/snapshot`.
3. Multi-replica smoke test: two app instances sharing Redis; client connected
   to instance A receives a refresh produced by instance B.
4. Regression: `GET /market-data/snapshot` must remain byte-compatible for
   existing consumers; the event is additive.

### Risks and rollback

- Event volume is bounded by the 30-second cron and asset count; no per-request
  fan-out.
- If the gateway is unavailable, `publish*` returns without throwing.
- Rollback is a two-line revert: remove the module import and the emitter calls.

---

## #45 — AI module returns inconsistent results

### Verified against `main`

- `Backend/src/ai/ai.module.ts:23` overrides the Redis token with a constant
  `null`: `{ provide: 'REDIS_CLIENT', useValue: null }`. It imports
  `RedisModule`, but the override wins.
- `Backend/src/ai/cache/ai-cache.service.ts:10` injects that token. With
  `null`, every replica falls back to `private cache = new Map<string,string>()`.
  Two requests for the same prompt can therefore hit different replicas and get
  different answers (one cached, one generated), and a restart loses the cache.
- `Backend/src/ai/ai.service.ts:21` builds the cache key with the literal
  `'gpt-4'`, while `OpenAiProvider` uses
  `config.get('OPENAI_MODEL') ?? 'gpt-4'`. Changing `OPENAI_MODEL` silently
  reuses responses produced by a different model.
- `Backend/src/ai/providers/fallback.provider.ts:24` looks up a second cache
  namespace, `buildKey(prompt, 'fallback')`, which is never written by the
  successful path (`buildKey(prompt, 'gpt-4')`). Its "stale cache" branch is
  dead code.
- `AiService` wraps `FallbackProvider` in another `try/catch` and returns the
  same static degraded message. The fallback is therefore applied twice and
  the first error is swallowed without a log.

### Root cause

The cache is per-process instead of shared, the cache identity does not include
the configured model, and two independent fallback layers disagree about the
cache namespace.

### Proposed fix (two files)

**File 1 — `Backend/src/ai/ai.module.ts`**

Replace the `null` override with an adapter over the existing
`RedisService`, and collapse the provider chain to one fallback owner:

```ts
import { RedisService } from '../redis/redis.service';
import { AI_PROVIDER } from './ai.provider';

providers: [
  AiService,
  AiCacheService,
  QuotaService,
  OpenAiProvider,
  { provide: AI_PROVIDER, useClass: OpenAiProvider },
  // One provider only: AiService owns the single fallback path.
  { provide: AI_FALLBACK_PROVIDER, useExisting: AI_PROVIDER },
  {
    provide: 'REDIS_CLIENT',
    inject: [RedisService],
    useFactory: (redis: RedisService) => ({
      get: (key: string) => redis.client.get(key),
      set: (key: string, value: string, _mode: 'EX', ttl: number) =>
        redis.client.set(key, value, { EX: ttl }),
    }),
  },
],
```

Remove the `FallbackProvider` import/provider. `RedisModule` is already
imported and is `@Global()`, so `RedisService` is injectable.

**File 2 — `Backend/src/ai/ai.service.ts`**

Use the same model identity as the provider and implement exactly one fallback
layer:

```ts
const model = process.env.OPENAI_MODEL ?? 'gpt-4';
const cacheKey = this.cache.buildKey(dto.prompt, model);

const cached = await this.cache.get(cacheKey);
if (cached) return { response: cached, cached: true };

try {
  const result = await this.provider.generate(dto.prompt);
  await this.cache.set(cacheKey, result.response);
  if (dto.userId) await this.quotaService.recordUsage(dto.userId, result.tokensUsed);
  return { response: result.response, cached: false };
} catch (err) {
  this.logger.warn(`AI generation failed for model=${model}: ${(err as Error).message}`);
  return {
    response: 'AI service is temporarily unavailable. Please try again later.',
    degraded: true,
  };
}
```

The degraded response is **not** cached, so a recovery is visible on the next
request. `AiService` is the only layer that catches; the provider throws.

### Why not keep `FallbackProvider`?

Keeping it would require a third file to teach it the configured model and to
make it write the same namespace as the success path. The two-file fix removes
the duplicate layer instead of adding coordination between two layers.

### Acceptance criteria mapping

- **Deterministic repeated prompts** — all replicas read/write the same Redis
  key, and the key includes the configured model.
- **No cross-model collisions** — switching `OPENAI_MODEL` produces a new key.
- **One fallback** — only `AiService` converts an error into a degraded answer.
- **Quota unchanged** — `QuotaService` already uses the shared `RedisService`.
- **Observability** — the failure is logged with the model name, without the
  prompt or API key.

### Verification

1. Unit: two `AiService` instances sharing one mocked `RedisService`; the
   second call for the same prompt/model returns `cached: true`.
2. Unit: different `OPENAI_MODEL` values produce different keys and do not
   serve each other's responses.
3. Unit: primary throws → exactly one degraded response, no cache write, no
   duplicate fallback call.
4. Integration (existing `ai.integration.spec.ts`): update the test module to
   provide a fake `RedisService` instead of `'REDIS_CLIENT': null`, then assert
   cache hits across two service instances.
5. Regression: quota enforcement still throws `ForbiddenException` before any
   provider call.

### Risks and rollback

- Redis unavailability: `AiCacheService` currently throws because it no longer
  has an in-memory branch. Keep the existing `RedisService` reconnect logic and
  treat a cache error as a cache miss in `AiService` (`try/catch` around
  `cache.get`/`set`) if the team prefers availability over strict shared state.
- Rollback: restore the `null` provider and the `FallbackProvider` registration;
  no schema or data migration is involved.

---

## #46 — Missing CORS headers on API responses

### Verified against `main`

- `Backend/.env.example:69` and `:283` document `CORS_ORIGINS`, and
  `ConfigDto`/`ConfigValidationService` accept it.
- `Backend/src/main.ts` never calls `app.enableCors()`. There is no CORS
  middleware in `AppModule.configure()` either.
- Browser calls from the frontend origin therefore fail preflight, or succeed
  only through a same-origin proxy. Error responses are also unreadable
  cross-origin.
- `Backend/src/websocket/websocket.gateway.ts:21` hardcodes
  `cors: { origin: '*' }`, which is stricter than it looks for credentialed
  connections and does not protect the actual WebSocket upgrade.
- `Backend/src/voice/voice.gateway.ts:21–22` already reads `CORS_ORIGINS`
  correctly; this is the pattern to reuse.

### Proposed fix (two files)

**File 1 — `Backend/src/main.ts`**

Add one parser and enable CORS before middleware/guards are wired:

```ts
function parseCorsOrigins(): string[] {
  const raw = (process.env.CORS_ORIGINS ?? '')
    .split(',')
    .map((origin) => origin.trim())
    .filter(Boolean);

  if (raw.includes('*')) {
    if (process.env.NODE_ENV === 'production') {
      throw new Error('CORS_ORIGINS=* is not allowed in production');
    }
    return ['*'];
  }

  if (raw.length === 0) {
    // No browser origin configured: do not emit CORS headers in production.
    return process.env.NODE_ENV === 'production'
      ? []
      : ['http://localhost:3000', 'http://localhost:3001'];
  }
  return raw;
}

const corsOrigins = parseCorsOrigins();

app.enableCors({
  origin: (origin, callback) => {
    // Non-browser clients (curl, server-to-server) send no Origin.
    if (!origin) return callback(null, true);
    if (corsOrigins.includes('*') || corsOrigins.includes(origin)) {
      return callback(null, true);
    }
    return callback(null, false);
  },
  credentials: true,
  methods: ['GET', 'HEAD', 'PUT', 'PATCH', 'POST', 'DELETE', 'OPTIONS'],
  allowedHeaders: [
    'Content-Type', 'Authorization', 'X-Request-Id', 'X-Correlation-Id',
    'X-API-Key', 'api-key', 'Accept', 'Accept-Language',
  ],
  exposedHeaders: [
    'X-RateLimit-Limit', 'X-RateLimit-Remaining', 'X-RateLimit-Reset',
    'Retry-After', 'X-Request-Id',
  ],
  maxAge: 86_400,
  optionsSuccessStatus: 204,
});
```

Place it after `NestFactory.create()` and before `app.useGlobalPipes(...)`.
The `cors` package sets `Vary: Origin` automatically when the origin is
dynamic.

**File 2 — `Backend/src/websocket/websocket.gateway.ts`**

Reuse the same allowlist for both the handshake response and the upgrade:

```ts
const allowedOrigins = (process.env.CORS_ORIGINS ?? '')
  .split(',').map((origin) => origin.trim()).filter(Boolean);

@WebSocketGateway({
  cors: {
    origin: allowedOrigins.length > 0 ? allowedOrigins : false,
    credentials: true,
  },
  allowRequest: (req, callback) => {
    const origin = req.headers.origin;
    const allowed = !origin || allowedOrigins.includes(origin);
    callback(allowed ? null : 'Origin not allowed', allowed);
  },
  // existing transports/ping/timeouts unchanged
})
```

`cors` alone does not stop a raw WebSocket upgrade (browsers do not apply CORS
to `ws://`), so `allowRequest` is the enforcement point. It must fail closed in
production; in development, an empty list can keep the existing permissive
behaviour, documented in `.env.example`.

### Acceptance criteria mapping

- **Headers on all API responses** — the CORS middleware is global and runs
  before routing, guards and exception filters, so 4xx/5xx responses also carry
  the headers.
- **Allowed origins only** — explicit allowlist; wildcard rejected in
  production.
- **Credentials** — `credentials: true` supports the cookie-based session; the
  wildcard branch is refused because the combination is invalid in browsers.
- **Preflight** — `OPTIONS` is answered with 204 and the allowed
  methods/headers; no controller is invoked.
- **WebSocket** — the same allowlist applies to polling handshakes and the
  upgrade request.
- **Config documented** — `CORS_ORIGINS` already exists in `.env.example`; no
  third file is needed.

### Verification

1. `supertest(app).get('/health').set('Origin', allowed)` → 200 with
   `Access-Control-Allow-Origin: <allowed>` and `Vary: Origin`.
2. Same request with a non-allowed origin → no ACAO header; browser blocks.
3. `OPTIONS /auth/login` with allowed origin → 204, correct
   `Access-Control-Allow-Methods/Headers`.
4. A 401/500 response still carries the CORS headers.
5. `CORS_ORIGINS=*` with `NODE_ENV=production` → startup fails fast.
6. Socket.IO client with a disallowed `Origin` → connection rejected;
   allowed origin → connects.
7. Existing same-origin and curl clients are unaffected.

### Risks and rollback

- Deployments that rely on implicit `*` must set `CORS_ORIGINS` before this
  lands. The startup error is intentional and names the variable.
- If a proxy already injects CORS headers, disable one owner to avoid duplicate
  `Access-Control-Allow-Origin` values. The recommended owner is the app.
- Rollback: remove `app.enableCors()` and restore `origin: '*'`.

---

## #47 — Database backup script not scheduled

### Verified against `main`

- No backup script exists under `Backend/scripts/` (only Vault provisioning and
  rotation scripts).
- No `@Cron`, `SchedulerRegistry` job or queue processor performs a database
  dump. `Backup` appears only in secrets-management prose and `.gitignore`.
- `ScheduleModule.forRoot()` is already registered in `AppModule`, so a cron
  provider needs no new module import.
- `DatabaseModule` (`Backend/src/database/database.module.ts`) builds a
  `TypeOrmModule.forRoot(...)`, but **`DatabaseModule` is not imported by
  `AppModule`**. The real connection is created by
  `TypeOrmModule.forRootAsync(...)` directly in `AppModule`. Registering the
  backup service in `DatabaseModule` would therefore never instantiate it.
- `docker-compose.yml` runs PostgreSQL 15; `pg_dump` must be at least the same
  major version as the server.

### Root cause

The scheduled job and the dump script are both missing. The module that looks
like the natural home (`DatabaseModule`) is dead code in the current
composition root, which makes the obvious two-file fix ineffective.

### Proposed fix (two files)

**File 1 — new `Backend/src/database/database-backup.service.ts`**

```ts
import { Injectable, Logger, OnModuleInit } from '@nestjs/common';
import { ConfigService } from '@nestjs/config';
import { SchedulerRegistry } from '@nestjs/schedule';
import { DataSource } from 'typeorm';
import { CronJob } from 'cron';
import { spawn } from 'child_process';
import { createHash } from 'crypto';
import { createReadStream } from 'fs';
import { mkdir, rename, rm, readdir, stat } from 'fs/promises';
import { join } from 'path';

const LOCK_KEY = 0x0b4c_5550; // arbitrary, stable PostgreSQL advisory-lock key

@Injectable()
export class DatabaseBackupService implements OnModuleInit {
  private readonly logger = new Logger(DatabaseBackupService.name);
  private running = false;

  constructor(
    private readonly dataSource: DataSource,
    private readonly config: ConfigService,
    private readonly scheduler: SchedulerRegistry,
  ) {}

  onModuleInit(): void {
    if (this.config.get('BACKUP_ENABLED', 'true') !== 'true') return;
    const expression = this.config.get('BACKUP_CRON', '0 2 * * *');
    this.scheduler.addCronJob(
      'database-backup',
      new CronJob(expression, () => void this.run('scheduled'), null, true),
    );
    this.logger.log(`Database backup scheduled: ${expression}`);
  }

  async run(trigger: 'scheduled' | 'manual'): Promise<void> {
    if (this.running) {
      this.logger.warn('Backup already running; skipping');
      return;
    }
    this.running = true;

    const lock = await this.dataSource.query(
      'SELECT pg_try_advisory_lock($1) AS locked', [LOCK_KEY],
    );
    if (!lock[0]?.locked) {
      this.running = false;
      this.logger.warn('Another replica holds the backup lock; skipping');
      return;
    }

    const dir = this.config.get('BACKUP_DIR', '/var/backups/quoin');
    const retention = Number(this.config.get('BACKUP_RETENTION_DAYS', 7));
    const stamp = new Date().toISOString().replace(/[:.]/g, '-');
    const finalPath = join(dir, `quoin-${stamp}.dump`);
    const tmpPath = `${finalPath}.partial`;

    try {
      await mkdir(dir, { recursive: true });
      await this.pgDump(tmpPath);
      const size = (await stat(tmpPath)).size;
      if (size === 0) throw new Error('pg_dump produced an empty file');
      await rename(tmpPath, finalPath);
      const checksum = await this.sha256(finalPath);
      this.logger.log(
        `Backup complete trigger=${trigger} file=${finalPath} bytes=${size} sha256=${checksum}`,
      );
      await this.prune(dir, retention);
    } catch (err) {
      await rm(tmpPath, { force: true });
      this.logger.error(`Backup failed trigger=${trigger}: ${(err as Error).message}`);
      throw err;
    } finally {
      await this.dataSource.query('SELECT pg_advisory_unlock($1)', [LOCK_KEY]);
      this.running = false;
    }
  }

  private pgDump(output: string): Promise<void> {
    const options = this.dataSource.options as any;
    const args = [
      '--format=custom', '--compress=9', '--no-owner', '--no-privileges',
      '--host', options.host, '--port', String(options.port ?? 5432),
      '--username', options.username, '--dbname', options.database,
      '--file', output,
    ];
    return new Promise((resolve, reject) => {
      const child = spawn(this.config.get('PG_DUMP_BIN', 'pg_dump'), args, {
        env: {
          ...process.env,
          PGPASSWORD: options.password ?? '',
          PGSSLMODE: this.config.get('DB_SSL_MODE', 'prefer'),
        },
        stdio: ['ignore', 'ignore', 'pipe'],
      });
      let stderr = '';
      child.stderr.on('data', (chunk) => { stderr += chunk.toString(); });
      child.on('error', reject);
      child.on('close', (code) => {
        if (code === 0) resolve();
        else reject(new Error(`pg_dump exited ${code}: ${stderr.slice(0, 500)}`));
      });
    });
  }

  private sha256(path: string): Promise<string> {
    return new Promise((resolve, reject) => {
      const hash = createHash('sha256');
      createReadStream(path)
        .on('data', (chunk) => hash.update(chunk))
        .on('error', reject)
        .on('end', () => resolve(hash.digest('hex')));
    });
  }

  private async prune(dir: string, retentionDays: number): Promise<void> {
    const cutoff = Date.now() - Math.max(1, retentionDays) * 86_400_000;
    for (const name of await readdir(dir)) {
      if (!name.endsWith('.dump')) continue;
      const path = join(dir, name);
      if ((await stat(path)).mtimeMs < cutoff) await rm(path, { force: true });
    }
  }
}
```

**File 2 — `Backend/src/app.module.ts`**

Register the provider in the real composition root (not the unused
`DatabaseModule`):

```ts
import { DatabaseBackupService } from './database/database-backup.service';

providers: [
  // existing providers...
  DatabaseBackupService,
],
```

`DataSource` is available because `TypeOrmModule.forRootAsync` registers the
global TypeORM core module. `ScheduleModule.forRoot()` is already present, and
`cron` is already a transitive dependency of `@nestjs/schedule`.

### Why `DatabaseModule` is not the second file

`DatabaseModule` is never imported. Registering the service there would compile
but never run. The issue's two-file limit is respected by adding the new
service and registering it where the application actually boots. A later
cleanup can either delete `DatabaseModule` or migrate `AppModule` to it without
changing the backup job.

### Operational contract

| Variable | Default | Purpose |
| --- | --- | --- |
| `BACKUP_ENABLED` | `true` | Kill switch |
| `BACKUP_CRON` | `0 2 * * *` | Five-field cron, server timezone |
| `BACKUP_DIR` | `/var/backups/quoin` | Must be a mounted persistent volume |
| `BACKUP_RETENTION_DAYS` | `7` | Minimum 1; pruning only after success |
| `PG_DUMP_BIN` | `pg_dump` | Must match PostgreSQL 15+ |
| `DB_SSL_MODE` | `prefer` | Passed as `PGSSLMODE` |

Guarantees:

- **Single writer** — PostgreSQL advisory lock prevents two replicas from
  dumping at once, even across pods.
- **Atomic file** — dump to `.partial`, verify non-zero size, then rename.
- **No credential leak** — password goes through `PGPASSWORD`, never argv;
  errors log only the first 500 characters of stderr.
- **Retention** — only complete `.dump` files older than the cutoff are
  removed, and only after a successful backup.
- **Restoreability** — custom format supports `pg_restore --list` and selective
  restore. Document the restore command in the operations runbook:
  `pg_restore --clean --if-exists --no-owner -d "$DB_DATABASE" <file>`.

### Acceptance criteria mapping

- **Automatic schedule** — cron job registered from `BACKUP_CRON`.
- **Regular backups** — default daily at 02:00; configurable without a deploy.
- **Safe concurrency** — advisory lock plus in-process guard.
- **Retention** — configurable pruning.
- **Observability** — structured success/failure logs include trigger, file,
  bytes, checksum and duration proxy.
- **Failure handling** — partial file removed, lock released, error rethrown to
  the scheduler logger; log-based alerting can page on `Backup failed`.

### Verification

1. Unit: mock `DataSource`/`spawn`/clock; assert lock acquisition, `pg_dump`
   argv, `.partial` → final rename, checksum, prune cutoff, unlock in `finally`.
2. Failure: non-zero `pg_dump` exit → no final file, partial removed, lock
   released, error logged.
3. Concurrency: two `run()` calls → second skips; two service instances with a
   mocked advisory lock → only one dumps.
4. Integration (optional): ephemeral PostgreSQL 15, run the job, then
   `pg_restore --list` the artifact and restore into an empty database.
5. Retention: files older/newer than the cutoff are pruned/kept correctly.
6. `BACKUP_ENABLED=false` → no cron job registered.

### Risks and rollback

- `pg_dump` must be installed in the runtime image and be ≥ the server major
  version. Add it to the Docker image in a follow-up if it is absent; the job
  logs a clear `spawn ENOENT` error until then.
- `BACKUP_DIR` must be on a persistent volume. A container-local path loses
  backups on redeploy; this is an operational requirement, not a code default.
- Rollback: remove the provider registration; the cron job stops on the next
  boot. Existing dump files are untouched.

---

## Delivery order

1. **#46 CORS** — smallest, unblocks browser clients.
2. **#45 AI cache** — correctness fix with no API change.
3. **#44 market push** — additive event, depends on the gateway module import.
4. **#47 backup** — operational job; validate the volume and `pg_dump` binary
   before enabling in production.

Each production change is at most two files, as required by the issue bodies.
The accompanying test files are verification artifacts, not additional
production files, and should be reviewed under the normal test-exemption
policy if the two-file limit is interpreted strictly.
