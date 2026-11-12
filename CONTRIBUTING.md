# Contributing

This repository is one unit of [infrastructure blocks](https://github.com/kamaslau/infra-blocks): single-binary Kubernetes, with Tailscale joining cloud and on-prem nodes into one network.

Default port: 6443 (API server). It comes from `.env`; nothing here hardcodes it.

## The shape a change has to keep

One service, one port, one documented credential, removable with a single command.

A second container is acceptable only when it is the user interface of the first — Adminer beside PostgreSQL, Grafana beside Prometheus. A queue, a sidecar, a proxy or a "small helper service" is not: it turns a unit into a stack, and a stack is the thing this repository exists to avoid.

The test for any change: could someone who has never seen this repository run it from the README alone, and remove it without leaving anything behind?

## Pull requests

- **Ports come from `.env`.** If you add a variable, add it to `.env.example` (or `.env.sample`) with a working default, and mention it under Service URL.
- **The README is English.** Chinese belongs only under *Notes for deployments in China*. That section exists because it is the part a foreign HQ cannot supply; mixed into the body, it stops being findable.
- **`Verify` has to keep proving something.** Any behaviour change updates the command in that section. A `Verify` that returns success while the service is down is worse than no `Verify`.
- **Credentials stay marked local-only.** If a change makes the default credential reachable from outside the host, say so in the README in the same commit.
- **Image tags**: `latest` is a deliberate choice for a disposable unit. Pin a tag only when a floating one would break documented behaviour, and write down which behaviour.
- **Do not rename** the compose project, the containers or the volumes. Existing clones keep their data under the current names, and a rename orphans it without an error message.
- **Commits**: imperative subject, one concern per commit. `fix: mount the volume at /var/lib/postgresql/data` is the shape. `update` is not.

## Adding a unit

Open an issue on [infra-blocks](https://github.com/kamaslau/infra-blocks) before writing anything. The list is meant to stay short, and a unit that overlaps an existing one is a maintenance cost rather than an addition.

If it is accepted:

1. Repository named `infra-blocks-<slug>`, MIT license, description in the form *what it is + when a client needs it*.
2. README with the six sections used across the suite: title and one-line purpose, Service URL, Default user, Usage, Verify, Why this shape — plus *Notes for deployments in China* only when there is something true to write there.
3. A row in the index tables, under Runtime or Storage, with its default port.

## Reporting a problem

Open an issue with four things: the command you ran, what you expected, what happened, and the output of `docker compose ps`. Without the last one, most reports turn into guesswork.

---

Part of [infrastructure blocks](https://github.com/kamaslau/infra-blocks) · MIT
