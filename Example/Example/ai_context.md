# AI CONTEXT – Stripe Terminal + Raspberry Pi Vending Machine

This document exists to give any AI coding assistant (including ChatGPT plugins in Xcode) full project context without relying on chat history.

If you are an AI reading this: assume this context is authoritative and up‑to‑date.

---

## 1. High‑Level Goal

Build a card‑only, unattended vending machine controlled by a Raspberry Pi, using:

- Stripe Terminal (M2 Bluetooth reader)
- Custom iOS app (Swift / SwiftUI) as the client
- Node.js backend running on the Pi
- Physical shelves driven by relays / MOSFETs / I2C expanders

Primary flow:

1. iOS app connects to Stripe M2 reader
2. Customer pays
3. Raspberry Pi actuates shelf motor (vend)
4. Payment is captured only if vend succeeds

---

## 2. Hardware Architecture

### Core Components
- Raspberry Pi (hostname: visionpi, static LAN IP usually 192.168.0.134)
- Stripe Terminal M2 (Bluetooth)
- Shelves / Motors driven via:
  - Relay boards and/or MOSFET driver boards
  - MCP23017 I2C GPIO expanders
- Optional MDB hardware exists, but current flow does NOT depend on MDB for Stripe

### Vend Control Model
- Each shelf is addressed by either:
  - Relay channel index, or
  - MCP23017 port/pin
- Vend action = energize motor for X ms
- Vend confirmation may be time‑based or sensor‑based (beam break planned)

---

## 3. Network & Runtime Environment

### Raspberry Pi Backend
- OS: Raspberry Pi OS (Linux)
- Runtime: Node.js
- Backend listens on:
  - http://0.0.0.0:4242 (Stripe backend)
  - Additional vend API may run on :8000

### Required Environment Variables (Pi)

export STRIPE_SECRET_KEY=sk_test_...

Stripe keys are never embedded in the iOS app.

---

## 4. Stripe Terminal Architecture (IMPORTANT)

### Stripe Mode
- Currently operating in TEST MODE
- All Stripe objects (Locations, Readers, PaymentIntents) must be test‑mode

### Terminal Location
- Stripe Terminal Location is REQUIRED
- First Bluetooth connection must explicitly pass locationId (tml_...)
- Error "no last registered location" means location was not supplied

Once a reader is connected with a location, Stripe automatically registers it.

### Reader Type
- M2 Bluetooth reader
- Registered via SDK connection, not dashboard pre‑registration

---

## 5. Backend Endpoints (Node.js on Pi)

### Stripe Connection Token

POST /connection_token

Returns:
{ "secret": "pst_test_..." }

Used by the iOS app to initialize Stripe Terminal SDK.

Note: Stripe connection tokens can optionally be scoped to a Terminal Location.

---

### Create PaymentIntent (manual capture)

POST /create_payment_intent

Request:
{ "amount": 500, "currency": "usd" }

Response:
{ "client_secret": "pi_..._secret_...", "payment_intent_id": "pi_..." }

Rules:
- Use test mode keys during development
- Set capture_method = manual so we can vend first, then capture

---

### Capture PaymentIntent

POST /capture_payment_intent

Request:
{ "payment_intent_id": "pi_..." }

Response:
{ "status": "succeeded" }

---

### Cancel PaymentIntent

POST /cancel_payment_intent

Request:
{ "payment_intent_id": "pi_..." }

Response:
{ "status": "canceled" }

---

### Vend Endpoint (Pi Control)

POST /vend

Example payload:
{
  "shelfId": 3,
  "durationMs": 250
}

Responsibilities:
- Activate shelf motor
- (Optional) verify success via sensor
- Return success / failure

---

## 6. iOS App Responsibilities

- Discover Bluetooth readers
- Connect reader with locationId
- Collect payment method
- Confirm payment intent
- Call Pi /vend endpoint on success
- Notify backend to capture or cancel payment

### ATS / Networking
- App allows HTTP calls to LAN IP (ATS relaxed for development)

---

## 7. Canonical Payment → Vend Flow

1. iOS app requests /connection_token
2. Terminal SDK connects to M2 (with locationId)
3. Backend creates PaymentIntent (manual capture)
4. iOS collects & confirms payment
5. iOS calls Pi /vend
6. Vend succeeds → backend captures payment
7. Vend fails → backend cancels / refunds

---

## 8. Known Gotchas (Already Solved)

- Bluetooth discovery alone ≠ registered reader
- Missing locationId blocks M2 connection
- Expired Stripe keys crash backend
- Test vs Live mismatch causes silent failures

---

## 9. AI Assistant Guidance

When helping with this project:
- Prefer clear architecture over abstractions
- Avoid inventing new payment flows
- Assume hardware control happens on the Pi, not iOS
- Stripe Terminal SDK is authoritative for reader behavior
- Keep solutions practical and debuggable

If context is missing, update this file instead of guessing.

---

## 10. Status

- Backend running on Pi
- M2 connects once locationId is provided
- PaymentIntent + vend capture flow in progress
- Shelf automation expanding
