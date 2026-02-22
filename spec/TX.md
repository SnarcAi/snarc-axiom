# SNARC AXIOM — Transaction Format Specification

> **License**: [CC BY-NC 4.0](../LICENSE) — Non-commercial use only. Commercial use requires a license. See [COMMERCIAL_LICENSE.md](../COMMERCIAL_LICENSE.md).


## Overview

All transactions are submitted as **envelopes** to the mempool:

```
TX_envelope = (
  tx      : SpendTx | MintTx | BurnTx | HaltTx | ResumeTx
  fee     : uint64        -- fee in protocol units
  nonce   : {0,1}^256     -- anti-replay / admission proof field
  ts      : uint64        -- Unix timestamp (advisory; not consensus-critical)
)
```

Validators accept envelopes through the two-tier mempool (Tier 0 cheap
checks → Tier 1 ZK verification). See `SPEC.md` for admission rules.

---

## 1. SpendTx — Standard Coin Transfer

Spends one coin and creates one new coin of equal value.

```
SpendTx = (
  sn      : {0,1}^256     -- nullifier = PRF_s(rho), revealed at spend
  cm_new  : G             -- new coin commitment = Com(<v, rho_new>; r_new)
  pi      : Proof         -- ZK proof for relation R (see paper §4)
)
```

### ZK relation R — what `pi` proves

The prover holds private witness `(v, rho, r, s, w, rho_new, r_new)` and
demonstrates:

| Constraint | Statement |
|------------|-----------|
| C1 | `cm = Com(<v, rho>; r)` — old commitment well-formed |
| C2 | `VerifyMem(ACC, cm, w) == 1` — coin exists in accumulator |
| C3 | `sn = PRF_s(rho)` — nullifier correctly derived |
| C4 | `cm_new = Com(<v, rho_new>; r_new)` — value conservation |
| C5 | Prover knows spend key `s` bound to the committed coin |

The **unspent condition** (`H(sn) ∉ SpentSetRoot`) is verified
**outside the circuit** against current state — this keeps the circuit
small and allows cheap Tier-0 pre-filtering.

### Multi-output extension

For `m` outputs with values `v_1, ..., v_m`:

```
SpendTxMulti = (
  sn       : {0,1}^256
  cm_new_i : G[m]         -- m new commitments
  pi       : Proof         -- proves ∑ v_out_i == v_in (in-circuit)
)
```

---

## 2. MintTx — Coin Issuance

Creates a new coin without spending an existing one. Requires issuer
authorization.

```
MintTx = (
  cm_new   : G            -- new coin commitment = Com(<v, rho>; r)
  v_pub    : uint64       -- public value (visible to validators)
  sig_auth : Signature    -- signature by authorized issuer key
)
```

**Enforcement**:
- `sig_auth` verified against a registered issuer public key in epoch state
- Value cap check: `v_pub <= V_max_tx` (in-circuit range proof on `v_pub`)
- Per-epoch issuance counter checked by committee (see `SECURITY-POLICY.md`)

**Result**: `cm_new` is inserted into `ACC`; no nullifier is added.

---

## 3. BurnTx — Coin Redemption

Destroys a coin and reveals its value publicly. BurnTx is **disabled by
default**; enabled only after audit sign-off (see `SECURITY-POLICY.md`).

```
BurnTx = (
  sn       : {0,1}^256    -- nullifier, same derivation as SpendTx
  v_pub    : uint64       -- value claimed (verified in ZK proof)
  pi_burn  : Proof        -- ZK proof: knows (v, rho, r, s, w) with v == v_pub
)
```

**Result**: `H(sn)` inserted into `SpentSetRoot`; no `cm_new` added to `ACC`.

---

## 4. HaltTx — Emergency Freeze

Suspends all state transitions. Requires `t`-of-`n` threshold authorization.

```
HaltTx = (
  epoch    : uint64
  reason   : {0,1}^512    -- human-readable UTF-8 incident description
  sigma    : TSig          -- threshold signature over (epoch || H(reason))
)
```

**Effect**:
- The `Apply` function stops being called
- No SpendTx, MintTx, or BurnTx is accepted
- Read-only state queries (certificate verification, light client proofs)
  remain available

**Critical limitation**: `HaltTx` **does not** reassign coin ownership,
modify `ACC`, modify `SpentSetRoot`, or move any funds. It only prevents
new state transitions from being applied.

---

## 5. ResumeTx — Resume After Freeze

Re-enables state transitions after a HaltTx. Requires **the same**
`t`-of-`n` threshold authorization as HaltTx.

```
ResumeTx = (
  epoch     : uint64
  halt_ref  : {0,1}^256   -- hash of the HaltTx being lifted
  action    : enum { RESUME | MIGRATE | SHUTDOWN }
  sigma     : TSig         -- threshold signature over (epoch || halt_ref || action)
)
```

**Actions**:

| Action | Effect |
|--------|--------|
| `RESUME` | Normal operation resumes at the current state |
| `MIGRATE` | Epoch transition begins with new cryptographic parameters |
| `SHUTDOWN` | Protocol is terminated; no further transactions accepted |

`ResumeTx` does not reverse any completed state transitions. Coins that
existed before `HaltTx` remain valid; their witnesses and nullifiers are
unchanged.

---

## 6. Batch Format

Transactions are grouped into batches for BFT finalization.

```
Batch = (
  epoch     : uint64
  seq       : uint64          -- must equal prev_state.seq + 1
  prev_hash : {0,1}^256       -- H(S_{k-1})
  tx_list   : TX_envelope[]   -- ordered list, max M_max entries
  vdf_out   : (y, pi_vdf)     -- VDF output for epoch beacon
  sigma     : TSig            -- threshold signature; computed last
)
```

`tx_list` ordering: sorted by `prio(TX) = H(sn || cm_new || y_epoch)` for
SpendTx; MintTx and BurnTx are placed at fixed positions (beginning and end
of batch respectively) for auditability.

`M_max` is an epoch-level parameter (reference value: 300 transactions/batch).

---

## 7. Admission Proofs (Tier 0)

Each `TX_envelope.nonce` field carries one of:

| Mechanism | Format | Cost |
|-----------|--------|------|
| Stake ticket | `H(pk || epoch || counter)` | Stake deposit |
| PoW-lite | Nonce s.t. `H(epoch || TX || nonce) < 2^{-d}` | CPU work |
| VRF ticket | `VRF.Eval(sk, seed_epoch || H(TX))` where output `< tau_tx` | Key + randomness |

Validators check the admission proof in Tier 0 **before** ZK verification.
