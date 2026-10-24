# OpenSource Contribution Verifier

An Intelligent Contract on [GenLayer](https://www.genlayer.com/) that verifies open source contributions (GitHub pull requests) using live web evidence and LLM consensus, then awards on-chain points to the contributor.

**Deployed contract (GenLayer Studio):**
[`0x31df5122FC99Cab803BD4246ED6a68c11D7ae45A`](https://explorer-studio.genlayer.com/address/0x31df5122FC99Cab803BD4246ED6a68c11D7ae45A)

## The problem

Rewarding open source contributors (bounties, grants, community points) usually depends on a human manually checking each pull request. That is slow, subjective, and easy to game with trivial PRs (typo fixes, spam) or false claims about what a PR does.

## How it works

1. A contributor can call `submit_contribution(pr_url, claim)` with a GitHub PR link and a short description of what they contributed.
2. The leader validator fetches the PR page with `gl.nondet.web.render` and asks an LLM to judge three things:
   - `merged`: has the PR actually been merged?
   - `relevant`: does the PR content match the contributor's claim?
   - `quality`: an integer from 1 to 5 (5 = meaningful change, 1 = typo or spam).. 
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

Tested on GenLayer Studio with real PRs from this repository.

**1. Authorized claim (credited)**
- PR: https://github.com/isnoop4/Open-source-contribution-verifier/pull/3
- The PR body declares `genlayer-address: 0xfD7d408f5Cc0A4d507bDc41c48B4A00A8C9Fc399`
- Verdict: merged = true, relevant = true, band = LOW
- Awarded: 10 points, credited to the declared address
- `get_points` for that address returned 10, `get_pr_binding` returned the same address, `is_pr_claimed` returned true
- Transaction reached FINALIZED with validator consensus

**2. Claim by a different sender**
- The same kind of claim was sent from a second wallet
- Points were still credited only to the address declared in the PR body
- `get_points` for the sending wallet stayed 0

**3. PR without a declared address**
- A merged PR without a `genlayer-address` line was claimed
- Awarded: 0 points, the PR was not locked and `is_pr_claimed` stayed false

## Known limitations

- Identity binding: points are credited only to the address declared by the PR author in the PR body (`genlayer-address: 0x...`). The contract reads the PR author and body from the GitHub API, and validators must agree on the author, address, merged status, relevance and band. If no address is declared, no points are awarded and the PR stays claimable.
- The wallet is not proven with a signature. The proof is the line in the PR body, which only the PR author (or a repository maintainer) can write.
- A PR author can only receive points for the address they declare. A third party who claims someone else's PR cannot redirect the points.
- Quality bands are judged by an LLM and are limited to NONE, LOW, MEDIUM and HIGH.
- One PR can be claimed once. Only claims that award points lock the PR.

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

