---
name: lvtv-elastic-logs
description: Query and investigate LVTV Elasticsearch logs through local Boundary tunnels to the LVTV logs MCP servers for main/core, integrations, and dynamics clusters. Use when Codex needs to inspect Kibana/Elasticsearch logs, choose the correct cluster, discover indices and fields, analyze recent incidents, compare time windows, diagnose status/error spikes, study crawler/parser traffic, or explain request flows across Level Travel services.
---

# LVTV Elastic Logs

## Purpose

Use this skill as a generic investigation loop for LVTV Elasticsearch logs. Do not start from memorized field names or one past incident; first discover the index, sample its documents, infer the schema, then build narrow aggregations.

## Connection

Use one of three MCP profiles:

| Profile | MCP URL | Use for |
|---|---|---|
| `main` | `http://127.0.0.1:32022/mcp` | Core/app logs through Boundary target `k8s-core`: gateway, web-gateway, rails, nginx, nextjs, sidekiq, SEO, analytics, staging, jobs, and most `fbyc-*` app services. |
| `integrations` | `http://127.0.0.1:32012/mcp` | Integration/search pipeline logs through Boundary target `k8s-integrations`: actualizer, actualization-cache, booker, currency, ghe, go-tour, search-calendar, search-generator, searcher, stats-updater, storage. |
| `dynamics` | `http://127.0.0.1:32002/mcp` | Dynamic pricing/search logs through Boundary target `k8s-dynamic`, mainly `fbyc-dynamic`. |

Assume the user has already handled Boundary authorization and target connections locally. Do not send an `Authorization` header, do not read `~/elastic-mcp-token`, and do not run Boundary authentication, target connection, API calls, or target authorization from this skill. If a localhost port is not listening, report the missing Boundary tunnel and ask the user to open the corresponding target: `k8s-core` for `main`, `k8s-integrations` for `integrations`, or `k8s-dynamic` for `dynamics`.

Prefer the bundled helper from the repository root:

```bash
python3 .agents/skills/lvtv-elastic-logs/scripts/mcp_call.py --profile main list_indices '{"index_pattern":"*gateway*"}'
python3 .agents/skills/lvtv-elastic-logs/scripts/mcp_call.py --profile integrations list_indices '{"index_pattern":"*searcher*"}'
python3 .agents/skills/lvtv-elastic-logs/scripts/mcp_call.py --profile dynamics list_indices '{"index_pattern":"*dynamic*"}'
python3 .agents/skills/lvtv-elastic-logs/scripts/mcp_call.py --profile all find_index '{"index_pattern":"*unknown-fragment*"}'
```

## Generic Workflow

1. **Choose a profile if obvious.** Use the routing table above. If the user names an unknown index or only gives a fragment, search all profiles with `--profile all find_index`.
2. **Discover the index.** Use `list_indices` with broad patterns (`*gateway*`, `*rails*`, `*nginx*`, `*searcher*`, service-name fragments). Prefer datastream aliases such as `fbyc-gateway` when they work.
3. **Check freshness.** Query `max(@timestamp)` and the latest 1-3 docs. Confirm the index has data in the requested time window before analyzing absence.
4. **Sample documents.** Pull a few recent docs with `_source` to learn field names, nested shape, status fields, request fields, and Kubernetes metadata.
5. **Probe aggregatable fields.** For any candidate field, test a small `terms` aggregation on both the base field and `.keyword` if needed.
6. **Build the first aggregation.** Start with time range + top dimensions: status, error, path/action, user-agent, IP, pod, image, host/origin.
7. **Drill down.** Add filters for the dominant status/error/UA/path and fetch sample docs. Avoid relying only on bucket counts.
8. **Compare windows.** Compare `now-5m` against `now-30m`, `now-24h`, or a previous known-good interval to distinguish incident spikes from normal baseline.
9. **Explain with confidence.** Separate observed facts from inference. Mention profile, index, time window, and field/mapping uncertainty.

## Response Handling

The MCP returns Server-Sent Events lines:

```text
data: {"jsonrpc":"2.0", ...}
```

Tool output is usually in `result.content[].text`. Aggregation responses often contain a label chunk, then a JSON string chunk. Parse JSON-looking chunks from all content entries.

`size: 0` responses may omit `hits` and return only the aggregation object. Compute totals from buckets or add an explicit aggregation.

## Field Discovery Rules

Do not assume fields are consistent across indices. Use these as starting points only:

