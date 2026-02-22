# SNARC AXIOM — Security Policy and Responsible Deployment

> **License**: [CC BY-NC 4.0](../LICENSE) — Non-commercial use only. Commercial use requires a license. See [COMMERCIAL_LICENSE.md](../COMMERCIAL_LICENSE.md).


This document specifies the deployment tier policy, threat escalation
procedures, value cap enforcement, and migration triggers. It is a
normative companion to Section 11 of the paper.

---

## 1. Deployment Tiers

| Tier | Name | Proof system | Committee | Real value? |
|------|------|-------------|-----------|-------------|
| 0 | **Researchnet** | Groth16 | Fixed test set | ❌ Never |
| 1 | **Testnet** | PLONK / Halo2 | PoW-lite + stake | ❌ Test tokens only |
| 2 | **Mainnet** | STARKs + PQ TSig | Full sortition | ✅ Post-audit only |

**Tier 0 (this repository)**: Academic paper and specification.
No network. No issuance. No value.

**Tier 1**: Requires completion of at least two independent security audits
and an active bug bounty before any value-bearing token launch.
Value caps are enforced at the protocol level (see §3).

**Tier 2**: Requires all Tier 1 conditions plus:
- Formal verification of `Apply(S, B)` (Coq or Lean)
- Hardware wallet and side-channel audit
- Post-quantum cryptographic components (see §4)
- Jurisdiction-specific legal and regulatory review

---

## 2. Threat Escalation and Freeze Policy

### Level 1 — Yellow (Monitor)

**Trigger**: published theoretical attack, unconfirmed anomaly, or unusual
on-chain pattern detected by monitoring.

**Response**:
- Increase audit frequency
- Convene security committee within 24 hours
- Prepare freeze parameters
- No operational change

### Level 2 — Orange (Restrict)

**Trigger**: credible proof-of-concept exploit reported, or anomalous
nullifier collision rate / state growth exceeding 3σ baseline.

**Response**:
1. Halt `MintTx` and `BurnTx` immediately
2. Reduce per-epoch batch cap `M_max` to 10% of normal
3. Notify all validators and auditors
4. Begin emergency migration preparation

### Level 3 — Red (Full Freeze)

**Trigger**: confirmed exploit, double-finalization observed, or
trusted-setup compromise credibly demonstrated.

**Response**:
1. Issue `HaltTx` (requires `t`-of-`n` threshold authorization)
2. Only read-only state queries remain live
3. Publish incident report within 6 hours
4. Initiate emergency migration or coordinated shutdown via `ResumeTx`

### HaltTx / ResumeTx ownership guarantee

`HaltTx` and `ResumeTx` **do not** reassign coin ownership, modify
accumulator witnesses, or move any funds. They only suspend and resume
the `Apply` state transition function.

Neither transaction can be issued by a single party. Both require
`t`-of-`n` threshold authorization from the current validator set.

---

## 3. Value Cap Enforcement

Value caps apply to all Tier 1 deployments.

### Per-transaction cap

```
v <= V_max_tx
```

**Enforcement**: **in-circuit** as a range constraint.
The ZK proof is unsatisfiable for any `v > V_max_tx`.
No trusted party can override this.

### Per-epoch issuance cap

```
∑ (v_i for MintTx_i in B_k) <= V_max_epoch
```

**Enforcement**: **committee validation rule**.
Validators maintain a per-epoch issuance counter in the batch proposal.
Honest validators reject any proposal exceeding the cap.
The counter is included in `C_k` and is auditable.

### Total supply cap

```
∑ (v_cm for cm in ACC) <= V_max_total
```

**Enforcement**: **committee validation rule** backed by an
accumulator-derived supply counter in `S_k`.

### Redemption gate

`BurnTx` is **disabled by default**.
Enabled only by a signed operator declaration after audit sign-off.
The gate is a committee rule; no circuit change is required to enable it.

### Reference cap values (Tier 1 / testnet)

| Parameter | Reference value | Notes |
|-----------|----------------|-------|
| `V_max_tx` | 1,000 units | In-circuit enforced |
| `V_max_epoch` | 100,000 units | Committee enforced |
| `V_max_total` | 10,000,000 units | Committee enforced |

Operators set exact values; these are starting points, revisited after
each completed audit.

---

## 4. Quantum Migration Triggers

Migration triggers are **externally verifiable events**, not subjective
forecasts, to ensure objective escalation.

| Trigger | Definition |
|---------|-----------|
| **Standards milestone** | NIST formally deprecates RSA-2048 or elliptic-curve cryptography via a FIPS revision or sunset advisory with a fixed end-of-life date |
| **Community consensus** | A reproducible break or significant speedup against DL or factoring is documented in a peer-reviewed publication at a major venue (IEEE S&P, CCS, Crypto, Eurocrypt) and independently confirmed by at least one other research group |
| **Practical break** | A publicly reproducible demonstration breaks a concrete instance of BLS12-381 DL, RSA-2048 factoring, or the Groth16 proof system at any key size used in the deployment |

**On any trigger**: Level 2 (Orange) is declared within 24 hours.
Migration to Tier 2 begins under the epoch rotation protocol (paper §8).

---

## 5. Audit and Bug Bounty Requirements

### Tier 0

Internal review only. Open-source release with explicit
"experimental, no value" label.

### Tier 1

Minimum **two independent external security audits** covering:
1. The ZK circuit and relation R
2. The BFT + VDF finality sublayer
3. The admission and spam-resistance layer

Bug bounty active from day 1 of public testnet; severity-tiered rewards.

### Tier 2

All Tier 1 requirements plus:
- Formal verification of `Apply(S, B)` in a proof assistant (Coq or Lean)
- Hardware wallet integration and side-channel audit
- Jurisdiction-specific legal review

---

## 6. Cryptographic Agility

All cryptographic components are plug-in modules behind stable interfaces.
Replacement is executed as a coordinated epoch transition (paper §8) with
a mandatory overlap period of at least `W = 90 days` during which both
old and new components are accepted.

| Component | v1 (current) | v2 (planned) | v3 (future) |
|-----------|-------------|-------------|-------------|
| Proof system | Groth16 | PLONK / Halo2 | STARKs |
| Accumulator | RSA-2048 | RSA-2048 | Class-group |
| Threshold sig | BLS12-381 | BLS12-381 | Dilithium / Falcon |
| Hash (external) | BLAKE3 | BLAKE3 | SHA-3 family |
| Hash (circuit) | Poseidon | Poseidon | Rescue / Reinforced Concrete |

---

## 7. Responsibility Allocation

| Party | Responsibility |
|-------|---------------|
| Protocol authors | Specification correctness; honest disclosure of limitations |
| Implementers | Correctness vs. specification; side-channel hardening; wallet security |
| Operators | Audit compliance; value-cap enforcement; incident response; regulatory compliance |
| Users | Understanding deployment tier risk; not exceeding personal risk tolerance |

No party involved in creating this specification accepts liability for
losses arising from use of this protocol or any implementation derived
from it.
