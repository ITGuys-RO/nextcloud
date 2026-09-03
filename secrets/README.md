# secrets/

Templates only. **No real secret values in git, ever.**

`.gitignore` permits only `README.md` and `*.example` / `*.template` here.
Anything else (real `*.yaml` secrets) is ignored.

Pattern:
1. `cp secrets/<name>.example secrets/<name>.yaml`
2. fill real values in `secrets/<name>.yaml` (gitignored)
3. `kubectl apply -f secrets/<name>.yaml`

`*.example` template files are committed (cf-api-token, nextcloud-admin,
nextcloud-r2, valkey-auth). The real `*.yaml` files are applied out-of-band and
gitignored — they are never committed.

`nextcloud-db` and `backup-ssh` are gone: the database moved to SQLite (no
MariaDB credentials to hold) and the backup no longer rsyncs over SSH — it
uploads to R2 with the same credentials as the object store.
