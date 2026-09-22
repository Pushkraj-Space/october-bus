# Hosted MCP connector pilot report (September 2026)

Status record for [issue #122](https://github.com/october-dev/october-bus/issues/122): deploying and verifying the hosted October Bus MCP connector for the Muse Connector Platform. It records what was deployed, what was proven, what was found, and what remains. It is not a claim of Muse approval; Meta's review is tracked separately.

## Outcome

`https://bus.october.dev/mcp` is live on a single Linux server and every verification gate in issue #122 §5 passes on that address, including a Muse custom-connector round trip and a real coding harness on the receiving side. The connector is an operator-provisioned pilot: one scope, one connector key, explicitly linked agents, no self-service, no OAuth, no automatic wake-up.

## Code

| Item | Value |
| --- | --- |
| Gateway implementation | [PR #120](https://github.com/october-dev/october-bus/pull/120) `feat/hosted-mcp-gateway` @ `dba3ece` |
| Fixes found by this deployment | [PR #125](https://github.com/october-dev/october-bus/pull/125) `fix/hosted-gateway-review` @ `7fc501a`, stacked on #120 |
| Deployed build | `october-bus 0.1.0-pr120fix.7fc501ac58a1` (static linux/amd64, `-X bus.Version` stamped) |
| Verification | `go test -race ./...`, `go vet`, `gofmt -l` clean; nine CI checks green on #125 |

Defects in #120 found and fixed in #125: gateway start racing the daemon after reboot and tripping systemd's start limit; `connectTo` in gateway config failing on a fresh deployment; shared request budget taken before authentication; `GET`/`DELETE /mcp` allowlisted although the daemon is stateless; connector and bridge executions never leaving `starting`; forwarded-header hygiene; missing 429/502 tests.

## Deployment

| Item | Value |
| --- | --- |
| Host | Hetzner Cloud, US East, Ubuntu 26.04, 2 vCPU / 2 GB / 38 GB |
| DNS | `A bus.october.dev` → server IPv4; no AAAA |
| Services | `october-bus.service` (daemon, `127.0.0.1:4765`) and `october-bus-gateway.service` (gateway, `127.0.0.1:8787`), user `october-bus`, state `/var/lib/october-bus`, runtime `/run/october-bus` |
| HTTPS | Caddy, Let's Encrypt certificate for `bus.october.dev`, HSTS; only 22/80/443 open |
| Provisioning | `scope create muse-pilot`; `gateway init --public-url https://bus.october.dev --scope muse-pilot --agent muse`; key stored as digest only; no `connectTo` |
| Backup | `/etc/cron.daily/october-bus-backup`: `october-bus backup` snapshot plus private credential files, 0700 directory, 14-day retention; restore tested into a scratch daemon |
| Drills | gateway restart 3 s to ready; daemon restart 3 s (gateway waits for readiness); two full reboots recovered in 22 s and 24 s |

Incident during deployment: Ubuntu's packaged Caddy 2.6.2 panicked on `systemctl reload caddy` and its unit has no `Restart=`, leaving port 443 down until restarted. Mitigation: a drop-in with `Restart=on-failure`; use `systemctl restart caddy` on this build, or install current Caddy from the official repository.

## Verification gates (issue #122 §5)

| Gate | Result on `bus.october.dev` |
| --- | --- |
| Public endpoint | `/health/ready` 204 over public HTTPS from outside; valid certificate; no interactive proxy challenge |
| Authentication | missing key 401 with `WWW-Authenticate: Bearer`; invalid key 401; valid key works; retired keys non-functional |
| Scope boundaries | admin routes, traversal and encoded paths 404; two-scope isolation (no peers, no messages, no receipts across scopes) verified on staging |
| MCP protocol | `scripts/check-hosted-mcp.mjs` passes; 15 tools; stateless JSON responses; `GET /mcp` 405 |
| Useful work | Muse request → gateway → `mcp stdio --remote` bridge → Claude Code read a file on the laptop and replied → Muse displayed the reply; both receipts `acknowledged`; about 15 s end to end |
| Recovery | restarts and reboots above; idempotent replay returns the original message id and a mismatched body is refused; requests queue while the agent is offline and deliver once on reconnect; gateway restart preserves queued work; host suspend ends leases cleanly with data intact |
| Muse-specific | Muse custom connector on the production URL: secure credential entry, Bearer header, tool discovery, approval prompt before each tool call, `message_peer`, same-turn `check_inbox` retrieval, `acknowledge_messages` |

Clients exercised: Muse as connector client; Claude Code 2.1.278 as connector client over HTTP MCP with a Bearer key, and as receiving harness through `mcp stdio --remote`. No other product names were tested.

## Findings for documentation

- LLM connector clients poll `check_inbox` on their own after `message_peer` when the receiving agent answers quickly, with or without a "wait for the reply" instruction. Product copy may describe same-turn results for online agents but must still say results are collected later when the agent is slow or offline.
- Delivered but unacknowledged inbox messages are redelivered on later `check_inbox` calls. Muse did not always acknowledge replies; docs should tell connector clients to acknowledge after reading.
- One connector client omitted the optional `idempotencyKey`. The `message_peer` description should recommend always supplying a fresh one.
- The connector agent id (default `muse`) is reserved for the gateway; a bridge registering the same id displaces the connector execution.
- A laptop that suspends loses its execution lease; its bridge must be restarted. The daemon and all stored messages are unaffected.

## Remaining items (owner)

- Product page update at `www.october.dev/october-bus` (website repository).
- 512 × 512 PNG logo under 256 KiB, validated before upload.
- Operator contact and a private credential-delivery path for reviewers.
- Submission at the Muse Connector Platform and tracking of Meta's review.
- Merge order: #120, then #125 retargeted to `main`.
