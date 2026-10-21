# cloud/ rules

The optional shared deployment on Cloudflare: Worker and `RealtimeHub` Durable Object in `src/index.mjs`, D1 migrations in `migrations/`, R2 bucket for attachments, bindings in the root `wrangler.jsonc`. Risk level: high when a change touches migrations or remote resources. Root [AGENTS.md](../AGENTS.md) still applies.

## Remote operations need explicit approval

`npm run cloud:migrate` and `npm run cloud:deploy` act on the live database and Worker. Do not run them, or any `wrangler ... --remote` command, unless the user asks for that exact step. The local and dry-run commands below are always fine.

## Migrations

- Add a new file with the next unused number: `NNNN_snake_case.sql`. Check the folder first: numbers `0009` and `0010` are each used twice already, so do not assume the last file's number plus one is free without looking.
- Never edit a migration after it may have been applied anywhere. Wrangler tracks applied migrations by file name, so an edit silently never reaches existing databases. Past commits did edit `0001_initial.sql`; do not follow that pattern, write a new migration instead.
- A schema change usually has a matching local change in `server/database.mjs` (an additive `ALTER TABLE`). See "Adding a task field" in [ARCHITECTURE.md](../ARCHITECTURE.md).

## Parity with the local server

The Worker reimplements the `/api` surface and imports `shared/`. A change in `shared/task-input.mjs`, `shared/api-fields.mjs`, `shared/task-records.mjs`, or `shared/task-relations.mjs` affects the cloud too. The `parseCloud*` variants keep their own validation order and error codes on purpose.

## Secrets

`TASKBOARD_SHARED_SECRET` is set with `wrangler secret put`, or `.dev.vars` for local runs. Never write secrets into `wrangler.jsonc` or commit `.dev.vars`.

## Verify

```bash
npm run test:cloud
```

Also useful: `npm run cloud:migrate:local`, `npm run dev:cloud`, `npm run cloud:deploy:dry-run`, and `node --test test/cloud-migration.test.mjs test/cloud-companion.test.mjs`.
