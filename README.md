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
- Wires those tokens into each repo's GitHub Actions config.
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

## GitHub repo configuration

The role also configures every repo in `attic_repos`. so no click-ops is needed
to onboard one:

| What                                   | Where                | Value                       |
|----------------------------------------|----------------------|-----------------------------|
| `ATTIC_ENDPOINT` variable              | repo                 | `attic_github_endpoint`     |
| `ATTIC_TOKEN_CI` secret                | repo                 | Vault `<org>/<repo>/ci`     |
| `public-cache` environment             | repo                 | master branch only          |
| `ATTIC_TOKEN_PUBLIC` secret            | `public-cache` env   | Vault `<org>/<repo>/public` |

The environment's deployment branch policy is set to each repo's default
branch, since not all of them release from `master`.

## CI usage in GH Actions

```yaml
- name: Setup Attic cache
  uses: ryanccn/attic-action@v0
  with:
    endpoint: ${{ vars.ATTIC_ENDPOINT }}
    cache: ci
    token: ${{ secrets.ATTIC_TOKEN_CI }}
    skip-push: ${{ github.event_name == 'pull_request' }}
```

Pushing to the `public` cache requires the environment to be set like this

```yaml
jobs:
  push-public:
    environment: public-cache
    steps:
      - uses: ryanccn/attic-action@v0
        with:
          endpoint: ${{ vars.ATTIC_ENDPOINT }}
          cache: public
          token: ${{ secrets.ATTIC_TOKEN_PUBLIC }}
```

The `public` cache may additionally be added to `flake.nix`
`nixConfig.extra-substituters` with its public key (`attic cache info`).
