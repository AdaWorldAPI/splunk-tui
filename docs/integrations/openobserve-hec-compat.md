# OpenObserve Splunk HEC smoke test: bounded compatibility contract

**Status: POC source / NOT tested against live OpenObserve**, 2026-11-15.

This fork is a Splunk REST CLI/TUI, not an OpenObserve CLI. Its HEC sender works against a separately configured HEC URL and OpenObserve implements a subset. This is not whole-product compatibility.

## Source-pinned facts

- crates/cli/src/commands/hec.rs: parses HEC commands. Before this PR, --ndjson still expected a JSON array input. Now parses one event per line, rejects invalid line and nonzero HEC response codes.
- crates/client/src/endpoints/hec.rs: POST /services/collector/event, GET /services/collector/health, POST /services/collector/ack.
- AdaWorldAPI/openobserve at 624410da55a5f1c1b994c12e4ef6f64f7f26647a: source route allowlist implements /services/collector, /services/collector/event, /services/collector/health ONLY; no /services/collector/ack or /raw; HEC GUID token bound to an OpenObserve organization.
- AdaWorldAPI/sentinel-rs POC: https://github.com/AdaWorldAPI/sentinel-rs/tree/main/poc/openobserve-hec has synthetic Entra/Exchange-shaped fixtures and a separate Search API verifier.

| Function | OpenObserve POC expected | Boundary |
| --- | --- | --- |
| splunk-cli hec send | Compatible POST /services/collector/event | HEC 200/code 0 NOT an indexed/read-back proof |
| splunk-cli hec send-batch --ndjson | Now accepts real NDJSON file, emits NDJSON | Does not guarantee no duplicates on ambiguous POST |
| splunk-cli hec health | HTTP 200 and body containing HEC is healthy | OpenObserve body is JSON object, not Splunk's plain text |
| splunk-cli hec check-ack | UNSUPPORTED | OpenObserve does not expose this endpoint |
| splunk-cli search / general splunk-tui | UNSUPPORTED as OpenObserve management/search client | Uses Splunk REST, not OpenObserve /api/{org}/_search |
| Search-back | REQUIRED to claim successful test | Use OpenObserve Search API to check exact event IDs and scope |

## Disposable local POC recipe

1. Start OpenObserve on localhost-only according to https://github.com/AdaWorldAPI/sentinel-rs/tree/main/poc/openobserve-hec and generate a new HEC GUID token. Do not expose HTTP publicly.
2. In sentinel-rs, run the Rust offline generator: cargo run -p sentinel-core --example openobserve_hec_fixture and redirect stdout into a local .ndjson file. No actual Microsoft 365 data.
3. Set SPLUNK_HEC_URL to http://127.0.0.1:5080 and SPLUNK_HEC_TOKEN to the new GUID via protected local env. Confirm exact options with splunk-cli hec send-batch --help, then send the generated file with --ndjson.
4. Independently query OpenObserve's /api/default/_search endpoint or UI to confirm every event ID, tenant alias and evidence_grade=fixture. The existing sentinel-rs poc.py sends before verifying; if running the CLI as sender, use manual read-back or add a verify-only mode to poc.py in a separate change, not a second send.
5. Do not call hec check-ack expecting indexer delivery confirmation.

## Must-fail controls

- Invalid second NDJSON line refuses before any POST and includes input line number.
- NDJSON in JSON-array mode fails; JSON array mode remains supported.
- HTTP 200 but HEC code 4/7 refuses with error exit.
- Code 0 but no indexed read-back remains UNVERIFIED.
- A source event with untrusted tenant/customer metadata may never authorize a Sentinel-rs customer scope.
- Do not turn this CLI or HEC into a privileged raw SPL query proxy.

This change is limited to transport fidelity. It does not implement a SIEM or add a Lance/Arrow execution adapter.
