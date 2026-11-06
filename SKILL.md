---
name: trigger-test
version: 1.0.0
description: "Test whether your skills actually fire. Feed it synthetic user messages, it checks which skill SHOULD trigger vs which stays silent, and scores trigger accuracy per skill — catches the silent-failure case the official eval tool misses. 测试你的技能是否真的会触发：喂入模拟用户消息，逐个验证该触发的触发、不该触发的安静，输出每个技能的触发准确率报告——专治官方评测工具测不出的静默失灵。"
---

# Trigger Test

Your skill looks perfect on paper. Then a user types the exact phrase and… nothing fires. Trigger Test catches that **before** your users do: it simulates realistic user messages, applies trigger analysis per skill, and scores hit / miss / false-alarm for each.

## When to run

1. After publishing or updating any skill (pre-flight check)
2. When a skill "never seems to fire" or "fires at the wrong time"
3. After installing new skills alongside existing ones (regression check)
4. As a CI gate: run on save, fail on accuracy drop

## How it works

Trigger Test does NOT call any model. It evaluates the same thing a host model reads when deciding to load a skill: the `description` in each SKILL.md frontmatter. For every test message, the agent runs a structured 4-question analysis per skill (see references/analysis-protocol.md):

1. **Intent match** — does the message's core intent fall in this skill's claimed scenarios?
2. **Keyword evidence** — which trigger phrases overlap?
3. **Competition** — if another installed skill claims the same scenario, who wins?
4. **Verdict** — TRIGGER / SILENT / CONTESTED (both claim it)

The protocol is deterministic in structure (same 4 questions every time) so results are reproducible across sessions.

## Test suites

Two sources of test messages:

**A. Generated suite** (fast, zero setup)
From each skill's own description, the agent derives:
- 3 positive messages a real user might type (should trigger)
- 3 negative messages from NEIGHBORING intents (should stay silent — the hard cases)
- 1 ambiguous message (expected CONTESTED if a competitor exists)

**B. User's real messages** (ground truth)
The user pastes 5-10 messages they actually sent recently where they wished a skill had (or hadn't) fired. These carry the most signal — prioritize them.

## Scoring

```
hit        = should trigger & verdict TRIGGER
miss       = should trigger & verdict SILENT      ← the killer failure
false-alarm= should be silent & verdict TRIGGER   ← the annoying failure
contested  = two skills both claim, no winner declared

accuracy = hits / (hits + misses)
precision = hits / (hits + false-alarms)
```

Score each skill, then the whole installation. Full report template in `references/report-template.md`.

## What to do with the report

- **miss** → the description's trigger phrases don't cover how users actually phrase it; add the missed phrasing (verbatim from the test message) to the description
- **false-alarm** → the description over-claims; narrow it with an explicit "Do NOT use for…" sentence
- **contested** → run Skill Health Scan's overlap check; define a boundary sentence per skill
- After any description edit, re-run the same suite — scores must not drop

## Limits (honest)

- This evaluates **trigger selection**, not execution quality (a skill that fires correctly but runs badly needs a different test)
- Verdicts come from the agent's own reading of descriptions — same mechanism the host uses, which is the point: it surfaces what the host will actually see
- Small suites have noise; treat single misses as signals to investigate, not proof of failure
