# Manual MCP Client QA

This suite exercises the MCP surface the way an approved MCP Client does: over
`POST /mcp`, through a real grant, with no access to internal state. It exists
because automated coverage stops at the public boundary described in
[`docs/testing.md`](testing.md), and some behavior is only observable from a
client holding a live grant against a real WhatsApp Connection — initial
Directory sync, Ingestion Gap reporting, Connection state transitions, cursor
binding across grants, and Client Confirmation.

[`docs/mcp-contract.md`](mcp-contract.md) is the oracle. Every case below cites
the rule it tests. If an observation disagrees with a case, the contract
decides: either the server is wrong or the case is, and the case is the more
likely of the two. Do not "fix" a case to match observed behavior without
changing the contract first.

## What this suite is not

It does not replace the public-boundary suites, the database suites, or the
browser journeys. It adds no automation and runs on no schedule. Treat a run as
evidence for an issue, not as a merge gate.

## Recording rules

These are binding. A run log is committed or pasted into an issue, so it must
carry no tenant data.

Record only shapes, counts, enum values, booleans, timestamps, error codes, and
latencies. Never record contact or group display names, phone numbers or their
last four digits, message text or captions, filenames, cursors, opaque handles,
media bytes, or `Authorization` values. Where a case needs a handle, write
`<con>`, `<cvs>`, `<ctc>`, `<grp>`, `<msg>`, `<med>`, or `<snd>`. Where it needs
a search term, write `<term>` and note only its normalized term count.

This mirrors the platform's own rule: no tool, log, or telemetry emits message
content, media, credentials, full phone numbers, provider payloads or
identifiers, or tenant identifiers.

## Preconditions

A run needs, at minimum:

- One Personal Account with at least one WhatsApp Connection in `connected`
  state that has observed Stored Message activity in both a direct and a group
  Conversation.
- One WhatsApp Connection linked within the last few minutes, for the
  initial-sync cases. A run that skips this cannot cover section K.
- Three grants, to cover cursor binding: two MCP Authorizations and one API Key
  matching `normal_apk_<handle>.<secret>`.
- Grants at three scope levels: `connections:read` alone, plus
  `directory:read`, plus `messages:read`. Sections M requires `messages:send`
  and is opt-in.

Record, per run: UTC start and end, client name and version, grant kinds, scope
sets, Connection count and states, and whether section M ran.

Sections B through L are read-only and safe to run against a production
Personal Account. **Section M sends real messages to real people.** Run it only
against a Connection whose recipients have consented to test traffic.

## Case tables

Expectations cite `docs/mcp-contract.md` unless noted. `PASS` requires the
entire expectation, including nullability and field presence — "Unknown optional
values are returned as `null`, not omitted", so a missing key is a failure even
when the value would have been null.

### B. Discovery and scope gating

| ID | Action | Expected |
| --- | --- | --- |
| B1 | `tools/list` with `connections:read` only | Only `list_connections` is advertised. Every other tool is omitted from discovery when its scope is absent. |
| B2 | `tools/list` with `connections:read` + `directory:read` | Adds exactly `list_contacts` and `list_groups`. |
| B3 | `tools/list` with `messages:read` added | Adds exactly `list_chats`, `read_messages`, `search_messages`. |
| B4 | `tools/list` with `messages:send` added | Adds exactly `send_text_message`, `send_pdf_file`, `send_image`, `get_send_status`. |
| B5 | Call a tool whose scope the grant lacks, invoking it directly rather than from the advertised list | Rejected. Each tool rechecks its scope in the handler; discovery omission is not the only gate. |
| B6 | `resources/templates/list` with and without `messages:read` | The Stored Media template appears only with `messages:read`. `resources/list` returns an empty list either way, never a bulk media inventory. |

### C. `list_connections`

| ID | Action | Expected |
| --- | --- | --- |
| C1 | Call with `{}` | Success. Input is an empty object; the tool is not paginated. |
| C2 | Call with any property added | Schema rejection. Tool inputs are closed objects with `additionalProperties: false`. |
| C3 | Inspect each entry | `display_name` always present; `number_last_four` nullable; `state` one of `connected`, `connecting`, `disconnected`, `reconnect_required`, `degraded`; `state_changed_at` an RFC 3339 UTC string. |
| C4 | Compare the count against the Personal Account's Connections | Only non-deleted Connections selected by the current grant appear. A Personal Account has at most three. |
| C5 | Call while a Connection is deleting | That Connection is absent. Deleting Connections are immediately revoked and omitted. |

