# Live verification

The record of the live verification this deployment passed, per the workspace
Compute Provider Standard §7-8 and the plan that introduced it. Commands,
outcomes and timings only: no address, key id or token appears here.

## What was verified

| | |
|---|---|
| Date | 2026-09-04 (UTC) |
| Provider | DigitalOcean, region `ams3` |
| Plan | `s-4vcpu-8gb`, image `ubuntu-24-04-x64` |
| Package | `getcolors/clickstack` at `68887b5` (pin commit `8b43c1c`) |
| Mode | keygen (no `digitalocean-ssh-keys`), compute name = profile |
| State | R2, `clickstack-digitalocean/clickstack-infrastructure.tfstate` |

## Sequence and outcomes

| # | Command | Outcome |
|---|---|---|
| 1 | `./green create` | **refused before any provider call** by the SSH Keypair Standard account preflight: `cannot list digitalocean SSH keys for the create preflight: HTTP 401`. The DigitalOcean token copied from a sibling deployment had been revoked. Nothing was created. |
| 2 | `./green create` (live token) | **refused before any provider call** by the keypair create matrix: `~/.ssh/clickstack-digitalocean exists but no compute state is readable`. Attempt 1 had generated the keypair before its preflight failed. Verified read-only at DigitalOcean that no droplet and no account key named `clickstack-digitalocean` existed, removed the two local key files, retried. |
| 3 | `./green create` | **exit 0.** Stages: start 0.3 s, infrastructure 89 s (droplet in the region's default VPC, cloud firewall, account key), ssh-config 3 s, dns 5 s, ansible 190 s, acceptance 0.4 s (OTLP ingestion over public HTTPS read back from ClickHouse). |
| 4 | `ssh clickstack-digitalocean` | hostname answered; five containers up, app and collector healthy. |
| 5 | `./red create` | **exit 0**, idempotent: infrastructure 3 s, ssh-config 2 s, dns 3 s, ansible 70 s, acceptance 0.3 s. |
| 6 | `./blue create` | **exit 0**, idempotent: infrastructure 3 s, ssh-config 2 s, dns 3 s, ansible 70 s, acceptance 0.2 s. |
| 7 | `COLORS_PAR_PROVIDER_COMPUTE=vultr ./green create` | **exit 2** before any credential or provider call: `state holds a digitalocean machine; set provider-compute back to digitalocean and delete first`. No Vultr credential was demanded. |
| 8 | `COLORS_PAR_PROVIDER_COMPUTE=vultr ./green delete` | **exit 2**, the same refusal, ahead of the prevent-destroy guard. |

Attempts 1 and 2 are the standard working as designed: the preflight and the
create matrix both stop a run while stopping is still free. Attempt 3 was a
fresh create; 5 and 6 prove the three colours manage one state
interchangeably; 7 and 8 prove Compute Provider Standard §4 on a live state.

## Not verified here

- The Vultr side of the package, which `clickstack-vultr` covers.
- DigitalOcean cloud-firewall behaviour on VPC traffic: nothing in this
  single-node deployment depends on it and no gate exercises it.
