# SNARC AXIOM — Protocol Specification

> **License**: [CC BY-NC 4.0](../LICENSE) — Non-commercial use only. Commercial use requires a license. See [COMMERCIAL_LICENSE.md](../COMMERCIAL_LICENSE.md).


This document describes the protocol invariants, acceptance rules, and
attack surface summary. For state objects see `STATE.md`; for transaction
formats see `TX.md`; for deployment policy see `SECURITY-POLICY.md`.

---

## 1. System Invariants

These properties must hold at every finalized state `S_k` across all epochs.

### Cryptographic invariants

| ID | Invariant |
|----|-----------|
| INV-1 | **No double-spend**: for any nullifier `sn` finalized in batch `k`, `H(sn) ∈ SpentSetRoot_k` and no subsequent batch can include a transaction with the same `sn` |
| INV-2 | **Value conservation**: every SpendTx committed to a value `v`; the ZK circuit enforces `v_in == v_out` (or `∑ v_out_i == v_in` for multi-output) |
| INV-3 | **Coin membership**: every unspent coin `cm` has a valid accumulator witness `w` such that `w^{H'(cm)} ≡ ACC_k (mod N)` |
| INV-4 | **Nullifier uniqueness**: `sn = PRF_s(rho)` is unique per `(s, rho)` pair; collision requires breaking PRF security |
| INV-5 | **ZK soundness**: no SpendTx is accepted without a valid ZK proof for relation R |

### Distributed-systems invariants

| ID | Invariant |
|----|-----------|
| INV-6 | **Safety**: no two honest verifiers accept conflicting finalized states `S_k ≠ S_k'` at the same sequence number |
| INV-7 | **Atomicity**: `ACC_k` and `SpentSetRoot_k` are jointly certified by `sigma_k`; partial observation at protocol level is impossible |
| INV-8 | **Monotonic sequence**: `seq_k = seq_{k-1} + 1` for all `k > 0` |
| INV-9 | **Epoch chain**: each epoch's first batch includes `HC_{e-1}` binding `vk_e` to `S_{last, e-1}` |

---

## 2. Acceptance Rules

A validator accepts a SpendTx envelope and includes it in a proposed batch
if and only if all of the following hold (in order — fail-fast):

### Tier 0 (cheap — applied first)

```
R0.1  len(envelope) <= MAX_ENVELOPE_BYTES
R0.2  envelope.tx.type in {SpendTx, MintTx, BurnTx}
R0.3  AdmissionProof.Verify(envelope) == 1   [stake ticket / PoW-lite / VRF]
R0.4  envelope.fee >= f_min(epoch)
R0.5  H(sn || cm_new) not in mempool_cache   [duplicate suppression]
```

### Tier 1 (expensive — only for Tier 0 survivors)

```
R1.1  ZKVerify(pi, ACC_{current}, sn, cm_new) == 1
R1.2  SMT.NonMember(SpentSetRoot_{current}, H(sn)) == 1
R1.3  No intra-batch duplicate: sn not in {sn_j : j < i, TX_j already in batch}
```

A batch is accepted for finalization if and only if:

```
B.1   B.prev_hash == H(S_{k-1})
B.2   B.seq == S_{k-1}.seq + 1
B.3   All R1.x checks pass for all TX in B.tx_list
B.4   TSig.Verify(vk_epoch, H(B \ sigma), B.sigma) == 1
B.5   len(B.tx_list) <= M_max
```

---

## 3. Attack Surface Summary

This section catalogs the attack surface of SNARC AXIOM v1. Each entry
states the attack class, the protocol's defense, and any residual risk.

### 3.1 Cryptographic attacks

| Attack | Defense | Residual risk |
|--------|---------|---------------|
| Forge a ZK proof without witness | SNARK knowledge soundness (under KE assumption for Groth16) | Trusted setup compromise breaks soundness |
| Spend same coin twice | Nullifier uniqueness + INV-1 + safety theorem | None under honest majority |
| Inflate coin value | Pedersen commitment binding + in-circuit range proof | None under DL hardness |
| Deanonymize sender | ZK zero-knowledge + commitment hiding | Side-channel on prover; network-level timing analysis |
| Malleate a valid proof | Simulation-extractable NIZK (PLONK or Fiat-Shamir hardened Groth16) | Groth16 without sim-extractability: malleability possible; mitigated in v2 |
| Break accumulator membership | Strong RSA assumption | Trusted setup for RSA modulus; class-group alternative deferred to v2 |

