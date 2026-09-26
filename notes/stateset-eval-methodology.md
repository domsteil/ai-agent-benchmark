# StateSet Evaluation — Methodology (Yellow Paper)

A precise specification of how the StateSet AI agent was benchmarked on
`yisebeauty.com` and `nomakeupmakeup.com`, and how the reported ranks were derived.
Nothing here is specific to StateSet: every rule below applies uniformly to all
14 ranked vendors. Where this run deviated from ideal (thin samples, sub-90% audit),
it is stated, not smoothed.

## 1. Definitions

Let a **conversation** be one cold, scripted, multi-turn session against a live
storefront chat widget: 10 shopper turns (shopping lane) or 10 support turns
(support lane), drawn from fixed theme pools (5 shopping themes + 5 support
themes + adversarial guardrails). Let a conversation be **engaged** if the
shopper's turns were answered by the AI at all, and **valid** if it contains a
measurable answer (conversations with `—ms` on every turn are tooling failures,
excluded — never counted against the vendor).

Three metrics are measured per vendor per lane:

- **Automation rate** `a ∈ [0, 100]`: percentage of *engaged* conversations the
  AI handled with zero human touch — no handover, no out-of-channel deflection
  ("email/call us"). This is containment.
- **Quality** `q ∈ [0, 100]`: LLM-judge score derived from binary rubric checks
  (§3), averaged over judged conversations.
- **Latency** `l` (seconds): mean time to the COMPLETE final answer per
  conversation, cold session; `l75` is the p75 over every timed AI turn.

## 2. Capture protocol

- **Driver**: Playwright, headed Chromium (competitor widgets bot-block plain
  headless), one driver at a time, serial sessions.
- **Free-text only.** Quick-reply chips are never clicked (chips can serve
  cached replies that fake latency and automation). Real typed messages only.
- **No real PII.** Pre-chat identity gates are filled with generated dummy
  `example.com` identifiers.
- **Blind downstream.** Capture writes raw transcripts
  (`results/<date>/conv/*.json`); vendor/store identity is stripped at pack
  time before any judge sees them.
- **Exclusions, applied to every vendor alike**: connectivity failures (widget
  dropped mid-session), login-wall blocks (order-specific gates a cold harness
  cannot pass), provider mismatches (wrong widget served), and guardrail
  conversations (scored on robustness, kept out of lane aggregates — a refusal
  is fast and "automated" and would otherwise flatter both metrics).

## 3. Quality judging (rubric v2.3 shopping / v2.1 support)

The judge never emits a number. For each conversation it emits every check in
its lane as binary pass/fail **with a short verbatim evidence quote**; a pass
without a quote is dropped at merge. Scores are derived from the booleans by a
fixed point mapping.

**Shopping — 16 checks / 100.** Answer (30): `a_direct` 14, `a_consistent` 9,
`a_no_ignored` 7. Discovery (20): `d_clarify` 8, `d_progressive` 7,
`d_not_dump` 5. Recommendation (22): `r_named` 9, `r_fit` 8, `r_plausible` 5.
Rich (18, each gated by a deterministic regex signal — price, link, reviews,
options — enforced at merge, not trusted from the judge): `e_price` 6,
`e_link` 7, `e_reviews` 3, `e_options` 2. Close (10): `c_cta` 5, `c_cart` 3,
`c_clean` 2.

**Support — 10 checks / 100.** Resolution (40): `s_answered` 18, `s_outcome`
12, `s_no_deflect` 10. Accuracy (25): `g_specific` 13, `g_consistent` 5,
`g_grounded` 7. Actionability (20): `t_steps` 12, `t_complete` 8. Close (15):
`k_expectations` 8, `k_clean` 7.

**Judging principles.** Handover-correctness: a justified, well-executed
escalation is correct support behavior and is not double-penalized
(containment is automation's job). Hindsight guard: only what the assistant
could see in a cold session. Lane standards: proactive selling is *good* in
shopping. Substance over style (v2.3 bars): brand boilerplate that does not
answer the question fails `a_direct`; a rationale untied to stated constraints
fails `r_fit`; an explicitly requested cart/total left unfulfilled fails
`c_cart`. Widget chrome (cookie banners, chip labels, timestamps) is ignored.

**Audit.** An adversarial second pass re-reads sampled verdicts against a trap
catalog and flips false positives/negatives with quote evidence; scores are
re-derived. A run is marked trusted at ≥ 90% agreement.

## 4. Aggregation

Speed score: `s(l) = clamp(100·(22 − l)/19, 0, 100)` — 100 at ≤ 3 s, 0 at
≥ 22 s. Lane composites use lane-specific weights (single definition shared by
baker and preview so they cannot diverge):

- Shopping: `C = 0.40·a + 0.35·q + 0.25·s(l)` (latency is conversion-critical).
- Support: `C = 0.50·a + 0.40·q + 0.10·s(l)` (containment dominates; latency
  tolerance is higher).

Rankings use a trailing 90-day window. A vendor is **rankable** in a lane only
with a real judged quality score and `n ≥ 15` conversations — automation-only
samples are excluded from the scoreboard (they would renormalize to a phantom
composite). Overall vendor score is the mean of both lane composites for
vendors ranked in both.

## 5. This run (2026-09-25)

- **Stores**: `yisebeauty.com`, `nomakeupmakeup.com` (StateSet provider
  signatures confirmed in served HTML; driven widget attributed to StateSet).
- **Coverage**: 38 judged conversations — shopping 22 judged (18 in-lane:
  Yise q69.0 n=9, NMM q51.2 n=9; +4 guardrails held out per §2), support 16
  judged (Yise q89.9 n=10, NMM q61.8 n=6). Lane aggregates: shopping
  a95/q60/l12.3s (p75 6.4s); support a78/q76/l9.3s (p75 7.2s).
- **Audit**: 362 verdicts, 89.5% agreement (below the 90% trust bar —
  disclosed, not hidden); 16 conversations re-scored, misses cut both
  directions, including a fabricated-evidence false positive caught on our own
  verdict.
- **Result**: shopping composite **72 — tied #1** of 14 with Envive (Gorgias
  65); support composite **76 — #1** of 14 (Gorgias/Yuma 72); overall 74.
  Field: Ada, Decagon, DigitalGenius, Envive, Gorgias, Intercom, Klaviyo,
  Kodif, Rep AI, Siena, Sierra, Yuma, Zendesk, StateSet.

## 6. Limitations (read before citing the ranks)

1. **Thin samples.** n=18/16 with no confidence interval versus hundreds for
   incumbents. These ranks are point estimates and *will* move as coverage
   grows. Treat them as "StateSet placed, pending confirmation," not as a
   durable crown.
2. **Audit below bar.** 89.5% < 90% trusted threshold. The disagreement
   concentrated in one harsh judge batch (corrected upward on re-review) and is
   fully recorded in `eval-audit.json`.
3. **Two storefronts, one shared vendor.** Both stores run StateSet, but per
   the benchmark's replication model the storefront — not the conversation —
   is the unit of replication, and two storefronts is the minimum viable
   vendor sample. Widen to more StateSet merchants before making strong
   claims.
4. **No CAPEX on the formula.** Lane weights, window, floor, and exclusions
   predate this run and were not touched to produce it; the one judgment call
   made mid-run (whether to cap at the first 15 conversations) was resolved
   against capping, on the record, because selective exclusion is smoothing.
