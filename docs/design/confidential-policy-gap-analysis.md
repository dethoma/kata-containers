# Confidential Container Policy Gap Analysis

## Purpose

This note captures the current discussion about extending Kata Containers and
Confidential Containers with hcsshim-style confidential policy capabilities,
especially signed policy fragments.

The recommended direction is to extend the existing CoCo/Kata policy,
init-data, attestation, and KBS architecture rather than replace it.

## Existing foundation

Kata and Confidential Containers already provide:

- Rego policy enforcement at Kata agent RPC boundaries.
- `genpolicy` workload-specific policy generation.
- CoCo init-data carrying `policy.rego`, Attestation Agent configuration, and
  Confidential Data Hub configuration.
- Host-side init-data hashing and TEE launch integration.
- Guest-side init-data parsing and Attestation Agent integration.
- KBS/CDH attestation-gated secret release.
- A documented trust model that binds policy integrity to TEE evidence.

The major missing capability is therefore not basic policy enforcement. It is
the policy supply chain: signed, composable, versioned, updateable policy
artifacts with explicit provenance and rollback semantics.

## Main gaps relative to hcsshim

### Base policy lifecycle

The current policy path supports delivery of a complete Rego document and
policy replacement through agent APIs. Confidential operation needs an
explicit immutable, one-shot base-policy state, with replacement restricted to
development-only modes.

### Canonical identity and attestation binding

Init-data is measured and passed into confidential VM launch configuration, but
the policy identity contract needs to be made explicit and consistent. A
canonical policy or policy-manifest digest should be exposed to the attestation
and KBS authorization paths.

### Signed fragments

Kata has no native fragment protocol comparable to hcsshim. The missing
protocol should define:

- COSE-signed fragment envelopes.
- Media type and framework/API compatibility.
- Issuer and feed identity.
- Signer trust roots or DID/certificate validation.
- Minimum security version and downgrade prevention.
- Namespace, include, and parameter semantics.

### Transactional composition and updates

Fragments should be fully verified before policy state changes. Applying a
fragment must be transactional, with duplicate detection, replay protection,
rollback on composition failure, and a stable resulting policy digest.

### Transparency and provenance

Transparency trust lists and receipt validation are optional enhancements, but
they are part of the hcsshim policy supply-chain model and should be supported
by the fragment format rather than added as an incompatible extension later.

### Enforcement coverage

Kata's RPC policy coverage is strong, but a complete confidential policy audit
must also cover storage and mounts, overlays, devices, external processes,
signals, logging, shutdown, verified images, and other state-changing
operations. The policy must constrain semantics, not only RPC names.

### Runtime parity

The legacy Go runtime, runtime-rs, and shim paths need one protocol and
consistent behavior for policy manifests and fragment transport.

### Tooling and distribution

`genpolicy` currently produces a monolithic workload policy. It should evolve
to generate a base policy, policy manifest, canonical digest, fragment
metadata, and signing/verification inputs while preserving compatibility with
existing init-data and annotation workflows.

## Recommended architecture

Use three layers:

1. **Kata agent policy engine**: enforcement, policy state, fragment
   composition, and request authorization.
2. **CoCo policy and attestation layer**: init-data measurement, TEE evidence,
   KBS authorization, signer trust, and policy identity.
3. **Runtime transport**: common protocol implemented by the Go runtime,
   runtime-rs, and shims. The runtime transports policy artifacts but is not
   the policy authority.

Cryptographic verification primitives should be reused from a shared
Confidential Containers policy component where practical. Kata should retain
control of agent-specific enforcement and Rego composition.

## Proposed implementation sequence

1. **Document and test the existing contract.** Define exactly what is hashed,
   measured, verified, and consumed by KBS authorization.
2. **Harden base policy state.** Make the base policy immutable after
   initialization and define a canonical digest.
3. **Add a policy manifest.** Record base digest, fragment set, versions,
   signer identities, and compatibility metadata.
4. **Add signed fragments.** Implement COSE verification and trust-policy
   checks, preferably using shared CoCo cryptographic code.
5. **Add transactional updates.** Support rollback, replay protection,
   duplicate detection, and downgrade prevention.
6. **Bind KBS release decisions.** Authorize secrets against the policy
   manifest or its digest.
7. **Extend `genpolicy`.** Generate manifests and fragment-ready artifacts
   without breaking existing policy deployment.
8. **Audit enforcement semantics.** Prioritize exec, signals, mounts,
   storage, device access, logging, and lifecycle operations.
9. **Add end-to-end confidential tests.** Cover invalid signatures, wrong
   signers, SVN downgrade, namespace collisions, replay, rollback, digest
   mismatch, and KBS denial.

## Design principle

Reuse CoCo's existing init-data, TEE measurement, Attestation Agent, CDH, and
KBS work. Add hcsshim-like signed fragment lifecycle and policy composition
around Kata's existing agent enforcement engine. This turns the effort from a
wholesale policy rewrite into a focused integration and hardening project.
