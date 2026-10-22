# openwrt-asu

Helm chart that runs an [OpenWrt Attended Sysupgrade (ASU)](https://github.com/openwrt/asu) server on Kubernetes,
pinned to a dedicated build node and reachable **only from a private mesh network** (Tailscale/Headscale).

ASU builds firmware images by assembling **already compiled packages** inside an OpenWrt ImageBuilder container.
It does not recompile OpenWrt or the kernel. Upstream expects a Podman socket, which Kubernetes nodes
(containerd) do not provide, so this chart adds a rootful Podman service as a sidecar.

## Architecture

One pod (single replica, `Recreate` strategy) with three containers:

| Container | Role |
|-----------|------|
| `server`  | ASU API (`uvicorn`, port 8000), serves built images read-only |
| `worker`  | `rqworker`: for each build, asks Podman to start an ImageBuilder container |
| `podman`  | `podman system service` on a Unix socket shared through an `emptyDir`; creates the isolated `asu-build` network |

Redis (`redis-stack-server`) runs in a separate Deployment; it only caches job state and build results.

Built images (`/public`) and Podman's image storage live on the node (`hostPath`, default under
`/srv/build-data/asu/`) or on PersistentVolumeClaims (`persistence.type: pvc`).

## Security notes

* The `podman` container is **privileged**. Each build still runs in its own unprivileged container
  (no capabilities, `no-new-privileges`, dedicated network without access to Redis), but the sidecar itself
  has full access to the node. Keep this chart on a dedicated build node.
* The ingress restricts source addresses with `haproxy.org/allow-list` (default: the Tailscale CGNAT/ULA ranges),
  because host-network ingress controllers also listen on nodes with a public address. Do not expose it publicly.
* Images are **not signed** unless a `key-build` is provided to the worker (not handled by this chart yet).

## Install

```bash
helm install asu helm/openwrt-asu -n asu --create-namespace \
  --set ingress.host=asu.example.internal
```

Relevant values (see [values.yaml](helm/openwrt-asu/values.yaml)):

| Value | Purpose |
|-------|---------|
| `config` | content of `asu.toml` (upstream URL, branches, `repository_allow_list`, timeouts…) |
| `nodeSelector` / `tolerations` | default `workload=build` and the matching `NoExecute` taint |
| `persistence.type` | `hostPath` (default) or `pvc` |
| `ingress.host` / `ingress.allowList` | private host name and allowed source networks |

## Custom packages

To offer packages built from patched sources (for example a modified `wireless-regdb`), publish them in an
apk repository reachable by the build containers and add its URL prefix to `config.repository_allow_list`;
clients then pass it in the `repositories` field of the build request.

## License

MIT, see [LICENSE](LICENSE).
