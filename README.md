# infra-role-attic

Deploys [Attic](https://docs.attic.rs/), a multi-tenant Nix binary cache, as a
Docker Compose Setup under `/docker/attic`. Cache contents are stored in DO
Spaces. PostgreSQL is provided by `infra-role-postgres-ha` and reached over a
shared Docker network.

## What it does

- Templates `atticd.toml` into a bind-mounted config directory under
  `/docker/attic/config` and runs `atticd`.
- Creates the configured caches (`ci` and `public`, both public) via the atticd
  HTTP API.
- Mints a dedicated push token per repo per cache and pushes them to Vault.
- Registers a Consul service and healthcheck.

nginx, PostgreSQL, its metrics exporter, and DB backups are configured outside
this role (in `infra-ci`, via `infra-role-nginx` and `infra-role-postgres-ha`).

## Tokens

Consumer tokens are stored in Vault (KV v2) at `tokens/<org>/<repo>/<cache>`. To
read a token for distribution:

```sh
vault kv get -field=jwt secret/ci/attic/tokens/logos-co/logos-basecamp/ci
```

Set `attic_token_rotate=true` to re-mint and overwrite the tokens in Vault.

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
