# WunderGraph SDK – Offline Dependency & Asset Map
#
# This document lists **all WunderGraph–owned or hosted artifacts** and the
# **external registry / repository references** that must be archived so the
# WunderGraph SDK (TypeScript packages _and_ the accompanying `wunderctl` CLI
# binary) continue to work on an **air‑gapped / offline** network.
#
# It is organised in four sections:
#
# 1. Repositories (source of truth)
# 2. Release artifacts to mirror (npm, GitHub Releases, Docker, …)
# 3. Build / publish instructions for an offline mirror
# 4. Re‑installation instructions for local developers
#
# ---
#
# ## 1. WunderGraph Source Repositories
#
# | Purpose | Upstream repo | Default branch |
# |---------|---------------|----------------|
# | Monorepo containing the SDK source, TypeScript packages, Go server, CLI and docs | `github.com/wundergraph/wundergraph` | `main` |
# | GraphQL parser / executor library used by the server | `github.com/wundergraph/graphql-go-tools` | `main` |
# | High‑performance JSON AST helper library | `github.com/wundergraph/astjson` | `main` |
#
# > NOTE – the Go `go.mod` inside the monorepo has a local `replace` directive for
# > `github.com/wundergraph/graphql-go-tools` → `../graphql-go-tools`. Make sure
# > both repos are checked‑out at compatible commits when vendoring.
#
# ### Optional / related
# • [`github.com/wundergraph/cosmo`](https://github.com/wundergraph/cosmo) – not required by the SDK itself, but referenced in docs.
# • Documentation site sources live under `docs/` and `docs-website/` inside the monorepo.  If you want the docs offline, archive those directories or build the static site.
#
# ---
#
# ## 2. Release Artifacts that have to be mirrored
#
# ### 2.1 npm packages (published under the `@wundergraph` scope)
#
# These are published from `wundergraph/packages/*`.  Latest versions found in the
# current clone:
#
# ```
# @wundergraph/sdk           0.184.2
# @wundergraph/wunderctl     0.180.0   (thin JS wrapper that downloads the Go binary)
# @wundergraph/nextjs        0.15.9
# @wundergraph/react-query   0.9.33
# @wundergraph/react-relay   0.4.33
# @wundergraph/solid-query   0.5.33
# @wundergraph/svelte-query  0.3.33
# @wundergraph/vue-query     0.2.33
# @wundergraph/swr           0.19.3
# @wundergraph/orm           0.3.1
# @wundergraph/golang-client 0.8.19
# @wundergraph/rust-client   0.4.0
# @wundergraph/protobuf      0.118.2
# @wundergraph/metro-config  0.0.1
# create-wundergraph-app     0.6.0      (plain package, not scoped)
# ```
#
# If you intend to serve the SDK only, the **minimum set** is:
#
# ```
# @wundergraph/sdk
# @wundergraph/wunderctl
# ```
#
# …but most apps created with `create-wundergraph-app` will also pull in the
# framework‑specific client helpers above – consider mirroring them as well.
#
# #### Mirroring strategy
# 1. Spin up a private npm registry (e.g. Verdaccio, Artifactory, Nexus).
# 2. From the monorepo root run `pnpm install && pnpm -r build` to produce the
#    compiled `dist/` folders.
# 3. Publish into the internal registry:
#    ```bash
#    export NPM_REGISTRY="http://your.local.registry"
#    pnpm publish -r --access public --no-git-checks --registry "$NPM_REGISTRY"
#    ```
# 4. Freeze the packages: `npm pack` for every published tarball and archive the
#    `.tgz` files together with their `SHASUM`s.
#
# ### 2.2 `wunderctl` pre‑compiled binaries
#
# `@wundergraph/wunderctl` is _not_ the CLI; it is a thin installer that
# downloads the real Go binary from GitHub Releases using URLs of the form:
#
# ```
# https://github.com/wundergraph/wundergraph/releases/download/v${VERSION}/wunderctl_${VERSION}_${OS}_${ARCH}.tar.gz
# ```
#
# Required OS / architectures (as of Jun‑2025):
#
# * Darwin_arm64 (Apple Silicon)
# * Darwin_x86_64 (Intel macOS)
# * Linux_x86_64
# * Linux_arm64
# * Linux_i386 (optional)
# * Windows_x86_64
# * Windows_i386 (optional)
#
# #### Mirroring strategy
#
# 1. Either **download** every needed tarball from the corresponding GitHub
#    release tag **or** build them yourself:
#    ```bash
#    cd wundergraph/cmd/wunderctl
#    goreleaser release --clean --snapshot   # produces all OS/arch tars in dist/
#    ```
# 2. Store the archives in an internal object store (S3‑compatible, Gitea
#    attachments, plain file‑share, …).
# 3. Expose them over HTTPS at
#    `https://your.internal.host/wunderctl_${VERSION}_${OS}_${ARCH}.tar.gz` or keep
#    them in a local directory.
# 4. Tell the JS wrapper to **skip** the download and point to the local binary
#    by setting an env var before running `pnpm install`:
#
#    ```bash
#    export WUNDERCTL_BINARY_PATH=/opt/wg/bin/wunderctl  # path after you untar
#    ```
#
#    Alternatively, fork `@wundergraph/wunderctl` and patch `downloadURL()` in
#    `src/binarypath.ts` to use the internal host instead of `github.com`.
#
# ### 2.3 Docker images (optional)
#
# * `ghcr.io/wundergraph/wundergraph:latest` – server runtime
# * `ghcr.io/wundergraph/wunderctl:<version>` – CLI image
#
# Use `docker save` to create tarballs and store them alongside the other assets.
#
# ---
#
# ## 3. Building and Publishing Everything Offline
#
# ```bash
# # 0. prerequisites
# #    – Go ≥ 1.20, Node ≥ 18, pnpm ≥ 8, git, goreleaser
#
# # 1. clone all repos
# mkdir -p /srv/git && cd /srv/git
# for r in wundergraph wundergraph/graphql-go-tools wundergraph/astjson; do
#   git clone --depth 1 https://github.com/$r.git
# done
#
# # 2. build wunderctl binary (all os/arch)
# cd wundergraph/cmd/wunderctl
# GORELEASER_CURRENT_TAG="v0.180.0" goreleaser release --snapshot --clean
#
# # 3. build & publish node packages
# cd ../../..
# pnpm install
# pnpm -r build
# export NPM_REGISTRY=http://npm.localhost
# pnpm publish -r --access public --no-git-checks --registry $NPM_REGISTRY
#
# # 4. (optional) vendor Go deps
# cd cmd/wunderctl
# go mod vendor
# ```
#
# ---
#
# ## 4. Using the local mirror
#
# ```bash
# npm set registry http://npm.localhost
# pnpm add @wundergraph/sdk
# export WUNDERCTL_BINARY_PATH=/opt/wg/bin/wunderctl
# ```
#
# ---
#
# ## 5. External (non‑WunderGraph) dependencies
#
# • **Go modules** – run `go list -m -json all` inside `cmd/wundergraph-server` and mirror in Athens / goproxy.  
# • **JavaScript packages** – after `pnpm install` run `pnpm store export` and archive the tarball.
#
# ---
#
# ### TL;DR cheat‑sheet
#
# ```bash
# git clone github.com/wundergraph/wundergraph
# pnpm install && pnpm -r build
# pnpm publish -r --registry <local>
# cd cmd/wunderctl && goreleaser release
# ```
#