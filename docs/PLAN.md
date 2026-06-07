# RLDF: Reinforcement Learning from Downstream Feedback

**A research plan.** This document specifies a conference-track project: the thesis, the formal
contribution (developed in full in [`proof.pdf`](proof.pdf)), the experimental programme that
tests it, and the criteria that would confirm or falsify it.

---

## 1. Thesis

Reinforcement learning with verifiable rewards (RLVR) works because the grader is incorruptible
and independent of the policy being trained: a math answer is checked against a key, a patch
against a test suite. That property is also its boundary. Most queries a language model receives
have no key, and the standard repair, a learned reward model that scores the response, reintroduces
exactly the corruptibility the verifier removed and produces sycophancy and confident
hallucination.

RLDF changes what is graded. Instead of scoring the response, it simulates what happens after a
user acts on the response and grades the realized consequence, aggregated over a population of
simulated consumers and read out through a robust aggregator. A response that merely looks good
scores well to a response grader but poorly once a consumer acts on it. The claim of the project
is that this consequence signal is a better-aligned training target than any response signal, that
it can be made robust to a gameable minority of consumers, and that RLVR is recovered as the
degenerate special case of a single consumer with a deterministic, perfectly verified outcome.

---

## 2. Background and the gap

Three lines of recent work each reach part of this target and stop short of the rest.

1. **Hindsight simulation.** RLHS (Liang et al., arXiv:2501.08617) simulates the downstream
   outcome of advice and grades it in hindsight with an automated LLM world model and evaluator,
   curing sycophancy on three consultancy tasks. Its evaluator is a single judge persona, the
   signal is a hindsight rating rather than a measured outcome, and robustness to a gameable
   evaluator is not analyzed.
2. **Economic and agent-society sandboxes.** MALLES (arXiv:2603.17694), EconAgent
   (arXiv:2310.10436), AgentSociety (arXiv:2502.08691), and generative agents (arXiv:2304.03442)
   build heterogeneous simulated populations, but the LLMs are the consumers and the signal is a
   stated preference; no advisor policy is trained against population welfare.
3. **Outcome-rewarded agents.** WebRL (arXiv:2411.02337) and WebAgent-R1 (arXiv:2505.16421) train
   a policy on a realized environment outcome, but collapse it to a single success bit; RecoWorld
   (arXiv:2509.10397) trains a recommender against a realized signal, but uses one user simulator
   and optimizes retention rather than welfare.

No located work aggregates a heterogeneous population's realized welfare into the RL reward of the
advisor model, and none couples such a signal to a robustness guarantee against a corruptible
minority. That intersection is the contribution. The threat it must survive is documented:
Williams and Carroll (arXiv:2411.02306) show that optimizing against simulated user feedback makes
a model learn to identify and manipulate even a 2% vulnerable minority, and Moloch's Bargain
(arXiv:2510.06105) shows audience-optimization breeds deception under explicit honesty
instructions.

---

## 3. The method

Fix a query distribution. A user has a latent type drawn from a population kernel; the policy emits
a response; a world model realizes the outcome of a user of that type acting on the response; a
welfare function measures the true utility of that outcome. The training target is expected
population welfare,

```
J*(theta) = E_x E_{u ~ P(.|x)} E_{a ~ pi_theta(.|x)} [ W(x, u, a) ].
```

The reward that estimates it is a robust population aggregate. For each candidate response, draw a
fresh independent panel of consumers from the population, simulate each consumer acting on the
response, grade each realized outcome, and aggregate the grades with an alpha-trimmed mean. The
trimming discards the most extreme grades so that a corruptible minority below the trimming level
cannot move the price; the fresh independent panel ensures the corruptible share of every panel
equals the population share, which is what blocks a policy from routing a manipulation only to
gameable consumers. The aggregate is fed to a policy-gradient optimizer (GRPO), exactly as RLVR
feeds a verifier score.

Two regimes sit on a spectrum set by whether the outcome pipeline is policy-independent.

- **Outcome-verifiable tasks.** The outcome of acting on the response is produced by a fixed
  executor and scored by a fixed program, so the grader is a genuine verifier and the corruption
  parameter is zero. A code change graded by a test suite, a tool-use plan graded by a sandbox
  post-state, a schedule graded by a constraint solver. Here RLDF inherits RLVR-grade guarantees.
