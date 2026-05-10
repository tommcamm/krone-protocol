# krone-protocol

Wire-format contract between Krone's Android client (`krone`) and its Rust relay backend (`krone-groups-server`). Consumed by both repos as a git submodule at `protocol/`.

**License:** GPL-3.0-or-later (matches client + server).

## Layout

- `schemas/` — JSON Schema (draft 2020-12) for every request/response body.
- `vectors/signing_vectors.json` — canonical Ed25519 signing vectors (see [Test vectors](#test-vectors)). Both sides must reproduce these byte-for-byte.
- `VERSION` — protocol semver. Used in `GET /server-info` responses.

## Compatibility

- Additive changes (new optional fields) bump PATCH.
- New required fields / renames / removed fields bump MAJOR and ship under a new path prefix (e.g. `/v2/envelopes`). MVP is unversioned (`/envelopes`, `/server-info`, …) and implicitly `/v1`.
- Clients MUST tolerate unknown fields on responses.
- Servers MUST reject unknown fields on requests (strict) to catch client bugs early.

## Signed requests

All client → server requests except `GET /server-info` and `GET /healthz` are signed.

Headers:

| Header | Value |
|---|---|
| `x-krone-device-id` | 16-byte device id, lowercase hex (32 chars) |
| `x-krone-timestamp` | unix seconds, decimal string |
| `x-krone-signature` | Ed25519 signature, base64 (standard, padded) |

Signing input (concatenation, no separators beyond the fixed header labels):

```
"krone-req-v1\n"  ||
timestamp_decimal || "\n" ||
device_id_hex_lower || "\n" ||
method_upper || "\n" ||
path_with_query || "\n" ||
sha256(body_bytes)  // 32-byte digest, raw
```

Server verifies:
1. `|now - timestamp| <= clock_skew_seconds` (default 120).
2. Device's identity public key is on file (except on `POST /devices`, where the pubkey is in the body and used directly).
3. Ed25519 signature over the input above.

Server response includes `x-server-signature` (base64 Ed25519) over:

```
"krone-res-v1\n"  ||
request_id || "\n" ||
status_code_decimal || "\n" ||
sha256(response_body)
```

`request_id` is the value of the request's `x-request-id` header (or a server-generated ULID if none).

## Test vectors

`vectors/signing_vectors.json` pins the canonical byte-level output of both signing formats. Every implementation (Rust server, Android client, any future client) MUST reproduce, from the inputs in each vector:

1. `body_sha256_hex` — SHA-256 of the raw body bytes.
2. `signing_input_hex` — the exact bytes that go into Ed25519 (tag, separators, body hash — no padding, no length prefixes).
3. `signature_b64` — the Ed25519 signature produced by the referenced key, base64-standard with padding.

These three layers are independently checkable, which lets an implementor localize a mismatch: if body-hash matches but signing-input doesn't, the canonicalization is wrong; if signing-input matches but signature doesn't, the Ed25519 binding is wrong.

Vectors are regenerated with:

```
cargo run --example gen_vectors > protocol/vectors/signing_vectors.json
```

from inside `krone-groups-server`. Seeds are fixed (see the top of `examples/gen_vectors.rs`), so output is deterministic — any diff to this file on a PR means the wire format changed and all implementations must be audited.

## Endpoints

See `schemas/` for exact shapes.

- `GET /server-info` — unsigned; returns `server_info_response`.
- `GET /healthz` — unsigned; returns `{status: "ok"}`.
- `POST /devices` — signed (pubkey in body); request `device_registration_request`, response `device_registration_response`.
- `DELETE /devices/self` — signed, empty body; response `{status: "ok"}`.
- `POST /envelopes` — signed; request `envelope_submit_request`, response `envelope_submit_response`; each `recipient_device_id` must already be registered.
- `GET /envelopes/inbox?since=<cursor>&limit=<n>` — signed, empty body; response `inbox_response`.
- `POST /envelopes/ack` — signed; request `ack_request`, response `{acknowledged: n}`.

All error responses share `error_response`.