### D. `list_contacts`

| ID | Action | Expected |
| --- | --- | --- |
| D1 | Omit `connection_id` | Schema rejection. Except for `list_connections`, every tool requires an explicit `connection_id`. |
| D2 | Default `limit` | 20 records or fewer. |
| D3 | `limit` 1 and 50 | Both accepted. |
| D4 | `limit` 0 and 51 | Both rejected. Range is 1 through 50. |
| D5 | `search` of two characters | Rejected. A display-name prefix needs at least three characters. |
| D6 | `search` as one exact E.164 number beginning `+` | Accepted as an exact-number search. The full number is never returned or logged; only `phone_last_four` may appear. |
| D7 | `search` as a display-name prefix that matches mid-name rather than at the start | No match on the mid-name entry. The parameter is a prefix, not a substring. |
| D8 | Inspect ordering across a full page | Sorted by normalized display name, then `contact_id`. |
| D9 | Inspect `conversation_id` on a contact with known retained activity, with and without `messages:read` | Non-null only when the grant has `messages:read` **and** retained Stored Message activity exists. Null otherwise, and null does not mean the contact is inactive. |

### E. `list_groups`

| ID | Action | Expected |
| --- | --- | --- |
| E1 | Default call | `groups` entries carry `group_id`, `display_name`, `conversation_id`. No description, profile URL, or roster. |
| E2 | `search` of two characters, and of 65 | Both rejected. Range is 3 through 64 characters. |
| E3 | Inspect ordering | Sorted by normalized display name, then `group_id`. |
| E4 | Pass a returned `group_id` as `read_messages.conversation_id` | Rejected. `group_id` names a WhatsApp Recipient, not a Conversation handle. |
| E5 | For a joined group with no observed Stored Message | `conversation_id` is null. A joined group may have no Conversation yet. |
| E6 | Compare membership against the account | Only groups marked current and joined in the latest Directory projection appear, qualified by the page's `stale` and `partial`. |
| E7 | Leave a group, then re-list | The group drops out of the projection once reconciled. Note the observed lag. |

### F. `list_chats`

| ID | Action | Expected |
| --- | --- | --- |
| F1 | Default call on a Connection with both direct and group activity | Both kinds present. |
| F2 | `kind: "direct"` and `kind: "group"` | Each filters to that kind only. |
| F3 | `kind` outside the enum | Schema rejection. |
| F4 | Inspect ordering | `last_activity_at` descending, then `conversation_id`. |
| F5 | Inspect a group row | `recipient_id` is a `grp_` handle; `phone` and `phone_last_four` are always null for groups. |
| F6 | Inspect a direct row where the provider Directory lacks metadata | `display_name`, `phone`, `phone_last_four` nullable. |
| F7 | Inspect every row for excluded fields | No body, snippet, unread state, provider identifier, or roster. |
| F8 | Call on a Connection with **zero** observed Stored Messages | Success with `chats: []`, `has_more: false`, `next_cursor: null`. See defect D-1 — this currently fails. |

### G. `read_messages`

| ID | Action | Expected |
| --- | --- | --- |
| G1 | Call with a `cvs_` handle from `list_chats`, no `older_cursor` | Newest page. Records ordered oldest to newest within the page. |
| G2 | Pass a `ctc_` or `grp_` handle as `conversation_id` | Schema rejection on the `cvs_` pattern. |
| G3 | Pass a well-formed `cvs_` handle owned by another Connection | Not found, in the same shape as an unknown handle. |
| G4 | Inspect the echo fields | `conversation_id` echoes the request; `recipient_id` is the current `ctc_`/`grp_` handle; `kind` is `direct` or `group`. |
| G5 | Read a Conversation containing a Deleted Message Tombstone | `content_type: "unknown"` is permitted; `deleted: true` carries `text: null`, `media: null`. |
| G6 | Inspect `text_truncated` on every record | Always present. `text_total_utf8_bytes` is the full byte count when text exists, null otherwise. |
| G7 | Read a Conversation with an edited message | `edited_at` non-null; no prior edit content is returned. |
| G8 | Read a message with `ready` media at or under 16 MiB | `resource_uri` non-null, `resource_unavailable_reason: null`, plus one MCP `resource_link` content block. |
| G9 | Read a message with `ready` media above 16 MiB | State stays `ready`, `resource_uri: null`, reason `too_large_for_mcp`, `resource_size_limit_bytes: 16777216`. |
| G10 | Read a message whose media is `pending`, `rejected`, or `failed` | `resource_uri: null` with the matching reason. No binary bytes appear in JSON or text. |
| G11 | Read a Conversation spanning a known Ingestion Gap | The intersecting gap appears with a cause from the documented set. `ends_at` null for an active interval. |

