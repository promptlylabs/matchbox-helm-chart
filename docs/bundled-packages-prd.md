# PRD — Bundled FHIR IG packages in the matchbox Helm chart

## Context

The Promptly `matchbox` Helm chart (`promptlylabs/matchbox-helm-chart`) deploys the upstream `ahdis/matchbox` FHIR server image. Today, implementation guides (IGs) are loaded **at startup** in one of two ways, both via the rendered `application.yaml` ConfigMap:

1. **Registry pull** — `hapi.fhir.implementation-guides.<key>: { name, version }`. Matchbox fetches the package over HTTPS from `packages2.fhir.org`/`simplifier.net` on every pod start. This is how stg + prod-uk are configured today (`infrastructure-live/{stg,prod-uk}/eu-central-1.../matchbox/application/values.yaml`).
2. **Inline URL** — the same block also accepts `url:` per matchbox's [validation config docs](https://ahdis.github.io/matchbox/validation/#configuration-parameters), supporting `classpath:`, `file:`, and `https:` schemes.

We want a third option: ship IG `.tgz` packages **bundled with the chart**, mounted into the pod, and loaded from disk. The matchbox docs describe this as "use `classpath:` to load on startup from a resources folder"; in practice the upstream image doesn't expose an extensible classpath, so the equivalent on-disk mechanism is `file:` against a mounted volume.

## Problem

- Startup IG fetch couples pod boot to the availability of public FHIR package registries. We've seen IG fetch failures cause matchbox to fail readiness.
- Custom / unpublished IGs (e.g. internal-only Promptly IGs, or pre-release versions) cannot be referenced by `name`+`version` because they aren't in any public registry.
- Pinning by `version` does not protect against the registry serving a re-published `.tgz` with the same version string.

## Goals

1. Let chart consumers declare IG `.tgz` files to ship with the chart release and have them available at a known path inside the matchbox pod.
2. Allow those bundled IGs to be loaded by matchbox at startup with no external network calls.
3. Keep the chart contract explicit — the consumer's `application.yaml` still names the IGs that get loaded; the chart only handles the file plumbing.
4. Preserve backwards compatibility — `packages` defaults to `[]`, existing deployments unaffected.

## Non-goals

- Building / publishing IGs (out of scope; consumers hand us a built `.tgz`).
- Rebuilding the matchbox container image to expose a real Java classpath dir for `classpath:` URLs. We'll use `file:` against a mounted volume, which is functionally equivalent for matchbox's IG loader.
- Supporting IG packages larger than ~900 KiB in a single ConfigMap. Larger payloads need a different transport (initContainer + S3, OCI artifact, sidecar) — to be addressed separately if needed.
- Auto-injecting IG entries into `hapi.fhir.implementation-guides`. Consumers continue to declare which IGs to load; the chart only ensures the `.tgz` is on disk.

## Current state (chart)

Repo: `promptlylabs/matchbox-helm-chart`, chart at `charts/matchbox/`.

- `templates/deployment.yaml` already supports passthrough `volumes` / `volumeMounts` (lines ~68-80, ~129-132 in values.yaml). Today consumers could in principle ship a `.tgz` via their own ConfigMap and these passthroughs, but it's undocumented, requires per-consumer template writing, and bypasses chart validation.
- `templates/configmap.yaml` renders `.Values.matchbox` → `application.yaml` mounted at `/config/`. The container reads it via `SPRING_CONFIG_ADDITIONAL_LOCATION`.
- `values.schema.json` validates top-level keys.

## Proposed design

### 1. New values key: `packages`

```yaml
# Bundled FHIR IG packages.
# Each entry references a file shipped in the chart at `charts/matchbox/files/packages/<file>`.
# The chart creates a ConfigMap with binaryData for the listed files, mounts it
# at /packages/ inside the container, and exposes each as file:/packages/<file>.
# Consumers reference these in their matchbox.hapi.fhir.implementation-guides
# block via the `url:` field — see README for the full example.
packages: []
#   - file: hl7.fhir.uv.sdc-4.0.0.tgz
#   - file: fhir.r4.wales.psom-1.0.0-rc3.tgz
```

Schema (`values.schema.json`):

```json
"packages": {
  "type": "array",
  "description": "Bundled FHIR IG packages shipped with the chart from files/packages/.",
  "items": {
    "type": "object",
    "required": ["file"],
    "properties": {
      "file": {
        "type": "string",
        "description": "Filename under charts/matchbox/files/packages/. Used as the key in the ConfigMap and the basename of the mounted path /packages/<file>."
      }
    }
  }
}
```

### 2. New template: `templates/configmap-packages.yaml`

```yaml
{{- if .Values.packages }}
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ include "matchbox.fullname" . }}-packages
  labels:
    {{- include "matchbox.labels" . | nindent 4 }}
binaryData:
{{- range .Values.packages }}
  {{ .file }}: {{ $.Files.Get (printf "files/packages/%s" .file) | b64enc }}
{{- end }}
{{- end }}
```

Notes:
- `Files.Get` returns an empty string for missing files — we should add a `required`/`fail` guard so a typo'd filename surfaces at `helm template` time, not as a silently-empty mount.

