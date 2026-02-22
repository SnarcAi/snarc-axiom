# SNARC AXIOM

> **⚠️ Research-only. No token. No network. No assets.**
> This repository contains a paper and protocol specifications only.
> Do not deploy with real value. See [DISCLAIMER.md](DISCLAIMER.md).

---

**SNARC AXIOM** is a research protocol design for **ledgerless, privacy-preserving payments** using:

- a dynamic RSA accumulator over coin commitments (`ACC`) — constant-size accumulator value (256 bytes for RSA-2048)
- a Sparse Merkle Tree root over spent nullifiers (`SpentSetRoot`) — constant 32 bytes
- zero-knowledge proofs (Groth16 / PLONK) for transaction validity
- hybrid **BFT threshold finality + VDF ordering** for global state consistency

The core insight: blockchain's three roles — canonical ordering, nullifier tracking, coin-validity auditing — can be decomposed. Coin validity becomes a ZK proof. Double-spend prevention becomes a nullifier registry. Global consistency becomes a BFT finalization certificate. Together, these replace the ledger with two constant-size state roots.

---

## Repository Contents

```text
paper/                 LaTeX sources + compiled PDF
spec/                  Protocol specification (state, transactions, invariants)
README.md              This file
DISCLAIMER.md          Legal / scope disclaimer (read before using)
SECURITY.md            Responsible disclosure policy
CONTRIBUTING.md        Contribution guidelines
LICENSE                Dual license summary (paper CC BY 4.0, spec CC BY-NC 4.0)
COMMERCIAL_LICENSE.md  Commercial licensing terms for spec/
```

---

## Paper

- [`paper/snarc-axiom-paper.pdf`](paper/snarc-axiom-paper.pdf) — main PDF (article format)
- [`paper/snarc-axiom-acmart-ccs.pdf`](paper/snarc-axiom-acmart-ccs.pdf) — ACM sigconf PDF (CCS-ready)
- [`paper/main.tex`](paper/main.tex) — LaTeX source (article)
- [`paper/main_acmart.tex`](paper/main_acmart.tex) — LaTeX source (ACM sigconf)
- [`paper/axiom.bib`](paper/axiom.bib) — bibliography (16 references)

### Build

```bash
cd paper/
pdflatex main.tex
bibtex main
pdflatex main.tex
pdflatex main.tex
```

Tested on TeX Live 2023. Zero errors, zero warnings.

---

## Protocol Specification

| File | Description |
|------|-------------|
| [`spec/SPEC.md`](spec/SPEC.md) | Invariants, acceptance rules, attack surface |
| [`spec/STATE.md`](spec/STATE.md) | Global state `S_k`, certificate `C_k`, `Apply()` |
| [`spec/TX.md`](spec/TX.md) | SpendTx / MintTx / BurnTx / HaltTx / ResumeTx |
| [`spec/SECURITY-POLICY.md`](spec/SECURITY-POLICY.md) | Deployment tiers, freeze policy, value caps |

---

## Paper Structure

| Section | Title |
|---------|-------|
| §1 | Introduction |
| §2 | Notation and Preliminaries |
| §3 | Threat Model |
| §4 | Protocol Overview and ZK Relation |
| §5 | State Consistency Without a Ledger (Hybrid BFT + VDF) |
| §6 | ACC Update Atomicity |
| §7 | Spam and DoS Resistance |
| §8 | Epoch Rotation and Committee Freshness |
| §9 | Permissionless Committee Selection (VRF Sortition) |
| §10 | Concrete Instantiations and Benchmarks |
| §11 | Security Policy and Responsible Deployment |
| §12 | Related Work |
| §13 | Conclusion |

---

## Key Properties

| Property | Status |
|----------|--------|
| Double-spend resistance | ✅ Theorem + proof sketch |
| Value conservation | ✅ Theorem + proof sketch |
| Sender anonymity | ✅ Zero-knowledge argument |
| Atomic state transition | ✅ Formal definition + theorem |
| BFT safety (partial synchrony) | ✅ Theorem |
| BFT liveness (post-GST) | ✅ Theorem |
| Sybil resistance | ✅ Theorem |
| Post-quantum | ❌ v1 not PQ-secure — deferred to v3 |
| Trusted-setup-free | ❌ v1 uses Groth16 — deferred to v2 |

---

## Scope and Non-Goals

- **Not** an implemented cryptocurrency
- **Not** investment advice
- **No** token issuance, exchange listing, or fundraising — now or planned

---

## Citation

```bibtex
@misc{snarcaxiom2025,
  author       = {SNARC AXIOM Authors},
  title        = {{SNARC} {AXIOM}: Ledgerless Privacy-Preserving Payments
                  via Accumulators, Nullifier Roots, and Hybrid
                  {BFT}+{VDF} Finality},
  howpublished = {GitHub repository},
  year         = {2025},
  url          = {https://github.com/SnarcAi/snarc-axiom}
}
```

---

## Contact

- Technical discussion: open a [GitHub Issue](../../issues)
- Security vulnerabilities: see [SECURITY.md](SECURITY.md)
- Commercial licensing: see [COMMERCIAL_LICENSE.md](COMMERCIAL_LICENSE.md)

---

## License (Dual)

| Directory | License | Commercial use |
|-----------|---------|---------------|
| `paper/` | [CC BY 4.0](LICENSE) | ✅ Allowed with attribution |
| `spec/` | [CC BY-NC 4.0](LICENSE) | ❌ Requires commercial license |
| `code/` (future) | Apache 2.0 or AGPL 3.0 | TBD at release |

Commercial use of the protocol specification requires a license.
See [COMMERCIAL_LICENSE.md](COMMERCIAL_LICENSE.md) for terms and contact.