### 3.2 Distributed-systems attacks

| Attack | Defense | Residual risk |
|--------|---------|---------------|
| Double-finalization (safety) | BFT quorum intersection; honest majority f < n/3 | None under threshold; total committee capture breaks safety |
| Censorship (liveness) | VDF ordering beacon; provable misbehavior attribution | Sustained censorship possible if > f validators collude; only weak fairness guaranteed |
| Long-range attack | Handoff certificate chain + weak subjectivity window W | Clients must sync within W epochs of last trusted checkpoint |
| Eclipse attack | Out-of-scope (network layer) | Requires network-level mitigations |
| Sybil attack on committee | Stake-weighted VRF sortition; sybil non-amplification theorem | Plutocracy risk if stake concentrated |

### 3.3 Operational attacks

| Attack | Defense | Residual risk |
|--------|---------|---------------|
| Mempool flooding / ZK-verify exhaustion | Two-tier mempool; Tier 0 cheap filters gate Tier 1 | Cost bounded but not zero |
| Quantum attack on Groth16/BLS/RSA | Migration trigger policy (see SECURITY-POLICY.md) | v1 not PQ-secure; migration to v3 required before production |
| Key theft (spend key) | Out-of-scope (wallet security) | Wallet implementation is critical path |
| Admin rug-pull | HaltTx/ResumeTx require t-of-n threshold; cannot reassign ownership | Validator collusion (> t nodes) could freeze the system |
| Supply inflation via MintTx | Per-epoch and total supply caps; issuer authorization sig | Committee-enforced caps; issuer key compromise |

### 3.4 Known limitations (honest disclosure)

1. **Trusted setup (Groth16)**: The ceremony must be run correctly and toxic waste discarded. MPC ceremony mitigates; elimination requires STARKs (v3).

2. **RSA modulus trust**: The RSA modulus `N = pq` must be generated with destroyed factors. Alternative: class-group accumulator (v2).

3. **Non-malleability**: Groth16 as described is not simulation-extractable. Full non-malleability requires PLONK or a wrapper; deferred to v2.

4. **Stake concentration**: The sybil non-amplification theorem holds per depositor, but large stakeholders still dominate committee selection. Mitigation (quadratic weighting, per-validator caps) is a deployment parameter.

5. **Spent-set archival**: `SpentSetRoot` is constant-size, but the underlying set grows at ~60 GB/year under reference parameters. Archival and checkpoint policy is an operator responsibility.

---

## 4. Parameter Summary (v1 Reference)

| Parameter | Value | Where used |
|-----------|-------|-----------|
| `lambda` | 128 bits | Security parameter throughout |
| RSA modulus | 2048 bits | Accumulator |
| Curve | BLS12-381 | Commitments, TSig, VRF |
| Proof system | Groth16 | ZK proofs |
| Hash (external) | BLAKE3 | PRF, beacon, envelope hashing |
| Hash (in-circuit) | Poseidon | Commitment and nullifier constraints |
| VDF | Wesolowski, T=4000 squarings | Ordering beacon (~2s eval, ~5ms verify) |
| TSig threshold | t = ⌊2n/3⌋ + 1 | BFT finality |
| Batch cap | M_max = 300 TX/batch | Throughput and DoS bound |
| Epoch duration | T_e ≈ 5s | Liveness target |
| Weak subjectivity window | W = 90 days | Long-range attack resistance |

---

## 5. Version History

| Version | Status | Notes |
|---------|--------|-------|
| v0.1 | Current | Research paper + spec (this repo) |
| v0.2 | Planned | PLONK/Halo2 (universal setup); non-malleability |
| v1.0 | Future | STARKs + PQ signatures + class-group accumulator |
