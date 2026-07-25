---
title: Provider caching
description: Cache slow provider routes in a local secret store
---

:::caution[Version compatibility]
Provider caching and `secretspec cache clear` are available starting with
SecretSpec 0.17.
:::

A cached alias is a complete provider route: `fallback` contains the
authoritative providers in read order, while `cache.provider` names the local
leaf provider used to accelerate reads.

```toml title="secretspec.toml"
[providers]
azure = { uri = "akv://team-vault", credentials = { client_secret = "keyring" } }
env = "env://"
local = "keyring://secretspec/cache/{project}/{profile}/{key}"

myprovider = {
  fallback = ["azure", "env"],
  cache = { provider = "local", max_age = "8h" }
}

[profiles.development.defaults]
providers = ["myprovider"]
```

## Read behavior

On a read, SecretSpec:

1. returns a fresh cache entry without constructing or contacting `azure` or
   `env`;
2. on a miss, expired entry, malformed entry, or cache error, tries `azure` and
   then `env`;
3. caches the value returned by whichever fallback answers.

Expired values are not returned when every fallback fails. A cache read or
write failure produces a warning but does not prevent the authoritative route
from answering.

Value-free resolution such as `check --json`, `check --explain`, and SDK
`no_values` requests may read an existing cache but never populate or refresh
it.

## Writes

Writes and generated values go to the first fallback (`azure` above), then
refresh the cache. A cache write failure does not hide a successful
authoritative write.

Select a leaf directly to bypass the cached route for one command:

```bash
$ secretspec check --provider azure
```

## Freshness and invalidation

`max_age` requires a unit: `s`, `m`, `h`, `d`, or `w`; compound durations such
as `1h30m` are accepted.

The cache entry is stored at SecretSpec's logical
`{project}/{profile}/{secret}` address even when the authoritative secret uses
a provider-native `ref`. It contains the value, write time, format version, and
a fingerprint of the fallback route and secret reference. Changing the route
or reference therefore invalidates an existing entry.

Clear one entry or every cached entry in the active profile:

```bash
$ secretspec cache clear API_KEY          # SecretSpec 0.17+
$ secretspec cache clear --profile production
```

Cache clearing is supported for keyring, pass, gopass, and dotenv cache
providers.

## Where cached aliases can be used

A cached alias works anywhere a complete route is selected:

- a secret or profile-default `providers` list;
- the user-global default provider;
- `SECRETSPEC_PROVIDER`;
- `--provider`.

It must be the only entry in a `providers` list because its own `fallback`
already defines the complete route. Its fallback entries and cache provider may
be aliases, provider names, or URIs, but must resolve to leaf providers; cached
aliases cannot be nested.

Credentials belong on leaf aliases, such as `azure` above, rather than on
`myprovider`.

## Security

The cache provider stores the secret value, not merely metadata. Choose an
encrypted provider such as keyring, pass, or gopass when cached values must be
encrypted at rest. Dotenv is useful for testing but stores cache entries as
plaintext.

## Next steps

- Review [Provider fallback](/concepts/providers/fallback/) for authoritative
  route and write semantics.
- See [`cache clear`](/reference/cli/#cache-clear-017) in the CLI reference.
- Review the [cached alias fields](/reference/configuration/#secretspec-017-cached-alias-values)
  in the configuration reference.
