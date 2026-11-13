# infra-blocks-k3s

[![Blocks](https://img.shields.io/badge/infra--blocks-index-555?style=flat-square)](https://github.com/kamaslau/infra-blocks)
[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](https://opensource.org/licenses/MIT)
![Repository size](https://img.shields.io/github/repo-size/kamaslau/infra-blocks-k3s?color=56BEB8)

A unit, not a demo — but the only one here that installs onto the host instead of into a compose file. There is nothing to `down -v`; the uninstall step is documented below.

Typical use: the client has a server room *and* two cloud hosts, and wants one place to deploy instead of three ways to copy files.

[K3s](https://docs.k3s.io/) — Kubernetes in a single binary — with [Tailscale](https://tailscale.com) carrying the node-to-node traffic, so the cluster can span networks nobody controls.

## Service URL

- Kubernetes API `https://<node>:6443`
- Ingress (Traefik, shipped with k3s) `http://<node>:80` / `https://<node>:443`
- No web UI. `kubectl` is the interface.

Port 6443 is the one a firewall rule is written against. There is no `.env` in this repository: the ports and addresses come from the flags in the install command, and the flags are in `plan/WithTailScale.md` with the reason each one is there.

## Default user

None. Authority is the cluster token plus the kubeconfig:

| Item | Where it lives |
| ---- | -------------- |
| Cluster token | `/var/lib/rancher/k3s/server/node-token` inside the first server (`K3S_TOKEN`) |
| Kubeconfig | `/etc/rancher/k3s/k3s.yaml` inside any server, mode 644 in this setup |
| Client access | the K3s binary's own `kubectl`, run inside the container |

Anyone holding the token can join a server to this cluster. Treat it as a root credential, not a join code.

## Usage

The layout is three servers with embedded etcd, each also running a worker, connected over Tailscale — a topology that survives the loss of one node and does not need a separate agent container anywhere.

1. Read `plan/WithTailScale.md` end to end. Phase 0 installs Tailscale on every node, Phase 1 bootstraps the first server, Phases 2 and 3 join the others.
2. Bootstrap, on the first node only:

   ```bash
   export K3S_TOKEN=$(openssl rand -hex 32)
   TS_IP=$(ip addr show tailscale0 | awk '/inet /{print $2}' | cut -d/ -f1)

   docker run -d --privileged --name k3s-m1 \
     --restart unless-stopped --network host \
     -e K3S_CLUSTER_INIT=true \
     -e K3S_KUBECONFIG_MODE=644 \
     -e K3S_TOKEN=$K3S_TOKEN \
     -v /etc/resolv.conf:/etc/resolv.conf:ro \
     rancher/k3s:v1.36.1-k3s1 server \
     --tls-san $TS_IP --advertise-address $TS_IP --node-ip $TS_IP
   ```

3. Join the other two with `--server https://<first-node-tailscale-ip>:6443`, the same token, and each node's own `--node-ip`.
4. Read `plan/TRAPS.md` before the first join fails. Every entry in it is a failure that produced a misleading error.

`rancher/k3s` is a container image here, but the container runs `--privileged --network host` because k3s writes to `/var/lib` and binds host ports. If the target is a single host and the team is comfortable with a systemd unit, the upstream `curl -sfL https://get.k3s.io | sh -` install produces the same cluster without Docker in the middle; the flags carry over.

### Uninstall

```bash
docker rm -f k3s-m1 k3s-m2 k3s-m3
docker volume rm $(docker volume ls -q --filter name=k3s)   # if you added one
```

For the binary install, `/usr/local/bin/k3s-uninstall.sh` on servers and `k3s-agent-uninstall.sh` on agents.

## Verify

From the first server, after Phase 1:

```bash
docker exec k3s-m1 kubectl get nodes -o wide
```

Three nodes, all `Ready`, `ROLES` showing `control-plane,etcd`, and `INTERNAL-IP` in the `100.x` range for every one of them. If the IPs are `172.x` bridge addresses, the `--node-ip` flag did not take and etcd is talking to an address the other hosts cannot reach.

```bash
docker exec k3s-m1 kubectl get pods -A
```

`coredns`, `local-path-provisioner`, `metrics-server` and the Traefik helm jobs in `kube-system`, reaching `Running`. `ContainerCreating` for more than a minute or two is usually an image pull against a registry that cannot be reached from that node.

## Why this shape

**Three servers, not one.** A single k3s server is one disk away from a rebuild; three with embedded etcd tolerate the loss of any one node and still need no external database.

**Tailscale, not VPC or public addresses.** Private VPC ranges (`172.17.x.x`, `10.x`) are not routable across providers, and public addresses are NATted by the cloud — etcd cannot bind to an address it does not own (`bind: cannot assign requested address`). A Tailscale `100.x` address is bindable, routable, and identical on a Raspberry Pi behind a home router and a VPS in another country.

**Containers, not systemd, where Docker already exists.** The nodes in this scenario all run Docker for other reasons; `docker run` keeps the install step consistent with the rest of the collection, and `docker rm -f` is a complete undo. The trade-off is real: k3s in Docker needs `--privileged` and host networking, and the upstream binary install is the better answer on a host dedicated to Kubernetes.

**k3d is not used**, and `plan/TRAPS.md` explains why in detail: k3d publishes its nodes on a Docker bridge, so etcd peers advertise `172.18.0.x` addresses that no other host can reach, and a multi-host cluster hangs on `MemberAdd request timed out`.

**No separate agent containers.** A k3s server runs a worker itself. Adding an agent on the same host means two processes fighting over port 6444, and the error (`address already in use`) does not mention the server that is holding it.

## Notes for deployments in China

Two failures dominate, both documented in `plan/TRAPS.md` with the fix:

- **DNS inside the container.** Aliyun hosts resolve through `100.100.2.x`, which cannot resolve Docker Hub. `containerd` then fails every pull with a timeout that looks like a network outage. Bind-mount `/etc/resolv.conf` and set a public resolver first.
- **Registry reachability.** `containerd` does not read `/etc/docker/daemon.json`. Mirror configuration goes in `/etc/rancher/k3s/registries.yaml` on each node, and it has to be in place before the first `kubectl apply` of anything that pulls an image.

Do not enable the Tailscale exit node on these hosts: it routes all traffic, including the etcd peer traffic the cluster depends on, through one machine.

## Repository layout

| Path | What it holds |
| ---- | ------------- |
| `plan/WithTailScale.md` | The three-phase bootstrap, with the expected output of each step |
| `plan/TRAPS.md` | Failures that produced a misleading message, and their causes |

---

Part of [infrastructure blocks](https://github.com/kamaslau/infra-blocks) · ← back to index

