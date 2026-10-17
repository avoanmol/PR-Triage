# PR-Triage
🛡️ PR Triage

A maintainer's shield against spam pull requests.

Local, read-only CLI that scores GitHub PRs on effort and value using deterministic signals + Gemini Flash. It recommends — you decide. It never posts to GitHub.

Quick Start
bash
pip install -e .
cp .env.example .env   # add GEMINI_API_KEY (required) + GITHUB_TOKEN (recommended)
pr-triage triage facebook/react --limit 10   # triage 10 open PRs
pr-triage triage facebook/react --pr 28456   # triage one PR
pr-triage demo                                # run on real Hacktoberfest PRs
How It Works
GitHub API → Signal Extraction → Evidence Packet → Gemini Flash → Policy Guard → Terminal Output
             (deterministic)      (structured)      (hardened)     (code-level)    (read-only)

Four signal extractors run locally — diff analysis, test detection, contributor profiling, issue linking — producing 15 structured signals. These go to Gemini Flash as an evidence packet. A post-classification policy guard enforces rules the LLM cannot override.

Verdicts
Verdict	Meaning	Output
substantive	Genuine effort — real code, tests, linked issue	✅ Public comment draft
minor	Small but legitimate — typo fix, doc edit	✅ Public comment draft
suspected-spam	Multiple strong signals converge	🔒 Private maintainer note
needs-review	Uncertain, incomplete, or conflicting evidence	🔒 Private maintainer note

Key rules:

suspected-spam requires ≥ 2 independent strong indicators. No single signal (account age, diff size, missing issue, first-time status) can trigger it alone.
Safe default is needs-review — used when evidence is missing, signals conflict, or confidence < 0.7. The fallback classifier can never return suspected-spam.
Public comments are drafted only for substantive and minor. The drafter raises ValueError for other verdicts — respectful by construction.
Signals
Category	Key Signals
Diff	files changed, lines added/deleted, extensions, docs-only, whitespace-only
Tests	test files changed/added, CI config touched
Contributor	account age, prior PRs (total + merged), public repos, first-time flag
Issue	linked issue exists, is open, labels

All fields use T | None — when an API call fails the classifier sees the gap and factors in uncertainty.

Privacy

The LLM receives: PR number, title, body (truncated to 2k chars), author, repo, and computed signal stats. Never sent: raw diffs, tokens, secrets, full issue bodies. A one-time privacy notice warns that private-repo metadata leaves your environment.

All PR text is treated as untrusted input — the system prompt instructs the model to ignore embedded instructions. Prompt injection attempts produce needs-review.

Development
bash
pip install -e ".[dev]"
pytest tests/ -v                          # full suite
pytest tests/test_verdict_policy.py -v    # policy enforcement
pytest tests/test_prompt_injection.py -v  # adversarial inputs
ruff check src/ tests/

Nine labeled fixtures in tests/fixtures/ cover: small valid fixes, first-time contributors, missing issue links, conflicting signals, prompt injection, clear spam, incomplete evidence, and veteran doc edits. False spam rate is tracked separately.

Project Structure
src/pr_triage/
├── models.py              # Pydantic models (Verdict, EvidencePacket, TriageResult)
├── cli.py                 # Typer CLI (read-only)
├── github_client.py       # GitHub REST API (GET-only)
├── signals/               # diff_analyzer, test_detector, contributor, issue_linker
├── classifier/            # engine.py (LLM + policy guard), prompts.py (hardened)
└── commenter/             # drafter.py (public drafts + private notes)

See agent.md for full coding conventions, verdict policy, and architecture details.