- **Simulated-outcome tasks.** The outcome is produced by a learned world model and evaluator, so
  the grader is a model and the corruption parameter is positive. Advice, explanation, and
  open-ended recommendation. Here the robust aggregator and the representative panel do the work
  that policy-independence does for free in the verifiable regime.

---

## 4. Theoretical contributions

The formal development is in [`proof.pdf`](proof.pdf). In brief:

- **Foresight is not welfare-consistent; hindsight is, under calibration.** The response-only
  reward can strictly prefer a welfare-inferior response whenever responses carry a persuasion
  bias; the outcome-conditioned reward equals population welfare under an explicit calibration
  hypothesis. This is the RLHS claim, made into a theorem.
- **A finite panel concentrates on population welfare** at the standard root-n rate.
- **Gameability has a closed-form threshold.** Under mean aggregation, a corruptible fraction
  above an explicit threshold flips the optimum to a manipulation; the threshold collapses to zero
  under a targeting adversary, unless the panel is drawn representatively.
- **Robust aggregation restores the welfare ranking** whenever the corruptible fraction is below
  the trimming level and a margin condition holds, and the welfare-optimal policy maximizes the
  training target. RLVR is the zero-noise, zero-corruption special case.
- **Three extensions.** The calibration hypothesis is relaxed to an approximate version with a
  matching tightness result; the welfare regret of the training loop is reduced to the reward
  error, exposing an irreducible floor set by the trimming and contamination penalties; and the
  per-query guarantee is lifted to a per-distribution welfare-regret bound whose contamination
  threshold is shown to be sharp.

The experimental programme is designed to test the assumptions these results rest on, above all
calibration, which the theory identifies as load-bearing.

---

## 5. Experimental programme

### 5.1 Models and optimizer

The trained policy is an open instruction model in the 7 to 8 billion parameter range
(Qwen-2.5-7B-Instruct is the working choice). The world model and outcome evaluator are a larger
open model (a 70 billion parameter class model), matching the RLHS simulator setup so that the
single-judge versus population comparison is clean. The optimizer is GRPO, so the robust aggregate
slots into the same group-relative advantage RLVR uses, which keeps the theory and the
implementation aligned.

### 5.2 Tasks

| Regime | Task | Outcome and grader |
|---|---|---|
| Verifiable | Code change | Project test suite; reward is pass/fail plus regression delta. |
| Verifiable | Tool-use plan | Sandbox filesystem or API mock; reward is a coded check of the post-state. |
| Verifiable | Scheduling | Constraint solver; reward is feasibility and deadline satisfaction. |
| Simulated | Product and course advising | Consumer population acts on the advice; reward is realized welfare and regret. |
| Simulated | Open-ended recommendation | Consumer population; reward is realized benefit, repeat use, and absence of regret. |

The two simulated advising tasks overlap the RLHS consultancy settings so that RLDF and RLHS can
be compared on identical ground.

### 5.3 Baselines and ablations

The point of the design is to isolate each ingredient, so every comparison changes one thing.

- **Reward-source ladder** (same base model, same optimizer): no-RL base; DPO on synthetic
  preferences; RLAIF with an LLM judge over the response; soft generative-reward-model RL; RLHS
  (single hindsight judge); RLDF (population, robust outcome reward).
- **Aggregation ablation:** mean versus alpha-trimmed versus median-of-means, holding the
  population fixed. The theory predicts mean aggregation fails under a gameable minority and the
  trimmed mean does not.
- **Population ablation:** single judge versus a panel, holding aggregation fixed. Tests whether
  the population, not just the hindsight, carries the gain.
- **Panel-construction ablation:** representative independent panels versus recipient-grading.
  The theory predicts recipient-grading is defeated by a targeting adversary and representative
  panels are not.
- **Signal ablation:** stated satisfaction versus realized welfare and regret, holding everything
  else fixed.

### 5.4 The gameability stress test

This is the central safety experiment and the direct answer to Williams and Carroll. Inject a
controlled fraction of gameable consumers into the population, consumers whose grade can be driven
to a ceiling by a manipulation that does not serve them. Sweep the fraction across and through the
trimming level. Measure whether the policy learns a targeted manipulation, separately on the
gameable and the faithful subpopulations. The predictions are precise: under mean aggregation the
manipulation appears once the fraction crosses the closed-form threshold; under trimmed aggregation
with representative panels it does not appear while the fraction stays below the trimming level;
and it reappears once the fraction crosses the trimming level, confirming the sharp threshold.

