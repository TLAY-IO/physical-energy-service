# TLAY Physical Energy Service

Developer Integration Guide | 0.2.0-review

[OpenAPI specification](./openapi.yaml) · [API reference PDF](./TLAY_Physical_Energy_Service_API_Reference.pdf)

## Overview

Purchase a fixed-duration power session from an eCandle device at an authorized site. This guide describes how an agent discovers an offer, pays in USDC, tracks physical execution and retrieves a delivery receipt.

Integration review | v0.2.0-review | 27 September 2026. This API contract is proposed and has not been validated against a deployed backend. Deployment configuration and the signature/proof profiles must be supplied before live use.

| Component | Responsibility |
| --- | --- |
| BoAT | Device identity and device-side signing |
| HashAnchor | Portable verification records linking machine events and delivery |
| TLAY service | Offers, orders, execution tracking and receipt access |
| Circle Agent Stack | Intended service discovery and x402 payment integration |

## Before you start

Obtain the operator-provided API base URL, an access grant for the designated site/output, and an x402 client configured for the supported payment rail. Set a user-approved spending limit. The review package includes no live server address or fixed price.

The access grant authorizes use of a physical output; it is separate from payment. Public offer discovery is open. Status and receipt access use a session-scoped token returned after purchase. No separate API subscription is part of this design.

## 1. Discover an offer

```
curl "$TLAY_BASE_URL/v1/energy/offers"
```

Select an available offer and inspect device_id, site_id, output_id, duration_seconds, max_power_w, price_usdc and terms_version. price_usdc is a decimal string with six fractional digits. A timed power session specifies a maximum load and duration; it does not promise a fixed energy quantity.

## 2. Reserve a quote

```
curl -X POST "$TLAY_BASE_URL/v1/energy/quotes" \
  -H "Content-Type: application/json" \
  -d '{"offer_id":"offer_demo","client_nonce":"buyer_nonce_demo",
       "access_grant":"operator_issued_grant"}'
```

Examples use illustrative IDs and credentials. Use the returned quote_id and preserve the nonce. Review expires_at, start_by, the frozen offer and refund_policy before paying. If the quote expires before payment, obtain a new one. A quote must reserve the authorized capacity until expiry.

## 3. Purchase the session

```
curl -i -X POST "$TLAY_BASE_URL/v1/energy/sessions" \
  -H "Content-Type: application/json" \
  -H "Idempotency-Key: unique-purchase-key-demo" \
  -d '{"quote_id":"quote_demo","client_nonce":"buyer_nonce_demo"}'
```

An unpaid request returns HTTP 402 and PAYMENT-REQUIRED. Use the configured x402 client to inspect the requirements, obtain spending authorization, and retry the same body and idempotency key with PAYMENT-SIGNATURE. The SDK handles the payment payload; this guide does not substitute a hand-written signature for that protocol.

A 202 response contains session and access_token. Store the token securely. Acceptance means the order is recorded and can enter execution; it does not mean the power session has finished. The selected SDK and facilitator must support authenticated recovery of a lost response without a second charge; this is an integration acceptance requirement.

```
{
  "session": {
    "session_id": "session_demo",
    "quote_id": "quote_demo",
    "device_id": "ecandle_demo",
    "state": "accepted",
    "payment_state": "accepted",
    "created_at": "2026-09-27T10:00:00Z",
    "delivered_seconds": 0,
    "payment_reference": "payment_demo"
  },
  "access_token": "session_scoped_token_demo"
}
```

## 4. Track execution

```
curl "$TLAY_BASE_URL/v1/energy/sessions/$SESSION_ID" \
  -H "Authorization: Bearer $SESSION_TOKEN"
```

| State | Meaning / action |
| --- | --- |
| accepted | Order recorded; await device dispatch. |
| starting | Device start requested; await acknowledgement. |
| running | Power session in progress. |
| completed | Requested session finished; retrieve receipt. |
| partially_completed | Some service delivered; inspect receipt and refund policy. |
| failed | Execution failed; inspect failure_code and payment_state. |

