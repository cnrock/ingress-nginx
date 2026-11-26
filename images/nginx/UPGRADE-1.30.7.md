# nginx 1.27.1 → 1.30.7 upgrade (NGINX2Root fixes)

Branch: `nginx-1.30.7-nginx2root-fix` (based on upstream `main`, controller v1.15.1)

## Why

On 2026-11-18, Xint (Theori) disclosed "NGINX2Root" and F5 shipped an
out-of-band security release. Five nginx RCEs, all CVSS v4 9.2:

| CVE | Flaw | Needs |
| --- | --- | --- |
| CVE-2026-123007 | HTTP/2 flow-control integer underflow → worker RCE | HTTP/2 enabled |
| CVE-2026-123633 | HTTP/2 reverse-proxy use-after-free | `proxy_pass` + HTTP/2 |
| CVE-2026-123634 | Resolver use-after-free | `resolver` + variable `proxy_pass` |
| CVE-2026-121810 | HTTP/3 (QUIC) use-after-free | HTTP/3 + proxying |
| CVE-2026-113425 | Mail proxy use-after-free | `proxy on` + `auth_http` |

ingress-nginx is a reverse proxy with HTTP/2 on by default, so it is directly
exposed to CVE-2026-123007 and CVE-2026-123633. Affected range is nginx
1.9.5–1.31.8, which includes the 1.27.1 this repo pins. ingress-nginx is
retired upstream and will not publish a fix.

Fixed in nginx **1.30.7** (stable) / 1.31.9 (mainline). This branch upgrades
to 1.30.7 (stable branch chosen for the smallest behaviour delta against the
module set this repo builds). The same F5 release also fixes five further
CVEs (incl. CVE-2026-118899, TLS 1.3), which are included automatically.

Note: the "root" in NGINX2Root additionally chains an *unpatched Linux kernel
SCTP use-after-free*. That part is a host/kernel issue, not an nginx one.
Hosts that do not use SCTP can block the chain with
`install sctp /bin/false` in a file under `/etc/modprobe.d/`.
Interim mitigation if you cannot roll images yet: `http2 off;` blocks only
CVE-2026-123007 and CVE-2026-123633.

## What changed

1. `images/nginx/rootfs/build.sh`
   - `NGINX_VERSION` 1.27.1 → **1.30.7**
   - tarball SHA-256 updated to the nginx.org 1.30.7 tarball
     (`2565b5c837050e958f70fb0f73c48e2bca4542ad349fd356f46fb0c72610bc6e`)
2. `images/nginx/rootfs/patches/` — the 35 local core patches were rebased:
   - 33 patches regenerated as clean, fuzz-free diffs against pristine
     1.30.7 (file names updated from `1.27.1` to `1.30.7`). Round-trip check:
     applying the new set to a pristine 1.30.7 tree reproduces the patched
     tree exactly.
   - `19_..._log_escape_non_ascii` was hand-rebased: in 1.30.7
     `ngx_http_log_variable()` gained an `end` parameter and length checks,
     so both escape call sites now pass the location conf. The
     `log_escape_non_ascii` directive (default `on`) is preserved.
   - `28_..._CVE-2025-23419` was **dropped**: the fix is already upstream in
     1.30.7 (see `handshake_rejected` handling in
     `src/http/ngx_http_request.c` and `src/stream/ngx_stream_ssl_module.c`).
3. `images/nginx/TAG` bumped to `v2.3.0` for the rebuilt base image.
4. `build.sh`'s `get_src` now actually verifies SHA-256 checksums (upstream
   had the check commented out): any source whose recorded hash is a real
   64-hex value is verified after download and the build aborts on mismatch;
   sources still carrying the `abc123` placeholder are skipped. If a build
   ever aborts on a GitHub archive whose hash changed because GitHub
   recompressed it, update that source's hash in `build.sh` after verifying
   the new tarball.

No third-party module versions were changed. If the build surfaces a module
that no longer compiles against 1.30.7 (lua-nginx-module is the usual
suspect), bump only that module's `*_VERSION` in `build.sh` and note it here.

## Build

```sh
# 1. nginx base image (single-arch, local)
cd images/nginx
podman build rootfs -t ingress-nginx/nginx:v2.3.0        # or: docker build

# 2. controller image, using the local base image
cd ../..
TAG=v1.15.1-nginx1.30.7 NGINX_BASE=ingress-nginx/nginx:v2.3.0 make image
```

Verify the result before deploying:

```sh
podman run --rm <controller-image> /usr/local/nginx/sbin/nginx -v   # expect 1.30.7
# or check the controller binary's built-in version output
```

## Verification done so far

- Patch round-trip: the new patch set applied to pristine 1.30.7 reproduces
  the patched tree exactly (0 diff lines).
- Native smoke build (2026-11-19, this repo's patched tree, built-in modules
  only: SSL, HTTP/2, stream, threads, PCRE-JIT): `configure` + `make` clean,
  binary reports `nginx/1.30.7`.
- Runtime check: config using `log_escape_non_ascii off;`, an HTTP/2 server
  and a stream proxy passes `nginx -t` and serves live requests.