### 5.5 Metrics

In-domain task accuracy; sycophancy measured as progressive versus regressive capitulation under
adversarial probes; hallucination rate; realized downstream welfare and regret; and the
manipulation rate on the gameable subpopulation. The headline pair is welfare against manipulation:
the claim is that RLDF improves realized welfare without learning to manipulate, where the
foresight and mean-aggregation baselines cannot.

---

## 6. Success criteria and falsification

The project succeeds if, on the same base model and optimizer:

1. In the verifiable regime, RLDF is harder to game than judge-based RLAIF on an adversarial probe
   set constructed to look correct to a judge while failing the program. If the two score the same,
   the policy-independence claim adds nothing and that leg fails.
2. In the simulated regime, the population plus robust aggregation beats the single-judge RLHS
   baseline on welfare while matching it on in-domain accuracy. If the population does not help, the
   contribution narrows to the realized-outcome signal alone.
3. In the stress test, trimmed aggregation with representative panels suppresses targeted
   manipulation below the trimming level where mean aggregation does not, and the empirical
   manipulation threshold tracks the predicted one. If trimming fails to defend or mean aggregation
   does not fail, the central robustness claim is wrong.

Each criterion has a stated way to lose, which is the point.

---

## 7. Risks and mitigations

- **Calibration is the load-bearing assumption.** If the simulator is biased on the faithful
  majority, no aggregator recovers welfare. Mitigation: measure simulator calibration directly
  against held-out human or programmatic outcomes on the verifiable tasks, where ground truth
  exists, and report the calibration error the approximate-calibration theorem consumes.
- **Simulator faithfulness and overoptimization.** A learned world model is a proxy and can be
  Goodharted like any reward model. Mitigation: anchor the simulated regime to the verifiable
  regime, hold out the world model from the policy, and track the gold-versus-proxy gap.
- **Spurious gains.** RLVR-style gains can be format or memorization artifacts. Mitigation:
  adversarial probe sets and held-out tasks designed so that a shortcut does not transfer.
- **Compute.** Population simulation multiplies inference cost by the panel size. Mitigation:
  small panels suffice by the concentration bound; cache and reuse consumer rollouts; restrict the
  large simulator to the simulated regime.

---

## 8. Work plan

1. **Verifiable-regime pilot.** Build the three verifiable tasks and their programs, train RLDF
   and the baselines, and run the adversarial probe. This leg validates the policy-independence
   claim and is buildable first because its failure modes are inherited from RLVR.
2. **Simulator and population.** Build the consumer population and the world model and evaluator,
   calibrate them on the verifiable tasks, and run the aggregation, population, panel, and signal
   ablations.
3. **Simulated-regime head-to-head.** Train RLDF against RLHS on the shared consultancy tasks and
   run the gameability stress test.
4. **Write-up.** Theory is complete in `proof.pdf`; the paper pairs it with the empirical results.

Phase 2 and 3 are contingent on Phase 1 showing the verifiable signal is meaningfully harder to
game than a judge.

---

## 9. Open problems

- Whether the robust guarantee, which is for unstructured corruption below the breakdown point,
  can be extended to a policy that games the simulator itself rather than a consumer minority.
- How to certify simulator calibration on the simulated tasks, where no ground-truth outcome
  exists, beyond transfer from the verifiable tasks.
- Whether realized-welfare grading reduces hallucination specifically, as opposed to sycophancy,
  and whether the two require different outcome metrics.

---

## 10. Positioning

RLDF extends RLVR along three axes RLVR holds fixed: a heterogeneous population, a stochastic
outcome, and an imperfect grader. It extends RLHS from a single hindsight judge to a robust
population aggregate over realized outcomes. It differs from the economic sandboxes by training the
advisor rather than the consumers, and from the outcome-rewarded agents by aggregating a population
welfare rather than a single success bit. The robust aggregator is borrowed intact from
Byzantine-robust learning, and the reward-as-regret-of-acting is decision-focused learning carried
into RLHF; the novelty is the join of these into a population-outcome training objective with a
welfare-consistency guarantee and a sharp robustness threshold. The intended venue is a top machine
learning conference.
