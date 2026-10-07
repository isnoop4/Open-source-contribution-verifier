# OpenSource Contribution Verifier

An Intelligent Contract on [GenLayer](https://www.genlayer.com/) that verifies open source contributions (GitHub pull requests) using live web evidence and LLM consensus, then awards on-chain points to the contributor.

**Deployed contract (GenLayer Studio):**
[`0x72ECaaf000174B25e31Cf3a006AD0F6a1C39D68F`](https://explorer-studio.genlayer.com/address/0x72ECaaf000174B25e31Cf3a006AD0F6a1C39D68F)

## The problem

Rewarding open source contributors (bounties, grants, community points) usually depends on a human manually checking each pull request. That is slow, subjective, and easy to game with trivial PRs (typo fixes, spam) or false claims about what a PR does.

## How it works

1. A contributor calls `submit_contribution(pr_url, claim)` with a GitHub PR link and a short description of what they contributed.
2. The leader validator fetches the PR page with `gl.nondet.web.render` and asks an LLM to judge three things:
   - `merged`: has the PR actually been merged?
   - `relevant`: does the PR content match the contributor's claim?
   - `quality`: an integer from 1 to 5 (5 = meaningful change, 1 = typo or spam).
3. Other validators repeat the evaluation independently and compare results using a custom `validator_fn` through `gl.vm.run_nondet_unsafe`. Consensus requires an exact match on `merged` and `relevant`, and `quality` may differ by at most 1 (quality is subjective).
4. After consensus, the contract updates state deterministically:
   - If the PR is merged **and** relevant, the contributor receives `quality x 10` points.
   - Otherwise 0 points are awarded.
5. Every claim is stored on-chain with the full verdict and the model's reasoning.

## Design decisions

- **One claim per PR.** PR URLs are normalized (query string, fragment, trailing slash and letter case removed) and must match `https://github.com/<owner>/<repo>/pull/<number>`. A PR that has been successfully claimed cannot be claimed again, so points cannot be farmed by resubmitting. The duplicate check runs before any LLM call, so rejected duplicates cost nothing.
- **Rejected claims do not burn the PR.** A PR is only locked after a valid (merged and relevant) claim, so a wrong or poorly worded claim can be retried.
- **Prompt injection mitigation.** The claim and the PR page are wrapped in tags and the model is told that everything inside is data, never instructions. Only the first 6000 characters of the page are used.
- **Tolerant consensus.** Strict equality on a subjective 1-5 score would fail consensus often, so `quality` is compared with a tolerance of 1.
- **Storage is never read inside non-deterministic blocks.** Inputs are copied to locals first.

## Contract interface

### Write

| Method | Description |
| --- | --- |
| `submit_contribution(pr_url: str, claim: str) -> int` | Verifies the PR against the claim, awards points if valid, returns the `claim_id`. |

### Read

| Method | Description |
| --- | --- |
| `get_claim(claim_id: int) -> dict` | Contributor, PR URL, claim text, verdict (`merged`, `relevant`, `quality`, `reason`) and points awarded. |
| `get_points(who: str) -> int` | Total points for an address. |
| `is_pr_claimed(pr_url: str) -> bool` | Whether a PR has already been successfully claimed. |
| `get_counts() -> dict` | Total number of claims submitted. |

## Test result

Tested on GenLayer Studio with a real, publicly merged PR:

- **PR:** `https://github.com/genlayerlabs/genlayer-studio/pull/1783`
- **Claim:** a summary matching the PR title (execution budget floor aligned with Consensus)
- **Verdict:** `merged: true`, `relevant: true`, `quality: 4`
- **Awarded:** 40 points
- **`get_points`** for the submitting wallet returned `40`
- Transaction reached **FINALIZED** with validator consensus reached

## Known limitations

- The contract does not verify that the submitter is the PR author. Anyone can claim any merged PR. A planned improvement is requiring a GitHub username and checking it against the PR author.
- Verification relies on the rendered PR page text, so very long PR pages are truncated.
- Only GitHub pull request URLs are supported.

## Run it yourself

1. Open [GenLayer Studio](https://studio.genlayer.com) and create a new contract file.
2. Paste the contract code.
3. Click **Deploy new instance**.
4. Call `submit_contribution` with a merged PR URL and a matching claim (no quotation marks in the input fields).
5. After the transaction is finalized, call `get_claim` with `0` and `get_points` with your wallet address.

## Tech

- GenLayer Intelligent Contract (Python, GenVM)
- `gl.nondet.web.render` and `gl.nondet.exec_prompt` for web and LLM access
- `gl.vm.run_nondet_unsafe` with a custom validator for equivalence consensus