### H. `search_messages`

| ID | Action | Expected |
| --- | --- | --- |
| H1 | Single-term query matching known retained text | Match returned, newest first by `sent_at DESC, message_id DESC`. |
| H2 | `limit` 20 and 21 | 20 accepted, 21 rejected. The limit cannot exceed 20. |
| H3 | Query producing zero normalized terms, e.g. punctuation only | Invalid. |
| H4 | Query with nine unique terms | Invalid. The cap is eight unique terms. |
| H5 | Query repeating one term | Duplicates removed; treated as one term. |
| H6 | Two-term query with the terms reversed | Same result set. Order and adjacency do not matter, but every unique term must occur. |
| H7 | Query using a prefix of a known word | No match. No substring, prefix, phrase, stemming, morphology, synonym, or fuzzy matching. |
| H8 | Query differing only by case and by an NFKC-equivalent form | Same result set. Normalization is NFKC, lowercase, NFKC. |
| H9 | Query targeting text that exists only in a prior edit, a Tombstone, a filename, or Directory metadata | No match. None of those are searched. |
| H10 | `after` later than `before` | Invalid. |
| H11 | Scope with `conversation_id` owned by another Connection | Not found. |
| H12 | Inspect every result for media fields | No Stored Media metadata, `resource_uri`, `resource_link`, provider URL, or binary content. |
| H13 | Inspect `coverage` on a Connection mid-backfill and on one with a known gap | `searchable_history_starts_at` null when nothing is searchable; `backfill_complete` true only once that boundary reaches `history_starts_at`; `partial_reasons` contains `index_backfill` and/or `ingestion_gap` consistently with `partial`. |

### I. Stored Media resources

| ID | Action | Expected |
| --- | --- | --- |
| I1 | `resources/read` on a `resource_uri` from `read_messages` | Binary `blob` with normalized MIME, `cacheScope: private`, zero TTL. |
| I2 | Same URI with an appended path segment, query string, or fragment | Rejected. The complete URI is parsed strictly. |
| I3 | Same URI with percent-encoded traversal | Rejected. |
| I4 | URI recombining a valid `message_id` with another message's `media_id` | Resource-not-found. Cross-linked handles share the not-found response. |
| I5 | URI for media that is `pending`, `rejected`, `failed`, or over the limit | The same resource-not-found response as an unknown handle. |
| I6 | URI for media on a Connection outside the grant | Same not-found response. |
| I7 | Inspect the response for any URL | No provider URL, public R2 URL, presigned URL, or fetchable HTTPS resource. |
| I8 | Read the same media twice and check the Activity Log | Each read creates an Activity Log entry and reserves the media's full verified size before decryption. |

### J. Pagination and cursor binding

| ID | Action | Expected |
| --- | --- | --- |
| J1 | Page a Directory tool to exhaustion | `has_more` false and `next_cursor` null on the final page only. |
| J2 | Reuse a `list_contacts` cursor on `list_groups` | Rejected. A cursor binds the tool. |
| J3 | Reuse a cursor with a different `connection_id` | Rejected. |
| J4 | Reuse a cursor with a different `limit` | Rejected. The cursor binds limit. |
| J5 | Reuse a cursor with a different `search`/`kind`/`direction` filter | Rejected. The cursor binds normalized filters. |
| J6 | Present a cursor issued to MCP Authorization A under Authorization B, then under an API Key grant | Rejected in both directions. Authorization IDs bind directly; API Keys bind as `api:<grant-id>`. |
| J7 | Present a REST-issued cursor to MCP | Rejected. MCP and REST cursor signing documents are separate. |
| J8 | Mutate one character of a cursor, and present an expired one | Both rejected. |

### K. Freshness and initial sync

Run these against a Connection linked within the last few minutes.

| ID | Action | Expected |
| --- | --- | --- |
| K1 | `list_contacts` and `list_groups` immediately after linking | Fields present and internally consistent. Record `as_of`, `stale`, `partial` and the wall-clock offset from link time. |
| K2 | Repeat every minute until both stabilize | Record when each tool first returns a non-empty page and when `partial` first turns false. Contacts and groups may converge at different times. |
| K3 | While a tool returns an empty page during initial sync | `partial: true`. Note whether an empty page is distinguishable from a genuinely empty account from the response alone. See defect D-3. |
| K4 | After the Connection later goes unavailable | Record `as_of`, `stale`, `partial` again. Note whether these differ from the initial-sync case. See defect D-3. |
| K5 | `search_messages` `coverage` during and after backfill | `backfill_complete` false then true; `partial_reasons` tracks `index_backfill` accordingly. |

