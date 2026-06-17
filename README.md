# infra-role-attic

Deploys [Attic](https://docs.attic.rs/), a multi-tenant Nix binary cache, as a
Docker Compose Setup (`atticd` + PostgreSQL) behind an nginx reverse proxy.
Cache contents are stored in DO Spaces;

## What it does

- Templates `atticd.toml` and a `docker-compose.yml` under `/docker/attic`.
- Runs `atticd` and `postgres`.
- Creates the configured caches (`ci` private, `public` public)
- Mints the per-cache consumer tokens and persists them to `0400` files under
  `/docker/attic/tokens/`.
- Configures nginx as a TLS-terminating reverse proxy.
- Registers a Consul service and healthcheck.
- Schedules a daily `pg_dump` (picked up by `infra-role-restic-backups` in infra-ci).


## Tokens

To read a token for distribution:

```sh
sudo cat /docker/attic/tokens/ci.jwt
```

The `public` cache's signing public key (for `flake.nix` substituters) can be
fetched anonymously:

```sh
curl -s https://cache.nix.logos.co/_api/v1/cache-config/public | jq -r .public_key
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
