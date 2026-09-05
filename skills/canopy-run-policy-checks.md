---
name: canopy-run-policy-checks
description: >-
  Verify a consumer's coverage against your own requirements with Canopy Connect
  Policy Checks - configure the team-wide rules, evaluate them against a Pull,
  and produce the PDF and policy forms that evidence the result.
generated: '2026-09-05'
method: generated
source: >-
  openapi/canopy-openapi.json (operationIds verified against the spec) +
  https://docs.usecanopy.com/reference/get-policy-check-team-settings
api: Canopy Connect API
base_url: https://app.usecanopy.com/api/v1.0.0
operations:
  - get-policy-check-team-settings
  - post-policy-check-team-settings
  - post-pull-policy-check
  - get-pull-policy-check-pdf
  - download-policy-form-by-id
  - get-pull-by-id
  - post-policy-search
---

# Run a Policy Check with Canopy Connect

Policy Checks are the compliance half of Canopy Connect: given a Pull, decide
whether the coverage meets the limits and deductibles you require.

## Steps

1. **Read the current configuration before you touch it.** Call
   `get-policy-check-team-settings`. **Do this first every time** -
   `post-policy-check-team-settings` overwrites team-wide settings in place, and
   Canopy keeps no history and offers no restore. The response you save is your
   only rollback.
2. **Configure the checks.** Call `post-policy-check-team-settings`. Every
   configured check runs on every submission - there is no per-request
   selection. The dwelling and vehicle rule shapes are
   `PolicyCheckDwellingSetting`, `PolicyCheckVehicleSetting` and
   `PolicyCheckPullVehicleSetting` in the spec.
3. **Evaluate a Pull.** Call `post-pull-policy-check` with the `pull_id`. A Pull
   that has not succeeded returns `400 PULL_NOT_SUCCESS`; a Pull type that does
   not support checks returns `400 PULL_NOT_SUPPORTED`.
4. **Produce the evidence.** `get-pull-policy-check-pdf` returns the Policy
   Check PDF. For the underlying carrier forms, use
   `download-policy-form-by-id`, which `302`s to a short-lived signed URL and
   returns the same URL in the JSON body. Policy forms require the feature on
   your plan. Do **not** use `get-policy-form-by-id` - it is deprecated in
   favour of `download-policy-form-by-id`.
5. **No credentials to hand?** `post-policy-search` runs a credential-less
   policy search (auto, home, or auto ID card). Treat every call as billable and
   irreversible - there is no cancel and no void.

## Testing

Use the published sandbox users tuned for this flow (see
`sandbox/canopy-sandbox.yml`):

- `user_good_auto_compliant` / `pass_good` - limits high, deductibles low; the
  passing case.
- `user_good_auto_noncompliant` / `pass_good` - limits low, deductibles high;
  the failing case.
- `user_good_underinsured` / `pass_good` - higher home Coverage A limit.
- `user_good_diffs` / `pass_good` - stable across pulls, so a check result does
  not drift between runs.

## Rules that will bite you

- **Settings are team-wide and overwrite-in-place.** There is no per-widget
  scoping and no versioning.
- **All configured checks always run.** You cannot request a subset.
- **Both scopes are needed for evaluation.** Under an OAuth App token,
  `post-pull-policy-check` requires `read:pulls`, `read:policy_checks` **and**
  `write:policy_checks`.
