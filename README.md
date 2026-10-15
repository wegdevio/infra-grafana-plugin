# infra-grafana-plugin

Public mirror for **unsigned/community Grafana plugins** that our Grafana deployments need but that aren't in Grafana's official signed plugin catalog.

This repo exists for one reason: `GF_INSTALL_PLUGINS` (and the `grafana/grafana` Helm chart's `plugins:` values list, which uses the same mechanism) needs a **plain, publicly-fetchable URL** to download a plugin zip from. Our other infra repos (`infra-manifest`, `pkg-manifest`, `studio-manifest`, ...) are all private, so release assets on them 404 for an unauthenticated Grafana pod. This repo is public specifically to host those binaries where nothing else lives — it holds **no source code of our own**, no credentials, no business logic. Just repackaged third-party plugin builds, each one a byte-for-byte repackaging of a specific upstream release/tag.

## How it's organized

One [GitHub Release](../../releases) per plugin version. Tag naming convention:

```
<plugin-id>-<version>
```

e.g. `aliyun-log-service-datasource-2.39.2`. The plugin ID matches the `id` field in that plugin's own `plugin.json` — this is also the string Grafana's `GF_PLUGINS_ALLOW_LOADING_UNSIGNED_PLUGINS` allowlist and the `plugins:` URL's `;<id>` suffix both need, so it's worth getting right. Each release's zip asset is named `<plugin-id>-<version>.zip` and its description records the upstream source commit/tag and a SHA256 checksum.

Multiple plugins, and multiple versions of the same plugin over time, all coexist fine here — each is its own independently tagged release, nothing shared or overwritten.

## Currently hosted

| Plugin ID | Version | Upstream | Release |
|---|---|---|---|
| `aliyun-log-service-datasource` | 2.39.2 | [aliyun/aliyun-log-grafana-datasource-plugin](https://github.com/aliyun/aliyun-log-grafana-datasource-plugin) @ `2.39.2` | [Release](../../releases/tag/aliyun-log-service-datasource-2.39.2) |

## Consuming a plugin from here

In the consuming Grafana Helm values (e.g. `infra-manifest/grafana/values.yaml`):

```yaml
plugins:
  - grafana-clickhouse-datasource   # signed catalog plugin — plain slug
  - https://github.com/wegdevio/infra-grafana-plugin/releases/download/<plugin-id>-<version>/<plugin-id>-<version>.zip;<plugin-id>

grafana.ini:
  plugins:
    allow_loading_unsigned_plugins: <plugin-id>   # scope this to exactly the plugin(s) you're installing — never a wildcard
```

See `docs/packaging-a-plugin.md` for how to add a new plugin release here.
