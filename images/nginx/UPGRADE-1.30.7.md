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
- Still to verify in the full image build: the third-party modules
  (lua-nginx-module, ModSecurity, geoip2, njs, OpenTelemetry) and HTTP/3.

## Updating next time

1. Bump `NGINX_VERSION` in `images/nginx/rootfs/build.sh` and update the
   tarball SHA-256 next to it (hashes are published on nginx.org).
2. Re-run the patch check: download the new tarball, apply every file in
   `images/nginx/rootfs/patches/` in `ls` order with `patch -p1`. Any patch
   that now fails is either upstreamed (drop it, after verifying the fix is
   in the new source) or needs a hand rebase like patch 19 above.
3. Bump `images/nginx/TAG`, rebuild base + controller, re-verify `nginx -v`.
