# Packaging a new plugin release

Steps to add a new unsigned Grafana plugin (or a new version of one already here) as a release in this repo. Written from how `aliyun-log-service-datasource-2.39.2` was packaged — follow the same shape for the next one.

## 1. Get the built plugin artifact

You need the **built** plugin — frontend `module.js` + `plugin.json`, and (if it's a backend plugin) the compiled backend binaries for each platform (`gpx_<name>_linux_amd64`, etc.). Not the source tree.

Many plugin repos publish this as a GitHub Release asset — if so, just download that and skip to step 2. If not (e.g. `aliyun-log-grafana-datasource-plugin`'s own Releases page has never actually had assets attached, despite its README describing that process), check whether the maintainer commits the built output directly into the repo at each release tag, usually in a `dist/` directory:

```bash
git clone --depth 1 --branch <upstream-tag> <upstream-repo-url> /tmp/plugin-src
ls /tmp/plugin-src/dist/   # should contain plugin.json, module.js, and (for backend plugins) gpx_* binaries
```

**Verify before trusting it:** check `dist/plugin.json`'s `id` and `info.version` match what you expect — the upstream tag should equal the committed version, don't assume it silently.

## 2. Repackage with the correct zip structure

Grafana's plugin loader requires the zip's **top-level directory to be the plugin ID** (from `plugin.json`'s `id` field) — not `dist/`, not the repo name.

```bash
mkdir -p plugin-package/<plugin-id>
cp -r /tmp/plugin-src/dist/* plugin-package/<plugin-id>/
cd plugin-package
zip -r -X ../<plugin-id>-<version>.zip <plugin-id>/
cd ..
shasum -a 256 <plugin-id>-<version>.zip
```

## 3. Validate locally before publishing

Don't skip this — plugin compatibility with the Grafana version we actually run isn't guaranteed just because the plugin's README claims a minimum version. Spin up a throwaway container on the **exact Grafana version/tag our deployments use** (check `image.tag` in the consuming Helm values, e.g. `infra-manifest/grafana/values.yaml`):

```bash
docker run -d --name plugin-test -p 13213:3000 \
  -v "$(pwd)/plugin-package/<plugin-id>:/var/lib/grafana/plugins/<plugin-id>" \
  -e "GF_PLUGINS_ALLOW_LOADING_UNSIGNED_PLUGINS=<plugin-id>" \
  grafana/grafana:<our-version>
```

Check:
- `docker logs plugin-test | grep -i "<plugin-id>\|error\|panic"` — look for `Plugin registered pluginId=<plugin-id>`, no crash/panic from the backend binary
- `curl http://admin:admin@localhost:13213/api/plugins/<plugin-id>/settings` — plugin metadata loads, version matches
- If it's a datasource plugin: create a test datasource via `POST /api/datasources` with dummy credentials, then hit `POST /api/datasources/uid/<uid>/health` — expect a clean structured error (e.g. a DNS/auth failure), not a 500 or empty response. That confirms the backend process is actually alive and handling requests, not just that the frontend module loaded.

Clean up (`docker rm -f plugin-test`) once satisfied.

## 4. Publish the release

```bash
gh release create <plugin-id>-<version> \
  <plugin-id>-<version>.zip \
  --repo wegdevio/infra-grafana-plugin \
  --title "<plugin-id> plugin v<version>" \
  --notes "Repackaged from upstream tag <version> (<upstream-repo-url>).
Source commit: <upstream-tag-or-commit>
plugin.json version verified to match the tag.
SHA256: <sha256-from-step-2>
Locally validated against grafana/grafana:<our-version>: plugin registers cleanly as unsigned, backend starts without error, datasource config + health-check both respond correctly."
```

Update the table in `README.md` with the new row.

## 5. Wire it up in the consuming Grafana values

See `README.md`'s "Consuming a plugin from here" section.