Payment and proof states are independent of execution. For example, a physical session may be completed while settlement remains pending. Do not treat a batch settlement reference as proof of physical delivery.

## 5. Retrieve and verify the receipt

```
curl "$TLAY_BASE_URL/v1/energy/sessions/$SESSION_ID/receipt" \
  -H "Authorization: Bearer $SESSION_TOKEN"
```

HTTP 202 returns the session while execution is active. HTTP 200 returns a terminal receipt, including partial or failed outcomes. Retrieval is included in the purchase. proof_state=ready requires a proof_bundle; pending means verification material is not yet complete.

| Check | How to interpret it |
| --- | --- |
| Order binding | Match session_id, quote_id, client_nonce and request_digest to the purchase. |
| Device signature | Decode the exact payload_b64 bytes; check the digest, trusted key, key validity and allowlisted signature profile. |
| Measured delivery | Decode the signed payload and match device, session, time, sequence and measurements to the receipt. |
| HashAnchor proof | Use the published profile and verifier version to verify the artifact and its references. |
| Payment record | Reconcile payment_reference; use settlement_tx_hash only when actually available. |

The exact signing payload, algorithm, public-key encoding and HashAnchor proof profile remain deployment requirements. A valid signature supports origin and integrity; sensor calibration and physical delivery acceptance are separate checks. Reject unknown profiles rather than treating them as verified.

## Retries, errors and refunds

| Condition | Recommended client behavior |
| --- | --- |
| 400 | Correct malformed parameters; do not repeat unchanged. |
| 401 / 403 | Check session token, physical access grant or payment authorization. |
| 402 | Inspect requirements and use the configured payment client. |
| 409 | Check quote expiry or idempotency conflict; reconcile any accepted payment first. |
| 429 | Back off; retain the same purchase identity. |
| 503 / connection lost | Recover the existing order; do not automatically pay again. |

Keep the same request and idempotency key for a retry. The service must authenticate the original buyer before returning a saved order or token. Knowing an idempotency key alone is not authorization. After receiving the token, use read endpoints to track the existing purchase.

The refund rule is disclosed in the quote. Failed or partial execution follows that rule and is reflected in payment_state. A refund is a separate recorded payment; x402 does not by itself guarantee automatic refunds. Operator contact and response times must be published with the deployment.

## API map and downloads

| Method | Path | Charge |
| --- | --- | --- |
| GET | /v1/energy/offers | Included / free |
| POST | /v1/energy/quotes | Included / free |
| POST | /v1/energy/sessions | Paid |
| GET | /v1/energy/sessions/{session_id} | Included / free |
| GET | /v1/energy/sessions/{session_id}/receipt | Included / free |
| GET | /v1/devices/{device_id}/keys/{key_id} | Included / free |
| GET | /healthz | Included / free |

The contract has one paid endpoint, five supporting business/identity endpoints and one health endpoint. Download openapi.yaml for the machine-readable definition and the API Reference PDF for the uploadable technical specification.

## Integration readiness

Before live use, confirm the base URL, network/asset/seller configuration, quote prices, key and proof profiles, refund process, and one independently reproduced payment-to-delivery flow. Test duplicate requests, expired quotes, device outage, partial delivery and lost responses. OpenAPI structural validity is not a live integration test.

## Background and support

TLAY has demonstrated machine payments using real hardware. Arc has published an eCandle/Bitaxe partner spotlight; TLAY also presented a machine-payment demonstration at the September 17 Arc mainnet celebration drone show. These demonstrations provide the background for this proposed marketplace service.

- [TLAY product documentation](https://www.tlay.io/docs/)
- [Arc partner spotlight](https://www.arc.io/blog/how-tlay-is-building-the-payment-layer-for-the-machine-economy)
- [Arc demonstration video](https://x.com/arc/status/2102155914484576408)
- [Circle seller integration guide](https://developers.circle.com/agent-stack/agent-marketplace/become-a-seller)
