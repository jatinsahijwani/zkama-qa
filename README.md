# ZKAMA: Response to Kusama ZK Bounty Curator Questions

**Project:** ZKAMA, a Zero-Knowledge Verifier SDK for Kusama PolkaVM
**Bounty:** Kusama Vision Bounty (Zero Knowledge & Advanced Cryptography)
**Team:** Jatin Sahijwani (Lead), Anirudh Singh Chouhan
**Repository:** [github.com/jatinsahijwani/zk-polka-sdk](https://github.com/jatinsahijwani/zk-polka-sdk)
**Contact:** jatinsahijwani2151@gmail.com
**Document status:** v1, initial response to the async follow-up questions from the curator call

---

## TL;DR

After the curator call, we got ten async follow-up questions covering the M1 technical flow, the native Rust verifier design, cryptographic choices, benchmark commitments, recursion utility, the M3 audit allocation, and the proposed funding structure. The detailed answers are below. Headline points first:

- **Yes**, we accept the proposed structure of **M1-only funding first**, with M2 and M3 contingent on a working testnet demo. Honestly, this is the structure we'd prefer anyway.
- **M1 scope:** we'd like to keep the original **$30,000 over 6 weeks**, delivering a working end-to-end pipeline on Kusama Asset Hub testnet. We've reviewed the scope carefully and we'd rather hold the budget steady than thin out the deliverables.
- **First-supported configuration:** BN254 curve, Groth16 proof system, SnarkJS-compatible verification key format, SCALE on-chain serialization.
- **Engineering posture:** we'll publish honest, reproducible benchmark numbers rather than promise specific performance multiples. If Rust-native verification doesn't outperform Solidity/Revive, we'll say so and adjust the SDK's recommended defaults.
- **Audit:** $18,000 of the original M3 budget is reserved for an external audit. Auditor selected via 2 to 3 quotes at M3 kickoff, with curator visibility into the selection.

A revised M1-only proposal PDF can be ready within **48 hours** of curator confirmation.

---

## Background

This doc responds to async follow-up questions raised by the Kusama ZK Bounty curators after the introductory call. During that call we walked through a live demo of the current `zk-polka-sdk` (the ZKAMA MVP), showing the circuit to proof to on-chain verification flow via CLI commands and a TypeScript integration script. The questions below go deeper into technical specifics, commitment language, and the funding structure.

The answers are written to be referenceable. Curators are welcome to link to specific sections, and we'll update this doc as scope evolves.

---

## Table of Contents

1. [Full M1 flow: circuit to proof to verifier to testnet verification](#1-full-m1-flow)
2. [How the native Rust verifier will work on PolkaVM](#2-native-rust-verifier-on-polkavm)
3. [First-supported curve, proof system, key format, serialization](#3-cryptographic-defaults)
4. [Benchmark commitments and baselines](#4-benchmarks-and-baselines)
5. [Why we expect Rust-native to outperform Solidity/Revive](#5-rust-vs-solidity-performance-thesis)
6. [Fallback plan if Rust-native is not faster or stable](#6-fallback-plan)
7. [Recursive proofs: concrete utility and milestone deliverable](#7-recursion-utility)
8. [M3 audit budget breakdown](#8-m3-audit-budget)
9. [Auditor selection and audit scope](#9-auditor-and-audit-scope)
10. [Acceptance of M1-only funding structure](#10-m1-only-funding-acceptance)
11. [Proposed next steps](#proposed-next-steps)
12. [Team & contact](#team--contact)

---

## 1. Full M1 flow

> **Question:** Can you show the full M1 flow from circuit → proof generation → verifier → PolkaVM testnet verification?

We demonstrated this end-to-end during the curator call using the `zk-polka-sdk` repo. The documented form of that flow:

### CLI pipeline

```bash
# 1. Compile a Circom circuit
zkama compile ./circuits/age_verify.circom
# Outputs: age_verify.r1cs, age_verify.wasm

# 2. Generate trusted setup (dev mode for testing)
zkama setup --dev --circuit age_verify
# Outputs: age_verify.zkey, verification_key.json

# 3. Generate a proof from inputs
zkama prove --circuit age_verify --input ./input.json
# Outputs: proof.json, public.json

# 4. Generate the verifier contract (Solidity for M1)
zkama generate-verifier --target solidity --circuit age_verify
# Outputs: AgeVerifier.sol

# 5. Deploy to Asset Hub testnet via Revive
zkama deploy --network kusama-asset-hub-testnet --contract AgeVerifier.sol
# Outputs: contract address, deployment tx hash

# 6. Verify proof on-chain
zkama verify-onchain --proof proof.json --public public.json --contract <address>
# Outputs: tx hash, verification result
```

### Programmatic equivalent

```javascript
import {
  compile, setup, prove, generateVerifier, deploy, verifyOnchain
} from '@zkama/sdk';

const artifacts = await compile('./circuits/age_verify.circom');
const keys      = await setup({ artifacts, mode: 'dev' });
const proof     = await prove({ keys, input: { age: 21 } });
const verifier  = await generateVerifier({ keys, target: 'solidity' });
const address   = await deploy({ verifier, network: 'kusama-asset-hub-testnet' });
const result    = await verifyOnchain({ proof, contract: address });
```

### End-to-end characteristics

- Total time on a developer laptop (Apple M2 or Linux x86 8-core): under 5 minutes for standard circuits.
- On-chain verification: one Asset Hub block.
- A recorded end-to-end demo video ships as part of M1 deliverables.

---

## 2. Native Rust verifier on PolkaVM

> **Question:** How exactly will the native Rust verifier work on PolkaVM?

The native Rust verifier is the genuinely novel engineering work in ZKAMA. Everything else in M1 is orchestration over existing tools (Circom, SnarkJS, Revive). The Rust-native path is what differentiates ZKAMA from being a thin wrapper.

### Compilation pipeline

- Verifier source is written in `no_std` Rust so we don't pull in the standard library.
- Compiled to PolkaVM's RISC-V target (`riscv32em-unknown-none-elf`, or PolkaVM's current target spec) using the standard Rust toolchain.
- Output is a PolkaVM contract binary deployable via Revive's contracts pallet.

### Cryptographic operations

BN254 pairing arithmetic is the heaviest operation in a Groth16 verifier. We handle it via one of two paths, selected at deploy time based on PolkaVM precompile availability:

| Path | When used | Tradeoff |
|---|---|---|
| **A. Precompile-backed** | BN254 precompiles available on Asset Hub | Faster, aligns with the BN254 precompile RFP from the curator team |
| **B. Pure Rust** | Precompiles not available, or precompile-independence is required | Slower but portable, relies on a `no_std` port of `ark-bn254` |

The SDK detects available precompiles at deploy time and selects the appropriate path automatically. Users can override via a CLI flag.

### Verifier state and entry point

The verification key gets committed to contract storage on first deployment (a one-time cost). Subsequent `verify(proof, public_inputs)` calls load the key from storage and run the verification routine:

```rust
#[polkavm_export]
pub fn verify(proof: Proof, public_inputs: Vec<Fr>) -> bool {
    let vk = load_verification_key_from_storage();
    groth16_verify(&vk, &proof, &public_inputs)
}
```

### Honest disclosure on maturity

PolkaVM and Revive are still maturing. Porting arkworks crates cleanly to a `no_std` RISC-V target will involve friction we haven't fully mapped yet, specifically around host-call interfaces and the conditional compilation gates for `std`/`no_std` boundaries. M2 is budgeted with this in mind, and any blockers will be reported transparently both to curators and upstream to the PolkaVM/Revive teams.

---

## 3. Cryptographic defaults

> **Question:** Which curve, proof system, verifier key format, and serialization format will you support first?

### Curve: BN254 (alt_bn128)

Why BN254:

- It has the most mature tooling in the Circom/SnarkJS ecosystem.
- BN254 precompile support is on the active draft RFP list from the curator team, so we get first-class alignment.
- EVM-compatible bytecode reuse path for Solidity verifiers, which lowers the migration barrier for teams coming from Ethereum.

Follow-on curves for later milestones: **BLS12-381** (better security margin, native Substrate support) and the **BN254-Grumpkin cycle** (for efficient recursion).

### Proof system: Groth16 first (M1), PLONK second (M2)

| Property | Groth16 (M1) | PLONK (M2) |
|---|---|---|
| Proof size | ~200 bytes | ~500 to 1,000 bytes |
| Verification speed | Fastest | Fast |
| Trusted setup | Per-circuit | Universal (one-time) |
| Tooling maturity | Circom/SnarkJS, large ecosystem | Noir/Barretenberg, growing fast |
| Recursion support | Limited | Native via Barretenberg |

Groth16 first because of ecosystem maturity and proof size. PLONK second because of the better trusted-setup story (one universal ceremony rather than per-circuit) and native recursion support.

### Verifier key format

- **Off-chain / portability:** SnarkJS-compatible JSON. Any verification key generated by ZKAMA can be consumed by existing SnarkJS toolchains and vice versa.
- **On-chain storage:** Compact custom binary encoding, documented in the SDK spec, optimized for the `proof_size` gas dimension. Verification keys are static per circuit, so encoding and decoding cost is paid once at deployment.

### Serialization format

- **Proofs on the wire:** Standard G1/G2 affine point encoding (~256 bytes for Groth16 BN254).
- **On-chain inputs:** SCALE encoding (Substrate native) for proof bytes and public inputs. This is the canonical Polkadot encoding and it integrates cleanly with `polkadot-api` / PAPI on the client side.
- **Off-chain interop:** SnarkJS JSON continues to be supported for any team migrating from Ethereum tooling.

---

## 4. Benchmarks and baselines

> **Question:** What benchmark numbers will you commit to delivering, and against what baseline?

We're committing to a transparent reporting methodology rather than a specific performance multiple. Overpromising on benchmarks is the fastest way to lose credibility, and underpromising costs nothing.

### Measured metrics (per standard circuit)

Standard circuits used for benchmarking:

- Poseidon hash
- Merkle membership proof
- Age verification

For each circuit we'll publish:

| Metric | Unit |
|---|---|
| Proof generation time | milliseconds (reference: Apple M2 or Linux x86 8-core) |
| Proof size | bytes |
| On-chain verification: `ref_time` gas dimension | gas units |
| On-chain verification: `proof_size` gas dimension | bytes |
| Contract deployment cost | gas + storage deposit |

### Baselines published side-by-side

1. **EVM baseline.** Same circuit, Solidity Groth16 verifier deployed on Polygon. Well-documented at roughly 250k gas for BN254 Groth16.
2. **PolkaVM Solidity/Revive baseline.** Same circuit, Solidity Groth16 verifier compiled via Revive to PolkaVM, deployed on Asset Hub testnet.
3. **PolkaVM native Rust.** Our target deliverable.

### What we explicitly do not commit to

We won't make a claim like "Rust-native will be N times faster than Solidity/Revive." We don't know the multiple yet, and generating those numbers honestly is what M1 and M2 actually produce.

If Rust-native ends up 1.2x faster, we'll report that. If it's 5x faster, we'll report that. If Solidity/Revive turns out to be faster on some dimensions, we'll report that too. All benchmark code, circuits, and methodology will be open-source so the numbers are independently reproducible.

---

## 5. Rust vs Solidity performance thesis

> **Question:** Why should we believe Rust-native verification will outperform Solidity/Revive on PolkaVM?

Honest answer, including where we could be wrong.

### Why we expect Rust-native to outperform

1. **No translation layer.** Solidity to Revive to PolkaVM RISC-V involves an EVM-semantics emulation step. Rust compiles directly to RISC-V. Removing the intermediate semantics translation removes overhead.
2. **Direct host-call access.** A Rust contract can invoke PolkaVM host calls (including BN254 precompiles when available) directly. A Solidity contract routes through Revive's translation of EVM opcodes, which adds per-call overhead.
3. **Multi-dimensional gas model alignment.** Rust gives us more control to optimize separately for `ref_time` versus `proof_size`. Solidity's opcode model is built around one-dimensional EVM gas, which under-utilizes PolkaVM's split.
4. **Compactness at the algorithm layer.** Rust pairing libraries (the arkworks family) are heavily optimized for size and speed at the algorithmic level, free from EVM stack-machine constraints.

### Where we could be wrong

1. Revive is well-engineered. The translation overhead may be small in practice.
2. SnarkJS-generated Solidity Groth16 verifiers are heavily optimized at the assembly level. A Rust verifier would need similar optimization passes to actually beat them.
3. PolkaVM's Rust support is still maturing. There may be codegen or ergonomic issues we haven't run into yet.

### Honest framing

We expect Rust-native to outperform on the `proof_size` dimension (where compactness matters most), and possibly on `ref_time` if precompile access is cleaner from Rust. We don't yet have concrete numbers to back this. Generating those numbers honestly is exactly what M1 + M2 produces.

---

## 6. Fallback plan

> **Question:** What is your fallback plan if Rust-native verification is not faster or not stable enough?

A tiered fallback by scenario:

### Scenario A: Rust-native is only marginally faster (under 1.5x)

- Continue to ship Rust as an option for projects that want a full-stack Rust target.
- Default the SDK to Solidity/Revive (the more mature path) and document the tradeoff honestly.
- Reorder M2 priorities: less time on Rust optimization, more on PLONK/Noir and the standards library.

### Scenario B: Rust-native has stability issues

(Compilation, runtime, or host-call interface bugs that block production use.)

- Ship Solidity/Revive as the production-recommended track.
- Continue Rust development on a slower timeline, behind an `--experimental` flag.
- Report blockers upstream to the PolkaVM/Revive teams as part of our work.

### Scenario C: Rust-native is actually slower than Solidity/Revive

- Still ship it as an option. Some projects want full-Rust stacks for reasons beyond gas.
- Document the performance tradeoff honestly in the SDK so users can make informed choices.
- Reallocate freed-up M2 budget to additional standards library circuits and developer tooling.

### Non-negotiable

The SDK still ships and is still useful. The M1 deliverables (CLI, JS SDK, Solidity verifier path, docs, tests) aren't contingent on Rust outperforming anything. **M1 has value on its own merits even if every Rust hypothesis turns out wrong.**

---

## 7. Recursion utility

> **Question:** How do you plan to use recursive proofs in your product, and how will you ensure recursion is actually useful? What exactly will recursion do in your product, and what will you prove in the milestone?

Fair pushback. Recursion is often academic fluff. Here's the concrete utility ZKAMA is committing to:

### Production use case: batch verification

When an application needs to verify many independent proofs (say, 100 user KYC proofs to gate access to a feature), verifying each on-chain costs ~250k gas × 100 = 25M gas. With recursion, those 100 individual Groth16 proofs get aggregated off-chain into a single PLONK proof, which is verified on-chain in one call (~500k gas).

| Strategy | Gas cost |
|---|---|
| Verify 100 proofs individually | ~25,000,000 gas |
| Verify 1 aggregated proof | ~500,000 gas |
| **Approximate savings** | **~50x** (varies with aggregation depth and circuit) |

### Concrete M3 milestone deliverable

Aggregate 10 individual Groth16 BN254 proofs (each for a Poseidon-based membership circuit) into a single PLONK proof using Barretenberg's recursion API. Deploy a PLONK verifier on Asset Hub testnet. Verify the aggregated proof on-chain and publish:

1. Gas cost of verifying 10 proofs individually
2. Gas cost of verifying the 1 aggregated proof
3. Off-chain aggregation time
4. Net gas savings as a percentage

### Honest kill-criterion

If the aggregation overhead (PLONK proof size + verification cost on PolkaVM) doesn't produce a clear net savings versus linear verification, we'll deprioritize this scope and reallocate budget to additional standards library circuits and audit coverage. The decision and the numbers will be public.

### Implementation

Via Noir + Barretenberg, which has native recursion support. We're not building recursion from scratch. We're integrating an existing recursion engine into a clean SDK interface.

---

## 8. M3 audit budget

> **Question:** How much of M3 is actually reserved for an external audit?

Specific breakdown of the M3 budget ($39,000 total):

| Line item | Amount |
|---|---|
| **External smart contract audit** | **$18,000** |
| Formal verification (MIRI / Kani) work | $4,000 |
| Recursive proof composition development | $8,000 |
| Workshops (3 sessions) + office hours (6 weeks) | $5,000 |
| Launch, CI/CD, community framework | $2,000 |
| Operations / QA coordination | $2,000 |
| **Total** | **$39,000** |

The $18,000 audit allocation is calibrated against quotes we've informally received for ZK verifier audits of comparable scope. It sits at the low end of the market. Most ZK audits of similar scope range from $20k to $40k, so we've been conservative about what an auditor can actually deliver at that price.

If quotes come back higher than expected, we'll either narrow the audit scope (audit only the Rust verifier core, not the Solidity templates) or come back to curators with an honest revised number rather than reduce audit quality.

---

## 9. Auditor and audit scope

> **Question:** Who is the intended auditor, and what exact code will be audited?

### Auditor selection process

We haven't locked in an auditor pre-bounty. The process at the start of M3:

1. Solicit 2 to 3 written quotes from established ZK auditors.
2. Share the comparison with curators.
3. Select with curator input.

### Candidate auditors

- **Veridise.** ZK-specialized, audited Aztec, Worldcoin, ZK Email.
- **Zellic.** Strong Rust and smart contract track record.
- **Hexens.** ZK and cryptography focus.
- **ChainSecurity.** Broader smart contract focus, useful if no ZK-specialist quotes come back in budget.

If curators have other preferred auditors (e.g. teams familiar with PolkaVM specifically), we're happy to include them in the quote process.

### Audit scope

**In scope:**

- Generated Solidity verifier templates (Groth16 BN254)
- Native Rust verifier core (BN254 pairing arithmetic, `no_std` memory safety, host-call interface)
- Proof serialization and deserialization logic
- Verification key parsing and validation

**Explicitly out of scope:**

- User-written circuits. Circuit-level correctness is the user's responsibility, with our docs guiding best practices.
- Upstream libraries (snarkjs, Barretenberg, arkworks). These are audited by their own teams, and we don't pay to re-audit them.
- The CLI / JS SDK orchestration layer. Useful to audit eventually but not security-critical (no secrets handled, no on-chain state).
- Recursive proof composition logic. It depends on Barretenberg's audited recursion engine. If our integration introduces new attack surface, we'll expand scope and discuss budget impact with curators.

### Audit deliverables

A public report published with the M3 release, plus any remediations the audit recommends incorporated into the codebase before M3 sign-off.

---

## 10. M1-only funding acceptance

> **Question:** Will you accept M1-only funding first, with M2/M3 approved only after a working testnet demo?

**Yes, and this is actually the structure we'd prefer.**

### Proposed M1 scope

We'd like to keep the original M1 budget of **$30,000 over 6 weeks**, delivering:

- `zkama-cli` with the full pipeline shown in Question 1
- `@zkama/sdk` TypeScript package
- Solidity Groth16 verifier generation via Revive
- One working example dApp (anonymous voting or age verification) deployed on Asset Hub testnet
- Public documentation and a getting-started guide
- Open-source repository with tests (over 80% coverage)
- A recorded end-to-end demo video

We considered reducing the M1 budget to make the ask smaller, but on review the original $30k matches the actual work involved. We'd rather hold the budget at $30k and deliver against it well, rather than thin the scope and ship something half-finished. The ~5.5 month, $105k total commitment was the part we'd flag as oversized for a first engagement, not M1 specifically.

### Validation gate after M1

- A working demo to curators on Asset Hub testnet.
- Public release of all M1 artifacts (repo, docs, demo video).
- Curators evaluate execution quality, code quality, and whether the work justifies continued funding.

**If the gate is passed:** M2 and M3 get re-scoped based on (a) what we learned in M1, (b) curator priorities, and (c) what the ecosystem actually needs. The total scope and budget for M2 + M3 are open to renegotiation. We won't hold curators to the original $75,000 for M2 + M3 if a leaner shape makes more sense.

**If the gate is not passed:** no further commitment from the curators. We'll have shipped a useful piece of open-source tooling either way, and the bounty has limited exposure.

### Why we're enthusiastic about this structure

1. It matches the curators' stated lean-POC funding philosophy from the Q1 report.
2. It caps curator risk at the M1 amount.
3. It forces us to ship something concrete and demoable before asking for more, which is the right discipline.
4. It gives both sides real data (working code, real benchmarks, real adoption signals) before committing to the larger scope.

---

## Proposed next steps

| Step | Owner | Timeline |
|---|---|---|
| Curators confirm M1-only structure ($30k, 6 weeks) | Curators | At their convenience |
| Revised M1-only proposal PDF delivered | ZKAMA team | Within **48 hours** of confirmation |
| M1 kickoff | ZKAMA team | Upon bounty funding |
| M1 delivery + testnet demo | ZKAMA team | T + 6 weeks |
| M2 / M3 re-scoping discussion | Both | Post-M1 demo |

Happy to jump on another short call if any of the answers above need clarification, or to discuss specific revised-scope shapes for the M1-only structure.

---

## Team & contact

### Core team

**Jatin Sahijwani**, Lead Developer.
Full-stack engineer, ZK circuit design and TypeScript SDK development. Built the ZKAMA MVP that won 1st place at the Polkadot AssetHub Hackathon (Goa). PBA-X graduate.
[GitHub](https://github.com/jatinsahijwani) · [X](https://x.com/jatinsahijwani1) · [LinkedIn](https://linkedin.com/in/jatinsahijwani)

**Anirudh Singh Chouhan**, Developer.
Systems developer with deep Rust and smart contract experience. Co-built the ZKAMA MVP. PBA-X graduate.
[GitHub](http://github.com/AnirudhSingh07/) · [X](https://x.com/kunwarAnirudhS3) · [LinkedIn](https://www.linkedin.com/in/anirudhsinghchouhan/)

### Operations & QA

**Edgetributor SubDAO.** 6 contributors (Shankar, Gagan, Prashant, Rama, Raj, Pranav) providing project management, architecture reviews, DevOps, and QA.

### Ecosystem contributions

- Co-founders of **HackTour INDIA**, with 500+ developer registrations across 6+ IRL Polkadot/Kusama meetups.
- 15+ hackathon wins across multiple chains.
- Prior ZK developer tooling grant delivered for the **Arbitrum Foundation**.

### Contact

- **Email:** jatinsahijwani2151@gmail.com
- **Matrix:** via the Kusama ZK Bounty channel
- **Code repository:** [github.com/jatinsahijwani/zk-polka-sdk](https://github.com/jatinsahijwani/zk-polka-sdk)
- **Demo repository:** [github.com/AnirudhSingh07/web3test](https://github.com/AnirudhSingh07/web3test)

---

## Document changelog

| Version | Date | Notes |
|---|---|---|
| v1 | Initial release | Response to ten async follow-up questions from the curator call |

This doc will be updated as scope evolves or further clarifications are requested.
