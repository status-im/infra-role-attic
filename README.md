# infra-role-attic

Deploys [Attic](https://docs.attic.rs/), a multi-tenant Nix binary cache, as a
Docker Compose Setup (`atticd` + PostgreSQL) behind an nginx reverse proxy.
Cache contents are stored in DO Spaces;

## What it does

- Templates `atticd.toml` and a `docker-compose.yml` under `/docker/attic`.
- Runs `atticd` and `postgres`.
- Configures nginx as a TLS-terminating reverse proxy.
- Registers a Consul service and healthcheck.
- Schedules a daily `pg_dump` (picked up by `infra-role-restic-backups` in infra-ci).

## Tokens

After the server is up, mint tokens and create the two caches. `atticadm`
runs inside the container and reads the same config/secret.

```sh
# Admin token with full permissions:
docker exec attic-server atticadm make-token \
  --sub admin --validity '1y' \
  --pull '*' --push '*' --create-cache '*' \
  --configure-cache '*' --configure-cache-retention '*'

# Log the attic client in against the deployment:
attic login logos https://cache.nix.logos.co <admin-token>

# Public cache:  safe to add to flake.nix.
attic cache create logos:public
attic cache configure logos:public --public

# CI cache: private, push from CI only.
attic cache create logos:ci
```

Then mint the per-use tokens stored as GitHub Actions secrets:

```sh
# CI: push + pull on the `ci` cache only.
docker exec attic-server atticadm make-token \
  --sub ci --validity '1y' --push 'ci' --pull 'ci'

# Public: push on `public` (used by master-branch builds only).
docker exec attic-server atticadm make-token \
  --sub public --validity '1y' --push 'public'
```

## CI usage in GH Actions 

```yaml
- name: Setup Attic cache
  uses: ryanccn/attic-action@v0
  with:
    endpoint: https://cache.nix.logos.co
    cache: ci
    token: ${{ secrets.ATTIC_TOKEN }}
    skip-push: ${{ github.event_name == 'pull_request' }}
```

The `public` cache may additionally be added to `flake.nix`
`nixConfig.extra-substituters` with its public key (`attic cache info`).
