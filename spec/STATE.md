# SNARC AXIOM — State Object Specification

> **License**: [CC BY-NC 4.0](../LICENSE) — Non-commercial use only. Commercial use requires a license. See [COMMERCIAL_LICENSE.md](../COMMERCIAL_LICENSE.md).


## 1. Global State

The system maintains a single global state object `S_k` at each finalization step `k`.

```
S_k = (
  ACC_k          : Z_N*          -- RSA accumulator value (256 bytes)
  SpentSetRoot_k : {0,1}^256     -- Sparse Merkle Tree root over spent nullifiers (32 bytes)
  epoch_k        : uint64        -- monotonically increasing epoch counter
  seq_k          : uint64        -- monotonically increasing batch sequence number
)
```

**State size**: constant, independent of transaction history.

**Genesis state**:
```
S_0 = (
  ACC_0          = g              -- accumulator generator
  SpentSetRoot_0 = SMT.EmptyRoot  -- empty Sparse Merkle Tree root
  epoch_0        = 0
  seq_0          = 0
)
```

---

## 2. Finalization Certificate

Each finalized batch produces a certificate `C_k` that clients use as the canonical finality artifact, replacing ledger entries.

```
C_k = (
  B_k     : Batch         -- the finalized batch (see TX.md)
  S_k     : State         -- the resulting state after applying B_k
  sigma_k : TSig          -- threshold signature over the preimage below
)
```

**Signature preimage**:
```
sigma_k = TSig_t( H( B_k.hash || S_k.ACC || S_k.SpentSetRoot || S_k.epoch || S_k.seq ) )
```

`sigma_k` covers both the batch content and the resulting state roots **jointly**.
This prevents partial observation: a client accepting `sigma_k` accepts both
`ACC_k` and `SpentSetRoot_k` simultaneously.

---

## 3. State Transition Function: Apply(S, B) → S'

`Apply` is a **pure, deterministic** function. The same `(S, B)` always produces the same `S'` or always fails.

### Preconditions (all must hold)

| # | Check | Failure action |
|---|-------|----------------|
| P1 | `B.prev_hash == H(S)` | Reject batch |
| P2 | `B.seq == S.seq + 1` | Reject batch |
| P3 | `TSig.Verify(vk_epoch, H(B \ sigma), B.sigma) == 1` | Reject batch |
| P4 | For all TX in B: `ZKVerify(TX.pi, S.ACC, TX.sn, TX.cm_new) == 1` | Reject batch |
| P5 | For all TX in B: `SMT.NonMember(S.SpentSetRoot, H(TX.sn)) == 1` | Reject batch |
| P6 | No duplicate nullifiers within B: `sn_i != sn_j` for `i != j` | Reject batch |

### Output state (if all preconditions pass)

```
ACC'          = S.ACC ^ ( ∏_i H'(TX_i.cm_new) ) mod N

SpentSetRoot' = SMT.BatchInsert(S.SpentSetRoot, [ H(TX_i.sn) for TX_i in B ])

S' = (
  ACC          = ACC'
  SpentSetRoot = SpentSetRoot'
  epoch        = S.epoch          -- unchanged mid-epoch
  seq          = S.seq + 1
)
```

`H'` is a deterministic hash-to-prime function.
`SMT.BatchInsert` processes insertions in `tx_list` index order (deterministic).

### Atomicity guarantee

If `Apply` succeeds, both `ACC'` and `SpentSetRoot'` are jointly covered by
the threshold signature `sigma_k`. Neither can be modified independently
without forging `sigma_k`.

If any precondition fails, `Apply` is undefined and `S` is **unchanged**.
No partial state update is possible.

---

## 4. State Invariants

These invariants must hold at every finalized state `S_k`:

| Invariant | Statement |
|-----------|-----------|
| I1 — No double-spend | For any nullifier `sn` inserted in batch `k`, `H(sn)` is in `SpentSetRoot_k` and cannot be inserted again |
| I2 — Value conservation | For each SpendTx: committed input value == committed output value (enforced in-circuit) |
| I3 — Monotonic seq | `seq_k = seq_{k-1} + 1` for all `k > 0` |
| I4 — Monotonic epoch | `epoch_k >= epoch_{k-1}` for all `k` |
| I5 — ACC membership | For every unspent coin `cm`: a valid membership witness `w` exists such that `w^{H'(cm)} ≡ ACC_k (mod N)` |
| I6 — Joint certification | `sigma_k` is valid over `(ACC_k, SpentSetRoot_k, epoch_k, seq_k)` jointly |

---

## 5. Light Client Verification

A light client holding finalization certificate `C_k` can verify coin liveness without full state:

```
1. Verify: TSig.Verify(vk_epoch, preimage, C_k.sigma) == 1
2. Compute: sn_coin = PRF_s(rho)   [locally, never revealed]
3. Check:   SMT.NonMember(C_k.S.SpentSetRoot, H(sn_coin)) == 1
4. Check:   ACC.VerifyMem(C_k.S.ACC, cm, w) == 1
```

Cost: O(log n) for the Merkle proof, O(1) for TSig and accumulator verification.

---

## 6. Epoch State

An epoch boundary is signaled by `epoch_k > epoch_{k-1}`.
The first batch of a new epoch includes a Handoff Certificate `HC_e` that
binds the new committee verification key `vk_{e+1}` to the last state of
epoch `e`.

```
HC_e = TSig_{V_e}( H( vk_{e+1} || e+1 || S_{last,e} ) )
```

See `spec/SECURITY-POLICY.md` for epoch rotation and freeze policy.