### L. Connection lifecycle

| ID | Action | Expected |
| --- | --- | --- |
| L1 | Link a Connection and poll `list_connections` | State progresses through `connecting` to `connected`; `state_changed_at` advances with each transition. |
| L2 | Take the Connection offline | State becomes `disconnected` or `reconnect_required`. Record the delay between the Ingestion Gap's `starts_at` and the observed state change. |
| L3 | While offline, call every read tool | Directory and search tools respond from the retained projection with `stale`/`partial` set. Record which tools still answer. |
| L4 | While offline, `search_messages` `coverage.gaps` | An open interval with `ends_at: null` and cause `connection_unavailable`. |
| L5 | Restore the Connection | The gap closes with a non-null `ends_at`. Record the total open duration. |
| L6 | Confirm no new WhatsApp Connection joined an existing grant automatically | The new Connection is absent from the old grant's `list_connections`. New Connections never enter an existing grant automatically. |

### M. Outbound sends — real-world effects

**Sends real messages.** Requires `messages:send` and recipient consent. Every
invocation requires Client Confirmation, including replays and retries after a
preflight rejection.

| ID | Action | Expected |
| --- | --- | --- |
| M1 | `send_text_message` to a `recipient_id` from a Directory tool | Client Confirmation prompted. Receipt carries `send_id`, `status`, `created_at`, `status_changed_at`, `idempotent_replay: false`. |
| M2 | Inspect the receipt for leakage | No text, caption, file metadata, destination, connection handle, idempotency key, provider identifier, or Stored Message handle. |
| M3 | Supply two destinations, or none | Rejected. Exactly one of `recipient_id`, `phone`, `username`. |
| M4 | Pass a `conversation_id`, a JID, or a channel identifier as the destination | Rejected. None are accepted destinations. |
| M5 | `text` empty, whitespace-only, and 4,097 scalars | All rejected. Range is 1 to 4,096 with at least one non-`White_Space` value. |
| M6 | `text` with leading/trailing whitespace, line breaks, and a decomposed Unicode form | Delivered byte-exact. The server does not trim, normalize, or truncate. |
| M7 | `idempotency_key` not matching `^[A-Za-z0-9_-]{21}$` | Rejected. |
| M8 | Repeat the exact same key, connection, destination, and content | `idempotent_replay: true`, same `send_id`. No quota consumed, no second provider call. Client Confirmation still prompted. |
| M9 | Reuse a bound key with different content or destination | `idempotency_conflict`, `retryable: false`. |
| M10 | Send while the Connection is not `connected` | `connection_unavailable`, `retryable: true`, before quota reservation. Confirm quota is unchanged. |
| M11 | Send to a well-formed handle that is unknown, removed, unjoined, or Connection-mismatched | `recipient_not_found`, `retryable: false`, identically for all four. |
| M12 | `get_send_status` on the `send_id`, repeatedly | Status never regresses; `failed` never replaces `delivered` or `read`; `status_changed_at` reports when the returned state was established. An `unknown` operation is never retried automatically. |

`send_pdf_file` and `send_image` inherit M2 through M11. Add: filename must end
`.pdf` and carry no path separator or control character; PDF bytes must begin
`%PDF-x.y` and fall between 8 and 16,777,216 bytes; image bytes must carry a
JPEG or PNG signature and not exceed 5,000,000 bytes, with MIME derived from
bytes rather than URL metadata; URL sources must be HTTPS without credentials
or a custom port, and IP, private, and reserved targets must be rejected.

### N. Not-found and error-shape boundary

| ID | Action | Expected |
| --- | --- | --- |
| N1 | Well-formed but unknown `con_`, `cvs_`, `snd_`, `med_` handles | Constant-shape not-found per tool. |
| N2 | Well-formed handle belonging to another Personal Account | Byte-identical to N1 for that tool. Cross-tenant handles are indistinguishable from unknown ones. |
| N3 | Handle of a correct type but wrong Connection | Same not-found boundary. |
| N4 | Inspect any actionable failure | `isError: true` with `error_code`, `message`, `retryable`, and `retry_after_seconds`/`resets_at` where applicable. |
| N5 | Inspect a `retryable: true` failure that recurs identically across repeated calls over hours | `retryable` describes whether a later invocation could succeed. A permanently failing operation reporting `retryable: true` is a defect. See defect D-2. |