| Meaning | Common fields |
|---|---|
| Timestamp | `@timestamp` |
| Status | `json.status_int`, `json.status`, `status` |
| Error reason | `json.error_message.keyword`, `json.error.keyword`, `message` |
| Path / route | `json.incoming_path.keyword`, `json.controller_action.keyword`, `json.path`, `url.path` |
| Full request | `json.request`, `message` |
| User-agent | `json.user_agent`, `user_agent.original` |
| Remote IP | `json.remote_ip`, `client.ip`, `source.ip` |
| Pod | `kubernetes.pod.name` |
| Image | `container.image.name` |
| Host / origin | `json.incoming_host.keyword`, `json.origin.keyword`, `host.name` |

If a `terms` aggregation returns empty unexpectedly, try:

- the same field with `.keyword`;
- the same field without `.keyword`;
- a sample-doc query to confirm the field exists in the selected window;
- a broader time window to exclude late ingestion or no fresh data.

## Datastream And Mapping Gotchas

- `get_mappings` can fail or be unhelpful on datastream aliases. Prefer sample docs and field probes.
- If strict mappings are needed, call `list_indices` and pass a concrete backing index such as `.ds-...-000166`.
- Avoid making conclusions from a zero-result query until freshness and field names are verified.

## Investigation Patterns

### Incident / Error Spike

Use when the user reports "посыпались 403/500", latency, missing data, or a fresh production issue.

1. Count statuses by minute.
2. Split the failing status by `error_message`, path/action, pod, image, UA, IP.
3. Search raw `message` for adjacent infrastructure errors (`bad connection`, timeout, refused, reset, panic).
4. Check whether failures are concentrated in one pod/image or broad across all pods.
5. Fetch representative failed docs and one successful doc from the same path for comparison.

### Crawler / Parser Traffic

Use when the user asks who is parsing/crawling or what flow a client follows.

1. Aggregate by user-agent over `now-24h` and `now-3h`.
2. Filter to scripted UA families (`Go-http-client`, `aiohttp`, `python-requests`, `Typhoeus`, `rest-client`, `curl`, `httpx`, `Scrapy`, `okhttp`, `Swagger-Codegen`).
3. For each UA, split by IP and endpoint/action.
4. Compare recent activity with previous observations. Report whether it is still active, reduced, stopped, or changed UA/IP.
5. Recommend mitigations using `(UA + IP/prefix + endpoint group)` before broad ASN blocks.

### Request Flow

Use when the user asks "какой flow", "по каким отелям", "какие ручки".

1. Identify stable client dimensions: UA, IP, token/key, request id, session-like params.
2. Group by endpoint/action and order sample docs by timestamp.
3. Decode domain-specific parameters only after locating code/docs that define them.
4. Report sequence and uncertainty: direct API client, browser, mobile app, bot, known crawler, or internal service.

## Known LVTV Examples

These examples are not universal rules; they are anchors from past investigations:

- `main/fbyc-gateway`: API gateway logs. Useful fields include `json.status_int`, `json.error_message.keyword`, `json.incoming_path.keyword`, `json.request`, `json.remote_ip`, `json.user_agent`, `kubernetes.pod.name`, `container.image.name`.
- `main/fbyc-web-gateway`: web gateway logs. Similar gateway shape, often page/web traffic.
- `main/fbyc-rails`: Rails app logs. Endpoint/action is commonly `json.controller_action.keyword`.
- `main/fbyc-nginx`: nginx access logs. Common fields include `json.status`, `json.uri`, `json.args`, `json.http_user_agent`, `json.remote_addr`, `json.http_x_forwarded_for`.
- `integrations/fbyc-searcher`: searcher logs. Common fields include `json.event`, `json.msg`, `json.success`, `json.duration`, `json.count`.
- `integrations/fbyc-go-tour`: tour integration logs. Common fields include `json.operator_name`, `json.partner_id`, `json.search_request`, `json.request_url`, `json.success`, `json.total_tours`, `json.unmatched_hotels`.
- `integrations/fbyc-storage`: gRPC storage logs. Common fields include `json.grpc.service`, `json.grpc.method`, `json.grpc.code`, `json.grpc.time_ms`, `json.request_id`.
- `dynamics/fbyc-dynamic`: dynamic service logs. Common fields include `json.grpc.service`, `json.grpc.method`, `json.incoming_request_path`, `json.search_job_id`, `json.search_scope`, `json.search_type`, `json.supplier_name`, `json.unique_hotels_found`.
- A `partner fetch error` with `code=403, message=partner not found` in gateway logs can be a real missing partner, but if it spikes with `message:"bad connection"`, suspect DB lookup failure.

## References

Read `references/queries.md` for generic query templates and LVTV-specific examples.
