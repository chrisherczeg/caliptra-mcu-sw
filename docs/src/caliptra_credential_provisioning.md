# Caliptra Authorization Credential Provisioning

**Status:** Discussion draft

This document captures the proposed credential-provisioning sequence for team
discussion. It is based on:

- [DSP0289 1.0, section 8.6, Initial provisioning](https://www.dmtf.org/sites/default/files/standards/documents/DSP0289_1.0.0.pdf#page=30)
- [caliptra-mcu-sw PR #906, WIP: SPDM authorization with hybrid OTP+Flash storage](https://github.com/chipsalliance/caliptra-mcu-sw/pull/906)

PR #906 assigns the Credential IDs as follows:

| Credential ID | Role |
| ---: | --- |
| `0` | Recovery |
| `1` | Vendor |
| `2` | Owner |
| `3` | Tenant |

## 1. Vendor manufacturing - recovery credential

- Vendor provisions the Credential ID `0` public-key digest into OTP.
- Firmware supplies the full public key, fixed attributes, and recovery policy.
- Credential ID `0` is locked.
- Device enters DSP0289 `DefaultState`; IDs `1`-`3` are unprovisioned.

**Open question:** How should Caliptra represent DSP0289 `DefaultState` after
entering Production? Should the absence of `CRED_BLOB` indicate
`DefaultState`, or should Caliptra require a valid authenticated `CRED_BLOB`
that explicitly records `DefaultState`? If absence indicates `DefaultState`,
how does Caliptra distinguish a new or legitimately reset device from an
`Owned` device whose blob was erased, lost, or corrupted?

## 2. Vendor supply-chain credential

- In a Vendor-controlled facility, Vendor provisions:
  - Vendor policy for Credential ID `1`.
  - Vendor credential ID `1`.
- Vendor verifies and locks Credential ID `1`.
- Device remains in `DefaultState`.

**Gaps:**

- PR #906 currently defines Credential ID `1` as non-lockable, so the proposed
  locking step is not possible.
- Vendor policy and permitted Production operations are not fully defined.

## 3. Vendor-to-Owner custody transfer

- Device leaves Vendor custody with:
  - Recovery Credential ID `0` locked.
  - Vendor Credential ID `1` locked.
  - Owner Credential ID `2` unprovisioned.
  - Tenant Credential ID `3` unprovisioned.
- Device remains in `DefaultState`.

**Open question:** How is access to the unauthenticated initial-provisioning
commands prevented during shipment and custody transfer?

## 4. Owner onboarding

- In an Owner-controlled environment, Owner verifies Vendor Credential ID `1`
  and its policy.
- Owner provisions:
  - Owner policy for Credential ID `2`.
  - Owner credential ID `2`.
- Owner verifies all provisioned credentials and policies.

**Decision needed:** Should Owner provision Tenant Credential ID `3` during
this onboarding stage, or provision it through an authorized operation after
ownership?

## 5. Take ownership

- Owner authenticates using Credential ID `2`.
- Owner sends `TAKE_OWNERSHIP`.
- Device transitions from `DefaultState` to `Owned`.
- Authorization policies are enforced for protected operations.
- `TAKE_OWNERSHIP` does not automatically lock Credential IDs `2` or `3`.

**Open questions:**

- How is `Owned` persisted?
- What operation does `TAKE_OWNERSHIP` perform on `CRED_BLOB`, if any?

## 6. Tenant onboarding, if deferred

- Owner authorizes provisioning of Tenant Credential ID `3`.
- Owner assigns the Tenant policy.
- Tenant cannot grant itself privileges.

**Gaps:**

- PR #906 does not define the Tenant provisioning stage.
- The Owner administrative policy needed to provision and manage Tenant
  credentials is not defined.

## 7. Normal Production operation

- Vendor uses Credential ID `1`.
- Owner uses Credential ID `2`.
- Tenant uses Credential ID `3`.
- Each credential can perform only operations allowed by its policy.

**Open question:** What is the complete operation-to-role policy for Vendor,
Owner, and Tenant?

## 8. Recovery

- If operational credentials are lost or corrupt, the device exposes recovery
  Credential ID `0`.
- Recovery authority authenticates `AUTH_RESET_TO_DEFAULT`.
- Device returns to `DefaultState`.
- Operational credentials and policies are reprovisioned.
- Owner calls `TAKE_OWNERSHIP` again.

**Open questions:**

- How is Recovery state persisted?
- What exactly does `AUTH_RESET_TO_DEFAULT` do to `CRED_BLOB`?
- How are locked supply-chain credentials preserved when DSP0289 resets only
  unlocked credentials and policies?
