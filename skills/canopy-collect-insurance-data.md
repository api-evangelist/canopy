---
name: canopy-collect-insurance-data
description: >-
  Collect a consumer's property & casualty insurance data through Canopy
  Connect - obtain consent, authenticate against the carrier, handle MFA, and
  retrieve the structured Pull once the carrier data lands.
generated: '2026-09-05'
method: generated
source: >-
  openapi/canopy-openapi.json (operationIds verified against the spec) +
  https://docs.usecanopy.com/reference/getting-started +
  https://docs.usecanopy.com/reference/about-webhooks
api: Canopy Connect API
base_url: https://app.usecanopy.com/api/v1.0.0
operations:
  - get-tos
  - post-consent
  - get-carriers
  - get-carrier
  - post-consent-and-connect
  - post-connect
  - post-idvoptions
  - post-idv
  - post-webhooks
  - get-pull-by-id
  - download-document-by-id
---

# Collect insurance data with Canopy Connect

## Before you start

- Base URL is `https://app.usecanopy.com/api/v1.0.0`. HTTPS with TLS 1.2+ only.
- Authenticate with HTTP Basic: Client ID as username, Client Secret as password
  (`Authorization: Basic <base64(client_id:client_secret)>`).
- Use a **sandbox** key against a **sandbox** link. Mixing key type and link type
  returns `400 INCORRECT_API_KEY_TYPE`.
- Test with the published sandbox users in `sandbox/canopy-sandbox.yml`
  (`user_good` / `pass_good` is the plain success case; `user_mfa` / `pass_good`
  exercises the MFA branch).

## Steps

1. **Register a webhook first.** Call `post-webhooks` with `hookUrl` and
   `eventTypes`. The Pull flow is asynchronous - if you skip this you will be
   polling. Note the API accepts only `COMPLETE`, `POLICIES_AVAILABLE`,
   `POLICY_AVAILABLE`, `AUTH_STATUS` and `VALIDATION_COMPLETE` as subscribable
   types; the other documented events are dashboard-only.
2. **Show the consumer the current terms.** Call `get-tos` to retrieve the live
   terms version number and HTML, or link the provider's privacy page. You must
   capture consent before you may connect.
3. **Pick a carrier.** Call `get-carriers` for the supported list (it carries UI
   attributes), or `get-carrier` for one by id. An unsupported choice returns
   `400 UNSUPPORTED_CARRIER` / `400 UNSUPPORTED_INSURER`.
4. **Consent and connect.** Call `post-consent-and-connect` with the consent
   record and the tokenized carrier credentials. If you are using Canopy Connect
   Components, the credential tokens come from `form.submit()` in the browser
   and the raw username/password never reach your server. This creates the Pull.
   Call `post-consent` on its own only when you are pre-collecting consent for a
   white-label flow.
5. **Handle identity verification.** Watch the `AUTH_STATUS` webhook.
   - Status `IDENTITY_VERIFICATION_OPTIONS` - call `post-idvoptions` with the
     consumer's chosen MFA channel.
   - Status `IDENTITY_VERIFICATION` - call `post-idv` with the code the consumer
     received.
   - Status `NOT_AUTHENTICATED` - the credentials failed; call `post-connect` to
     re-authenticate the same Pull. Read `login_error_message` for the carrier's
     own reason.
6. **Wait for the data.** `POLICIES_AVAILABLE` means every policy is parsed but
   documents are not ready. `COMPLETE` means everything, documents included, is
   available. `ERROR` means the Pull failed - inspect the status
   (`PROVIDER_ERROR` vs `INTERNAL_ERROR`).
7. **Read the Pull.** Call `get-pull-by-id` with the `pull_id` from the webhook
   body. This returns the full structured record: policies, drivers, vehicles,
   dwellings, coverages, premiums, claims and documents.
8. **Fetch documents.** Call `download-document-by-id` using the `download_url`
   on each Document. It responds `302` to a short-lived signed URL and also
   returns that URL in a JSON body, so you can read the body instead of
   following the redirect. Do **not** use `get-document-by-id` - it is
   deprecated in favour of this operation.

## Rules that will bite you

- **No idempotency key exists.** There is no `Idempotency-Key` header on any
  mutating operation. If `post-consent-and-connect` times out, retrying may
  create a second Pull against the consumer's carrier account. Correlate with
  your own `pullMetaData` and check `get-pulls` (filtered with `since`) before
  you retry.
- **A Pull cannot be undone.** There is no cancel and no delete. The only
  reversal available is `patch-pull-by-id` with `is_archived: true`.
- **No webhook signature is published.** Canopy documents no signing secret,
  timestamp header or replay window, so you cannot cryptographically verify a
  callback. Treat the webhook as a trigger only, and re-read the record with
  `get-pull-by-id` rather than trusting the payload.
- **No rate limits are published.** No 429 is declared and no `RateLimit-*` or
  `Retry-After` header is returned. Rate-limit yourself conservatively.
- **Errors are one field.** The envelope is `{"error": "<CODE>"}` with no
  message and no request id. The 48 codes are catalogued in
  `errors/canopy-problem-types.yml`.