## Run logs

### Run 1 — 2026-09-08, freshly linked Connection

Client: Claude (MCP). Grant: one MCP Authorization, scopes `connections:read`,
`directory:read`, `messages:read`. Connections: 1. Section M not run.

Connection linked and `connected` at 2026-09-08T23:12:04Z.

| Case | Result | Observation |
| --- | --- | --- |
| C1, C3 | PASS | Fields and enum values as specified. |
| D2, D3, D4 | PASS | Limits enforced at both ends. |
| F1-F7 | BLOCKED | `list_chats` unavailable; see D-1. |
| K1 | NOTE | `list_groups` returned an empty page with `as_of` equal to the link second, `stale: true`, `partial: true`, while the account does have groups. `list_contacts` in the same minute returned a full page with a current `as_of`. |
| K2 | NOTE | Groups appeared on a later run once the Directory projection finished initial sync. Contacts converged first. |
| K3 | FAIL | An empty page during initial sync was not distinguishable from a genuinely empty account except by `partial`. See D-3. |
| G1-G11 | BLOCKED | No `cvs_` handle obtainable; see D-1 and D-4. |

### Run 2 — 2026-09-10, same Connection after 31 hours offline

Client: Claude (MCP). Same grant and scopes. Connection state `disconnected`
since 2026-09-09T03:10:50Z.

| Case | Result | Observation |
| --- | --- | --- |
| C1, C3 | PASS | State `disconnected`, `state_changed_at` 2026-09-09T03:10:50Z. |
| F1, F2, F8 | FAIL | Three calls varying `kind` (`all`, `direct`, `group`) and `limit` (1, 20, 50), all `service_unavailable`, `retryable: true`. Twenty such calls across two days. See D-1. |
| E1, E3 | PASS | Non-empty page, `has_more: true`, ordering as specified. |
| E5 | PASS | Every `conversation_id` null, consistent with zero observed Stored Messages. |
| D9 | PASS | Every `conversation_id` null under the same condition. |
| H1 | PASS | Single-term query returned an empty result set. |
| H13 | PASS | `backfill_complete: true`, `partial: true`, `partial_reasons: ["ingestion_gap"]`. |
| L4 | PASS | One open gap, `starts_at` 2026-09-09T03:05:50Z, `ends_at: null`, cause `connection_unavailable`. |
| K4 | NOTE | Both Directory tools reported `stale: true, partial: true` with `as_of` frozen at 2026-09-09T03:00:53Z and 03:00:55Z — roughly five minutes before the gap opened, and about two seconds apart. |
| G1-G11 | BLOCKED | See D-4. |
| N5 | FAIL | See D-2. |

## Open defects

**D-1 — `list_chats` returns `service_unavailable` on every call.** Twenty
calls across two days and two Connection states, every `kind` value, and every
`limit` from 1 to 50. Latencies of 67-150 ms are faster than sibling tools that
succeed, so a provider timeout is ruled out; `list_contacts` returned live
roster data seconds either side of a failure. Every failing call shared one
condition that was never varied: the Connection had zero observed Stored
Messages, and therefore zero WhatsApp Conversations. Case F8 covers that state
and is the reproduction to try first.

**D-2 — `retryable: true` on a deterministic failure.** D-1 has never once
succeeded, yet reports `retryable: true`. Per the contract, `retryable`
describes whether a later invocation could succeed, so clients are being
instructed to spend retries and daily quota on an operation that cannot.

**D-3 — `stale` and `partial` carry two incompatible meanings.** During initial
sync they mean "not ready yet, wait"; on an unavailable Connection they mean
"the source cannot be confirmed", which waiting never resolves. Both present
identically, as does `conversation_id: null`, which covers both "no retained
activity" and "sync has not reached this entry". A client cannot distinguish
"not ready" from "empty" from "stalled" from the response alone. Cases K3 and
K4 record it.

**D-4 — `read_messages` is unreachable when `list_chats` is down.** The contract
names three sources of a `cvs_` handle: `list_contacts.conversation_id`,
`list_groups.conversation_id`, and `list_chats`. The first two are null until
retained Stored Message activity exists, so on a Connection with none, D-1
removes the only remaining source and `read_messages` has no valid input at all.
Section G is unrunnable in that state. This is a consequence of D-1, not a
separate fault, but it changes its severity: the failure removes two of six read
tools, not one.