- Full image build (2026-11-19): all third-party modules compiled unchanged
  against 1.30.7 — no module version bumps were needed (including
  lua-nginx-module v0.10.28, ModSecurity, geoip2, njs, OpenTelemetry).
  Built images, verified (`nginx/1.30.7`, controller reports
  `v1.15.1-nginx1.30.7`) and pushed to Docker Hub:
  `cnrock/ingress-nginx-base:1.30.7` (205 MB,
  sha256:48eaf9ffc53e16272cc54310b3172501ab254abe08d6c7962f4c6b9b488914e5)
  and `cnrock/ingress-nginx-controller:1.30.7` (341 MB,
  sha256:83052326ff41fb6ec226f530c6c88e1c6de7e10cfc5eb05149e1561242bd3087).

## Updating next time

1. Bump `NGINX_VERSION` in `images/nginx/rootfs/build.sh` and update the
   tarball SHA-256 next to it (hashes are published on nginx.org).
2. Re-run the patch check: download the new tarball, apply every file in
   `images/nginx/rootfs/patches/` in `ls` order with `patch -p1`. Any patch
   that now fails is either upstreamed (drop it, after verifying the fix is
   in the new source) or needs a hand rebase like patch 19 above.
3. Bump `images/nginx/TAG`, rebuild base + controller, re-verify `nginx -v`.

## Helm chart (fork, 2026-11-26)

The fork's chart (`charts/ingress-nginx`, version `4.15.1-cnrock.1`,
appVersion `1.15.1`) defaults the controller image to
`cnrock/ingress-nginx-controller:1.30.7` (`controller.image.repository` /
`tag` in `values.yaml`, digest intentionally empty — the fork image is
referenced by tag only). Upgrading the chart also installs the v1.15.1
RBAC, which includes the `discovery.k8s.io/endpointslices` get/list/watch
rule the old (v1.12-era) manifests lack — that rule is required because
v1.15.1 resolves upstreams exclusively via EndpointSlices.

Upgrade from the fork chart directory, keeping the release's existing
values, and make sure the values carry the node pinning for the dedicated
ingress node — a plain `helm upgrade` rewrites the Deployment, so a
nodeSelector applied earlier with `kubectl patch` would be lost:

```bash
helm list -n ingress            # confirm release name + chart version
# Audit only — DO NOT feed this file back with -f (see note below).
helm get values <release> -n ingress -a > old-values.yaml   # image overrides? nodeSelector?
# Carry forward ONLY non-image custom values from the audit into a small
# hand-built file (e.g. custom-values.yaml); delete any controller.image
# block — the new chart default is the 1.30.7 image.
helm upgrade <release> ./charts/ingress-nginx -n ingress \
  -f custom-values.yaml \
  --set controller.image.repository=cnrock/ingress-nginx-controller \
  --set controller.image.tag=1.30.7 \
  --set controller.nodeSelector.<label-key>=<label-value> \
  --wait
kubectl get pods -n ingress -o wide   # new pod must land on the dedicated node
kubectl get deploy -n ingress ingress-nginx-oadc-controller \
  -o jsonpath='{.spec.template.spec.containers[0].image}'
```

Notes: `--reuse-values` is not recommended for this jump — old values may
pin an image (`controller.image.tag: v1.12.3` or similar) that would
override the chart's new default. The same trap applies to
`-f old-values.yaml` after `helm get values -a`: `-a` dumps COMPUTED
VALUES, i.e. the old chart's defaults merged in, so that file itself
contains the old image repository/tag — feeding it back would reinstall
the old (vulnerable) image while looking like a successful upgrade.
Prefer an explicit values file. Do not
`helm rollback` to the pre-upgrade revision afterwards without re-checking
the image — a rollback restores the old chart values (old image).

## Upstream continuation — Chainguard fork (reference, 2026-11-26)

Upstream `kubernetes/ingress-nginx` was archived 2026-03-24 and ships no
more fixes. The maintained continuation is Chainguard's EmeritOSS fork:
https://github.com/chainguard-forks/ingress-nginx
(self-described as "a supported replacement of the original repository";
best-effort security fixes only, no community contributions, no free
pre-built images or charts — you build and host images yourself;
hardened images are a commercial cgr.dev offering).

Why it matters for this fork:

- Its release `controller-v1.15.16` does the same NGINX2Root fix a
  different way: it **backports the nginx 1.30.7 security fixes onto
  1.27.1** (PR #67) instead of upgrading nginx. Chainguard's own advisory
  confirms that build fixes all five NGINX2Root CVEs — an independent
  cross-check that the five CVEs we fixed are the right set. Note their
  image still reports `nginx/1.27.1` in its banner: backported fixes do
  not show in the version string.
- This fork's approach (a real 1.30.7 upgrade) is the cleaner one: the
  genuine upstream release, an honest version string, and every module
  recompiled against 1.30.7, so there is no ABI drift. The Chainguard
  backports add fields to core nginx structs (e.g. a `send_queued`
  counter for CVE-2026-123007), so a third-party dynamic module built
  against stock nginx could be ABI-incompatible with their image.
- For the next nginx CVE there will be no official ingress-nginx
  release. Watch this fork's `images/nginx/rootfs/patches/` — that is
  where future backports will land, and the place to cherry-pick from
  when the next fix is needed here.