### 3. Deployment changes (`templates/deployment.yaml`)

Add the volume + mount, gated on `.Values.packages`:

```yaml
# under volumeMounts:
{{- if .Values.packages }}
- name: packages
  mountPath: /packages
  readOnly: true
{{- end }}

# under volumes:
{{- if .Values.packages }}
- name: packages
  configMap:
    name: {{ include "matchbox.fullname" . }}-packages
{{- end }}
```

The `checksum/config` annotation already covers `configmap.yaml`; we should add a parallel `checksum/packages` annotation so a `.tgz` change triggers a pod roll.

### 4. Consumer-side usage (documented in README, applied later in `infrastructure-live`)

```yaml
packages:
  - file: my-internal-ig-1.2.0.tgz

matchbox:
  hapi:
    fhir:
      implementation-guides:
        my_internal_ig:
          name: my.internal.ig
          version: 1.2.0
          url: file:/packages/my-internal-ig-1.2.0.tgz
```

### 5. README updates

- New "Bundled IG packages" section under Configuration, with the example above and the size caveat (~900 KiB per file, ~1 MiB per ConfigMap total).
- Note that the `url:` form bypasses the package registry, so version-pinning by registry is no longer the trust boundary — the operator now owns the bytes.

### 6. Chart version

Bump `Chart.yaml` `version` to `0.3.0` (minor — additive, no breaking changes). Add `artifacthub.io/changes` entry.

## Constraints / caveats

- **ConfigMap size**: ~1 MiB hard limit per object (etcd). The current IGs in stg (`hl7.fhir.r4.core` ≈ 5 MB, `hl7.fhir.uv.sdc` ≈ 200 KB, `fhir.r4.wales.psom` ≈ 10 KB) are mixed — `r4.core` will not fit. The bundled-package path is for the small/internal IGs; `r4.core` continues to load via registry. Document this clearly.
- **Image immutability**: shipping the `.tgz` in the chart means the package version is pinned by the chart release, not by the IG registry. That's the point, but it means chart consumers must update the chart (or pin to a forked chart) to roll the IG.
- **Helm `.tgz` size**: packaged chart size grows by the sum of bundled files. Acceptable for small IGs.
- **No state in the running pod**: matchbox loads IGs into its DB on startup. Adding/removing a bundled package only takes effect on the next pod boot (handled by the checksum annotation).

## Acceptance criteria

- [ ] `helm template charts/matchbox` with `packages: []` produces byte-identical output to current `main` (backwards compat).
- [ ] `helm template charts/matchbox --set-file ...` with a sample `packages` entry produces: a `*-packages` ConfigMap with the file in `binaryData`, a volume + `/packages` mount on the deployment, and a `checksum/packages` pod annotation.
- [ ] `helm template` with a `packages` entry pointing at a missing file fails with a clear error (not silently empty data).
- [ ] `helm lint charts/matchbox` passes.
- [ ] `values.schema.json` rejects malformed `packages` entries (e.g. missing `file`).
- [ ] README has a "Bundled IG packages" section with a copy-pasteable example and the size caveat.
- [ ] `Chart.yaml` version bumped to `0.3.0`, changes annotation added.

## Testing

1. Unit (template): a `tests/` fixture or a `helm template`-based golden output covering empty `packages`, one package, multiple packages, and a missing-file case.
2. Manual: install into a dev cluster with one small `.tgz` bundled, confirm `/packages/<file>` exists in the pod, matchbox logs show the IG loaded from `file:/packages/<file>`, and no outbound traffic to `packages2.fhir.org` for that IG.

## Rollout

1. Chart PR: implement + tests + README + version bump → merge → CI publishes `matchbox-0.3.0` to `promptlylabs.github.io/matchbox-helm-chart`.
2. `infrastructure-live` follow-up PR (separate ticket): bump `dependencies[0].version` in `stg/.../matchbox/application/Chart.yaml` to `0.3.0`, add `packages:` block + matching `url:` entry under `hapi.fhir.implementation-guides`. Validate in stg, then prod-uk.

## Open questions

1. **Single `packages` ConfigMap vs. one per file** — single is simpler but hits the 1 MiB ceiling sooner. One-per-file is more code but each file gets its own object budget. **Recommendation: single, document the limit, revisit if we hit it.**
2. **Optional `key`/`name`/`version` on each `packages` entry to auto-inject the `implementation-guides` block?** Cleaner UX, more template logic, harder to override. **Recommendation: defer; keep the chart's concern narrow for v0.3.0.**
3. **Do we want `classpath:` for symmetry?** Would require either rebuilding the matchbox image with the file in `BOOT-INF/classes/` or switching to `PropertiesLauncher` + `LOADER_PATH`. **Recommendation: no — `file:` is equivalent for matchbox's loader and avoids a fork.**

## Out of scope (follow-ups)

- Loading large IGs (>1 MiB) — needs initContainer/S3 or OCI artifact transport.
- Signing / integrity verification of bundled `.tgz`s.
- A CI check in the chart repo that the listed `packages[].file` entries actually exist in `files/packages/`.
