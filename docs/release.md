---
type: Runbook
title: krateo-rancher-blueprint — release
description: How a release ships — one semver tag packages the forked chart, pushes it to GHCR and publishes the same archive to the Marketplace's blueprints index — plus the re-fork runbook for tracking a new upstream Rancher version.
resource: oci://ghcr.io/krateo-blueprints/charts/rancher-installer
tags: [release, oci, ghcr, re-fork, rancher]
timestamp: 2026-11-07T00:00:00Z
---

# Release

One plain-semver tag (`X.Y.Z`, **no** `v` prefix) releases the chart. The tag push
triggers `.github/workflows/release-tag.yaml`.

## What a tag ships

`release-tag.yaml` (triggered on tags matching `[0-9]+.[0-9]+.[0-9]+`) publishes the chart
twice, with the same bytes, as the organisation's publishing standard requires
([krateo-blueprints/charts PUBLISHING.md](https://github.com/krateo-blueprints/charts/blob/main/PUBLISHING.md)):

1. Refuses to start unless the tagged commit has a successful `lint` run and the org
   secret `CHARTS_PUBLISH_TOKEN` is set, so nothing reaches GHCR that cannot also reach
   the Marketplace.
2. Stamps `chart/Chart.yaml` and `compositiondefinition.yaml` (`CHART_VERSION` → the tag).
   The stamped chart version also drives the generated Composition apiVersion, so
   `0.2.0` → `composition.krateo.io/v0-2-0`.
3. Packages `chart/` with the stamped `compositiondefinition.yaml` copied into its root
   (the chart is a blueprint: it has a `values.schema.json` and a `CompositionDefinition`),
   and checks the archive itself: it must hold `compositiondefinition.yaml` (installing
   `oci://ghcr.io/krateo-blueprints/charts/rancher-installer` at the tag's version) and
   `values.schema.json`, and `Chart.yaml` must have an https `icon` and `keywords`.
4. Pushes it to `oci://ghcr.io/<owner>/charts/rancher-installer:X.Y.Z`. The OCI namespace
   is derived from the repository owner (`GITHUB_TOKEN` can only write its own namespace,
   so a hardcoded org would 403 the moment the repo moved).
5. Publishes the same archive to the Marketplace's `blueprints` index
   (`https://krateo-blueprints.github.io/charts/blueprints`) through the organisation's
   publisher (`krateo-blueprints/charts/.github/workflows/publish-chart.yaml`, pinned per its rule 9). The
   portal Marketplace installs from that index, not from GHCR; the publisher succeeds only
   once the served index lists the version and `helm pull --repo` installs it anonymously.
6. Attaches the stamped `compositiondefinition.yaml` to the GitHub release for the tag.

```console
$ git tag 0.1.0 && git push origin 0.1.0
# → oci://ghcr.io/krateo-blueprints/charts/rancher-installer:0.1.0
```

Then verify both copies exist:

```console
$ helm show chart oci://ghcr.io/krateo-blueprints/charts/rancher-installer --version 0.1.0 | head -3
$ helm pull rancher-installer --version 0.1.0 --repo https://krateo-blueprints.github.io/charts/blueprints
```

Nothing is bumped by hand: the committed `compositiondefinition.yaml` carries the
`CHART_VERSION` placeholder, and the stamped copy is attached to the release.

## PR-time checks

`.github/workflows/lint.yaml` runs on every PR and on every push to `main` (the release
requires its run on the tagged commit):

- stamps a temporary chart version, then `helm lint ./chart` and `helm template` (default
  values) as a render smoke test;
- the shared docs-standard linter (`lint-docs`), which enforces the Krateo Documentation
  Standard on this bundle.

`.github/workflows/security.yml` runs the org security scan.

## Re-fork runbook — tracking a new upstream Rancher

Because the chart is a **fork** of the upstream Rancher chart, not a wrapper
([overview](./overview.md)), updating Rancher is a re-fork, not a dependency bump:

1. Pull the target upstream Rancher chart version (e.g. from
   `https://releases.rancher.com/server-charts/stable`).
2. Replace `chart/templates/`, `chart/scripts/`, and `chart/values.yaml` with the new
   upstream contents.
3. **Re-apply the NodePort patch** to `chart/templates/service.yaml` — the block guarded
   by the `Krateo blueprint patch` comment (`service.httpNodePort` /
   `service.httpsNodePort`).
4. **Reconcile `chart/values.schema.json`**: keep it complete and crdgen-suitable (no
   `if/then`, no type-unions, scalar defaults) so `core-provider` can still generate the
   `RancherInstaller` CRD; add any new install knob you want exposed on a Composition.
5. Update `chart/Chart.yaml`'s `appVersion` to the new Rancher version, keep `name:
   rancher-installer`, and bump the chart `version` on the next tag.
6. Run `helm lint` / `helm template`, then the kind end-to-end
   ([examples/kind-nodeport](../examples/kind-nodeport/README.md)) before tagging.

## Version pinning downstream

The stamped `compositiondefinition.yaml` attached to each release pins `spec.chart.version`
to that release; nothing tracks a mutable tag. Consumers register the blueprint against
that pinned version.
