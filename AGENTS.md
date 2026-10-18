# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

Reference example TM Forum ODA Components, used to test the [ODA Canvas](https://github.com/tmforum-oda/oda-canvas). Each component is a Helm chart (`charts/<Name>`) that deploys an ODA Component custom resource plus the microservices behind it (source in `source/<Name>`). Charts are published as a Helm repo (`helm repo add oda-components https://tmforum-oda.github.io/reference-example-components`). The component list (TMFC001, 002, 005, 006, 007, 008, 028) is in `README.md`.

## Layout and how the pieces connect

- `charts/<Name>/` — `Chart.yaml`, `values.yaml`, `templates/` (the ODA `Component` CR, one Deployment+Service per API, MongoDB, init Jobs, PVC). Component-specific optional features (dependent APIs, MCP server) are toggled via `component.*` values.
- `source/<Name>/` — one directory per microservice (`*Microservice`) with a matching `*-dockerfile` at the component root. Dockerfiles use the component dir as build context. Most services are Node.js (Swagger-generated Express apps under `<x>Microservice/implementation/`, `npm start`); the MCP servers (`MCPServerMicroservice`) are Python (`uv`, `pyproject.toml`, pytest-style tests alongside).
- Shared pieces are reused across components: e.g. `partyroleapi`, `permissionspecapi` images are built from `source/ProductCatalog` but used by every chart; each component also has `openMetricsMicroservice`, `roleInitializationMicroservice` and an initialization job.
- `dockerbuild-config.yaml` — the single registry of images (name, version, build context, dockerfile, watched `paths`). Images are published to Docker Hub org `tmforumodacanvas`. Chart `values.yaml` image tags must match the versions here.
- `scripts/images.py` — drives builds (`list`, `build <name> [--push]`, `republish`, `select`). Needs Docker buildx and PyYAML. `select` is what CI uses.
- `skills/create-oda-component/` — a skill (with templates/references) for scaffolding a new component's source and chart; `.github/skills` is a symlink to `skills/`. `skills/skill-creator/` is vendored third-party material with its own license.

## Workflows / conventions that span files

- **Releasing a change to an image**: image tags are immutable. Changing anything under an image's watched `paths` requires bumping `version` in `dockerbuild-config.yaml` AND the matching tag in the chart `values.yaml`; the PR build fails if the tag already exists on Docker Hub. Merge to `master` builds multi-arch (amd64/arm64) and pushes.
- **Releasing a chart change**: bump `version` in `Chart.yaml` and add a `# version: x.y.z - ...` history comment line (existing convention). `release.yml` runs chart-releaser on every push to `master`.
- Adding a new component: follow `skills/create-oda-component/SKILL.md`, add its images to `dockerbuild-config.yaml`, and add a row to the README table.

## Commands

```bash
helm lint charts/<Name>
helm template test charts/<Name>                      # render; add --set component.MCPServer.enabled=true etc.
helm install <release> charts/<Name> -n components    # needs an ODA Canvas cluster

python scripts/images.py list
python scripts/images.py build <image-name>           # local build; add --push / --platforms

# Node microservice standalone (e.g. Product Catalog API on localhost:8080)
cd source/ProductCatalog/productCatalogMicroservice/implementation && npm start
```

## Testing

There is no unit-test suite for the Node services. Conformance/functional tests are Postman collections in `source/<Name>/tests/` (see `source/ProductCatalog/tests/README.md`), run against `{{Hostname}}` — `localhost:8080` for standalone, or the component's API endpoint when deployed in a Canvas. The Python MCP server has its own tests in `source/ProductCatalog/MCPServerMicroservice/` (`test_*.py`; run with `uv run pytest <file>` — needs the API running).
