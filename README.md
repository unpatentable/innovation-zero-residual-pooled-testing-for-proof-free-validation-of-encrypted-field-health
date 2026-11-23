# Zero-Residual Pooled Testing for Proof-Free Validation of Encrypted Field Health Data

> **Note:** This README is an ultra-condensed summary of the research reports published by Unpatentable.org. To view the full innovation documentation and explore the complete research, use the links at the bottom of this document.

**Domain:** [Digital & Information Technologies](https://unpatentable.org/innovation/?domain=digital-information-technologies)
**Subdomain:** [Secure Computing](https://unpatentable.org/innovation/?domain=digital-information-technologies&subdomain=secure-computing)

---

### Why We Invented This Technology
Recent maturation of NIST-standardized post-quantum cryptography (ML-KEM, ML-DSA), practical fully homomorphic encryption with hardware acceleration, and efficient zero-knowledge proof systems has crossed the performance thresholds needed for low-resource deployment for the first time. Humanitarian organizations, rural clinics, and refugee-serving agencies hold highly sensitive data (health records, identities, aid distribution) on cheap devices and weak networks, and 'harvest now, decrypt later' attacks put this data at risk, while privacy-preserving computation was previously too heavy to run on such hardware.

### The Innovation
## Abstract
Zero-Residual Pooled Testing (ZRPT) lets low-cost, intermittently connected field devices submit post-quantum encrypted public-health records without producing any zero-knowledge proof on the device. Each record is encrypted with lattice-based BFV under a threshold key as soon as it is captured. The server then evaluates a public degree-2 "residual" polynomial on the ciphertexts. This polynomial encodes surveillance edit checks such as one-hot validity, occupancy and logical exclusions, and it equals exactly zero for every valid record. Residuals are pooled with random weights under a non-adaptive group-testing design taken from Dorfman pooled screening. Pools are threshold-decrypted in a single round. Because honest residuals are identically zero, these decryptions reveal nothing about honest data. Malformed submissions are found with O(d·log N) decryptions instead of N.

## Background & Prior Work
Privacy-preserving aggregation of untrusted inputs usually makes the client prove its input is valid. Prio and its SNIPs (Corrigan-Gibbs & Boneh, NSDI 2017) and the IETF VDAF/Prio3 fully-linear proofs need client-side proof generation and two non-colluding servers. Lattice-based ZK proofs (Lyubashevsky–Nguyen–Plançon, CRYPTO 2022) can prove that an RLWE ciphertext is well formed, but they add seconds of compute and tens of kilobytes per submission on weak hardware (estimate). Multiparty/threshold RLWE (Mouchet et al., PETS 2021; Lattigo; OpenFHE) aggregates honest inputs well but does not validate them. FedGT (arXiv 2305.05506) uses group testing to find malicious federated-learning clients through statistical quality tests on group aggregates. Armadillo (arXiv 2511.10863) recursively splits batches of failed client-proof verifications.

ZRPT differs on three points. Clients send no proofs. Validity is checked by an exact algebraic residual evaluated homomorphically, so it is cryptographically sound rather than statistical. The residual is identically zero for honest inputs, which is what makes decrypting a pool leak nothing about them.

## Competitive Landscape & Strategic Edge
Current tools sit at the extremes. DHIS2, ODK and KoboToolbox protect data in transit and at rest, but aggregators see plaintext. Commercial HE vendors and MPC platforms (Duality, Zama, Inpher-class offerings) assume well-provisioned clients or trusted inputs. Prio-style systems (ISRG Divvi Up) assume clients can compute proofs and that two operators exist. Three things are missed:
1. In BFV with prime t, the validity predicates used in surveillance edit checks are low-degree polynomials whose zero set is exactly the valid set.
2. Decrypting a quantity that is identically zero for honest users is a disclosure-free act.
3. Epidemiology's pooled screening already gives the optimal test-count theory.

ZRPT moves the whole proving burden to servers, removes per-device proofs, and cuts threshold-decryption "disclosure events", which are costly and coordinated across organizations.

## Core Concept
**1. Device side: encrypt at capture with homomorphic slot packing.** Parameters are BFV with ring dimension n = 8192, ciphertext modulus log₂q ≈ 218 bits (within the HomomorphicEncryption.org 128-bit table), and plaintext prime t = 786433 = 3·2¹⁸+1. Since t ≡ 1 mod 2n, all 8192 slots are available for SIMD batching. Each record fills a block of w = 64 slots: one occupancy slot o_b and several one-hot field groups G (age band, sex, syndrome code, pregnancy, outcome, district code split into digit groups), so one ciphertext holds 128 records. Workflow:
- The app runs its normal plaintext edit checks, then encrypts the record under the joint threshold public key as a fresh ciphertext whose only non-zero slots are in the record's block.
- It adds that ciphertext to a running per-epoch accumulator A.
- It wipes the plaintext and the fresh ciphertext.

The device can never decrypt A, so a seized phone exposes nothing it has collected. Noise grows only additively (about √128). At sync, the device uploads A (about 446 KB uncompressed, estimate 2·8192·218 bits) with a header (device ID, epoch, sequence) signed with ML-DSA-44 (FIPS 204). The upload is appended to a Merkle transparency log. Device work per record is one RLWE encryption (NTT-dominated; tens of ms on low-end ARM, estimate) and one addition.

**2. Zero-residual polynomial.** For a submission x ∈ Z_tⁿ, the server computes the ciphertext Z = Enc(R(x)). Each slot of R is a public polynomial of degree ≤ 2:
- Booleanity in every slot: x_i² − x_i.
- Group consistency at a designated slot of each group: (Σ_{i∈G} x_i) − o_b. One-hot groups sum to the occupancy bit, so empty blocks must be all zero.
- Exclusion rules, e.g. pregnant ∧ male: x_u·x_v.
- Implication rules, e.g. outcome-recorded ⇒ occupied: x_u·(1 − x_v).

Products and sums across slots use a small fixed set of rotations (Galois keys for block-internal offsets only) and plaintext masks. Because t is prime, x(x−1) = 0 has roots only 0 and 1. So R(x) = 0 holds exactly when x satisfies every edit check, giving the residual this property:

 **R(x) = 0 ⇔ x is valid.**

The cost is one ciphertext–ciphertext multiplication with relinearization per submission. A ciphertext with abnormal noise, such as a reaction-attack probe, decrypts its residual to pseudo-random non-zero slots, so it is flagged as well.

**3. Pooling with post-commitment randomness.** After the epoch's log closes, the k threshold decryptors (e.g., Ministry of Health, WHO country office, an NGO) run commit-reveal on a seed σ bound to the log's Merkle root. Test membership and weights come from SHAKE256(σ‖τ‖j). Test τ includes submission j with probability p = 1/(d̂+1) (a Bernoulli design). The pool ciphertext is

 S_τ = Σ_{j∈P_τ} ρ_{τ,j}·Z_j,  with ρ_{τ,j} uniform in Z_t*,

using scalar plaintext multiplication, which grows noise by at most a factor t. Each slot is decrypted separately. If any member has a non-zero residual in slot s, then S_τ[s] is uniform, and Pr[S_τ[s] = 0] = 1/t ≈ 1.3·10⁻⁶ per test. Weights are fixed after submission, so colluding devices cannot arrange cancellation. Honest members add exactly 0, so the decrypted vector is a function of malformed inputs only.

**4. Single-round decoding using epidemiological pooling theory.** The design parameter d̂, the expected number of malformed submissions, is set from previous epochs' observed "prevalence", just as Dorfman pool size is set from disease prevalence. The test count is T ≈ e·d̂·ln N (the Bernoulli/COMP bound; Aldridge–Johnson–Scarlett). Decoding uses COMP: any submission that appears in a negative pool is cleared, and the rest are flagged. All T pools are decrypted in one coordinated batch, which matters when decryptors are offline HSM custodians in different time zones. An optional second round runs Definite-Defectives decoding or tests the few ambiguous items individually. Inside a flagged accumulator, block-level pooling (multiplying by plaintext block masks) finds the bad record blocks. The honest blocks are kept by mask-multiplying the bad blocks to zero. Example estimate: N = 20,000 device-weeks and d̂ = 10 give T ≈ 270 pool decryptions instead of 20,000, roughly 74× fewer.

**5. Decryption gating and lattice ZK.** No decryptor trusts the aggregator's choice of ciphertexts.
- **Replicated deterministic evaluation:** each decryptor rebuilds every S_τ from the public log and σ (BFV evaluation is deterministic). It partially decrypts only ciphertexts whose hash matches its own computation, so a malicious server cannot get an individual's raw ciphertext decrypted.
- **Proof of correct partial decryption:** each partial decryption d_i = c₁·s_i + e_i^sm comes with a lattice ZK proof (LNP22-style linear relation plus norm bound). The proof shows d_i is consistent with the committed key share and that ‖e_i^sm‖∞ ≤ B_sm. This stops a rogue decryptor from framing honest devices or hiding poisoners, and lets lightweight auditors check results without recomputing.

Smudging uses about 40 bits of statistical margin. Estimated noise budget: about 190 bits fresh minus about 45 (ct×ct) minus 20 (scalar ρ) minus 11 (pool summation) minus 5 (rotations) leaves more than 100 bits for smudging. This needs confirming with the library's noise estimator.

**6. Analytics release.** Validated accumulators are summed, and blocks are folded by log₂128 = 7 rotations into district × age × syndrome cross-tabs. Each decryptor adds an encrypted share of discrete-Gaussian or Skellam noise before joint decryption, so the release is differentially private. Small-cell suppression (cells < 5) is applied after the noise, as post-processing. Every decryption, whether pool or release, is a logged, purpose-bound event.

## Physics & Logical Feasibility
ZRPT relies on standard RLWE hardness (BFV at HE-standard parameters) and Module-LWE/SIS (ML-DSA, LNP22 proofs). Correctness depends on Z_t being a field (t prime), so booleanity roots are exactly {0, 1}, and on the Schwartz–Zippel bound for random weights (error 1/t per slot per test). Noise growth figures are textbook BFV bounds and are labelled estimates. The single critical assumption is threshold honesty: no coalition at or above the decryption threshold colludes. Smudging must be sized to hide noise that depends on the secret key. The noise flagging of step 2 does not stop the 1-bit decryption-failure leakage of crafted ciphertexts. It is mitigated by per-epoch key rotation, flagged-device revocation and rate limits. For high-threat deployments, an optional per-device proof of short encryption randomness can be added, at a stated cost in compute and bandwidth.

## Implementation Readiness & Key Constraints
1. **Months 0–9:** prototype on Lattigo multiparty BFV or OpenFHE threshold BFV; write a residual compiler that turns DHIS2 validation-rule syntax into degree-2 slot circuits; profile noise.
2. **Months 9–18:** build an Android client (an ARM64 native library inside DHIS2 Capture or ODK Collect), the transparency log, the decryptor daemon with replicated evaluation, and the LNP22-style partial-decryption proofs.
3. **Months 18–36:** third-party cryptographic audit, a pilot with one ministry and two partner organizations, and data-protection impact assessments (GDPR Art. 25/35 and national health-data laws).

Constraints:
- Uploads of about 450 KB per device-week suit opportunistic 3G/Wi-Fi sync, but are slow over 2G (estimate 40–90 s).
- Validation rules above degree 2 need more depth, larger q, or bit-decomposition.
- Governance agreements among the decryptor organizations are on the critical path.

## Utility & Purpose
Humanitarian surveillance of displaced populations, people in key-population HIV programmes, survivors of gender-based violence, and people in outbreak zones collects line-list data whose exposure can endanger the people in it. Today, aggregators see plaintext and field devices hold readable records. ZRPT lets cheap Android phones encrypt each record against quantum-capable adversaries at capture. It keeps poisoned or tampered submissions out of epidemic indicators without making phones produce proofs, and it reduces decryptions to a small number of auditable events that reveal nothing about honest data. Beneficiaries include ministries of health, WHO/UNHCR/UNICEF-type agencies, NGOs running community-health-worker networks, and the vulnerable individuals in the data.

## Potential Applications & Impact
- **Public health:** syndromic surveillance, IDSR weekly reporting, cross-border outbreak cross-tabs, and key-population programme monitoring.
- **Other domains:** encrypted smart-meter or census-style tabulation, encrypted federated-learning telemetry with bounded or one-hot features, and election-observation tallies.
- **Methodological ripple effect:** the "zero-residual + pooled decryption" pattern converts validity checks into disclosure-free events. It gives cryptographers an alternative to client-side SNIPs and FLPs wherever validity is low-degree. It also lets epidemiological group-testing theory (prevalence-adaptive pool sizing, non-adaptive designs) be used for cryptographic fault localization.
- **Long term:** the pattern helps make post-quantum, end-to-end-encrypted health information systems the default rather than an exception, and fits the planned migration to NIST post-quantum standards.

## Market/Adoption Feasibility (If Applicable)
Estimated development cost is $3–4.5M over three years: about 8 engineers and cryptographers, audits and a pilot. That is well under the $8M ceiling. Device cost is zero marginal, since it reuses existing ≥$60 Android phones. Server cost is modest (estimate: 20,000 relinearized multiplications at tens of ms each is minutes of CPU). Adoption is helped by open-source HE libraries, the DHIS2 footprint in more than 70 countries, NIST FIPS 203/204, and donor pressure for data-protection-by-design. The barriers are:
- legal agreements among threshold-key custodians;
- HE parameter literacy among ministry IT staff;
- the absence of a ratified ISO FHE standard, which may slow procurement approvals.

## Risk & Counterarguments
- **Coverage limit:** ZRPT checks format and logical consistency, not truthfulness. A compromised device can still submit valid but false records. It must be paired with rate limits per credential and statistical anomaly surveillance.
- **Collusion:** a threshold coalition breaks confidentiality.
- **Reaction attacks:** these leak about one bit per crafted submission if key rotation is neglected.
- **Thin margins:** noise budgets must be verified with each library, because an error there silently corrupts results.
- **Bandwidth:** large ciphertexts may exclude 2G-only sites.
- **Ethics and law:** cryptographic "unreadability" must not be presented as licence to over-collect. Some jurisdictions may require plaintext access by health authorities, which conflicts with threshold governance.

## Open Questions & Future Research Directions
- Higher-degree edit checks (range constraints, date arithmetic) via bit-sliced encodings or BFV/TFHE hybrids.
- Shrinking uploads with server-side ring packing or a hybrid scheme with an RLWE-friendly symmetric cipher, without breaking residual soundness.
- Formal UC treatment of combining pool decryption and DP release.
- Optimal adaptive and non-adaptive pool designs when decryptor availability is intermittent.
- Extending zero-residuals to lattice anonymous-credential attributes, so that beneficiary eligibility becomes a pooled, proof-free check.
- A formally verified noise-budget compiler from DHIS2 rules.
- Field measurement of device energy and sync latency.

## Novelty Threshold Justification
**Medium novelty**, as a unique cross-domain synthesis with a specific new mechanism. The closest prior art:
- FedGT ([arXiv 2305.05506](https://arxiv.org/abs/2305.05506v3)) group-tests clients using statistical tests on aggregates.
- Armadillo ([arXiv 2511.10863](https://arxiv.org/html/2511.10863v2)) recursively splits failed client-proof batches.
- Prio/VDAF need client proofs.
- Chatel et al. ([arXiv 2207.14071](https://arxiv.org/pdf/2207.14071)) verify server computation, not client inputs.

The searches found nothing combining these elements:
1. Homomorphically evaluated, exactly-zero-for-honest degree-2 residuals that encode surveillance edit checks, with no client proof.
2. Random-weighted, non-adaptive Bernoulli group-testing pools threshold-decrypted in one round.
3. Decryptor gating by replicated deterministic evaluation plus lattice ZK partial-decryption proofs.
4. Encrypt-at-capture slot-packed accumulators on the device.

## References & Prior Art Checks
- Corrigan-Gibbs & Boneh, "Prio," NSDI 2017; IETF CFRG VDAF draft.
- Mouchet et al., "Multiparty Homomorphic Encryption from RLWE," PETS 2021; Lattigo; OpenFHE.
- Lyubashevsky, Nguyen, Plançon, CRYPTO 2022.
- Dorfman, Ann. Math. Stat. 1943; Aldridge, Johnson, Scarlett, "Group Testing: An Information-Theory Perspective," 2019.
- Albrecht et al., HE Security Standard 2018; NIST FIPS 204.
- FedGT: https://arxiv.org/abs/2305.05506v3
- Armadillo: https://arxiv.org/html/2511.10863v2
- Chatel et al.: https://arxiv.org/pdf/2207.14071

**Prior art searched:**
1. "group testing identify malicious clients secure aggregation homomorphic encryption invalid inputs". Used: FedGT (arxiv.org/abs/2305.05506v3), Armadillo (arxiv.org/html/2511.10863v2), Willow (eprint.iacr.org/2024/936).
2. "server-side homomorphic input validation encrypted client data random linear combination threshold decryption instead of zero-knowledge proof". Used: ePrint 2026/2422 (eprint.iacr.org/2026/2422.pdf), Chatel et al. (arxiv.org/pdf/2207.14071), US 12067129. No anticipation found.
3. "BFV homomorphically evaluate x(x-1) validity check one-hot encrypted client inputs no client proof poisoning aggregation". Used: DHSA (arxiv.org/pdf/2208.07189), arXiv 2608.23396, arXiv 2603.03398. No anticipation of the core mechanism found.

---

### About the Reports
Each innovation on Unpatentable.org is published as a three-part open documentation set:
- **Innovation Report** – the master blueprint describing the first principals concept, mechanism, and rationale behind the invention.
- **Implementation Guide** – a step-by-step, practical manual detailing how to build or prototype the innovation. Includes estimated cost breakdown.
- **Industry Impact Report** – an analytical overview exploring the potential economic, industrial, and societal effects of the innovation.

Together, these documents form a complete public disclosure package — enabling researchers, developers, and policymakers to understand, replicate, and build upon each invention.

---

### Explore this Innovation
- [Innovation Report](https://unpatentable.org/innovation/zero-residual-pooled-testing-for-proof-free-validation-of-encrypted-field-health-data)
- [Implementation Guide](https://unpatentable.org/innovation/implementation-guide-zero-residual-pooled-testing-for-proof-free-validation-of-encrypted-field-health-data)
- [Industry Impact Report](https://unpatentable.org/innovation/industry-impact-report-zero-residual-pooled-testing-for-proof-free-validation-of-encrypted-field-health-data)

### Download PDFs
- [Innovation Report (PDF)](https://unpatentable.org/wp-content/uploads/2026/11/Zero-Residual-Pooled-Testing-for-Proof-Free-Validation-of-Encrypted-Field-Health-Data.pdf)
- [Implementation Guide (PDF)](https://unpatentable.org/wp-content/uploads/2026/11/Implementation-Guide-Zero-Residual-Pooled-Testing-for-Proof-Free-Validation-of-Encrypted-Fi.pdf)
- [Industry Impact Report (PDF)](https://unpatentable.org/wp-content/uploads/2026/11/Industry-Impact-Report-Zero-Residual-Pooled-Testing-for-Proof-Free-Validation-of-Encrypted-.pdf)

---

**Timestamps:**
- [View Main Innovation Report on Arweave](https://arweave.net/LF8Q1xm95eV29jy3WlHLBslKOYtI4bVuAfs4DJeXLIo)
- [View Implementation Guide on Arweave](https://arweave.net/145yKas3ZpedVIcDJn4WrZhnX5843lYiJjCLTI9g91U)

**Suggested Citation:**
> Unpatentable.org (2026). *Zero-Residual Pooled Testing for Proof-Free Validation of Encrypted Field Health Data.* Defensive Publication Series.
> https://unpatentable.org/innovation/zero-residual-pooled-testing-for-proof-free-validation-of-encrypted-field-health-data

---

```json
{
  "domain": "Digital & Information Technologies",
  "subdomain": "Secure Computing",
  "published": "2026-11-23",
  "arweave_main_tx": "LF8Q1xm95eV29jy3WlHLBslKOYtI4bVuAfs4DJeXLIo",
  "license": "CC BY 4.0",
  "arweave_implementation_tx": "145yKas3ZpedVIcDJn4WrZhnX5843lYiJjCLTI9g91U",
  "value_unlocked": "$72B"
}
```

Unpatentable.org is a public library of timestamped defensive publications —
open innovations released into the commons to prevent monopolization, democratize knowledge, and accelerate human progress.

License: CC BY 4.0
