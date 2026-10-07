# What Two Weeks of Running an AI Minecraft Bot Taught Me About Reliability

*A project retrospective — October 2026. The full system: a Paper Minecraft server, an LLM agent (Mindcraft/Mineflayer fork), and a mixture-of-experts model with 3B active parameters running entirely on a local NPU — zero cloud, zero API bills. The goal: have the bot play Minecraft from survival to... well, as far as anyone honestly can.*

---

## The honest starting point

Nobody's AI has beaten Minecraft. Not autonomously, anyway. The only zero-human-input run to the Ender Dragon is the famous Baritone speedrun bot — brilliant, but a scripted task machine, not a deciding agent. The research frontier (Voyager, GITM, Optimus) gets to diamonds and admits in print that open-ended play is unsolved. The prediction-market crowd (Manifold) puts a fully autonomous dragon kill before 2027 at ~73% — and keeps tightening the rules: as of September 2026, random seeds are mandatory and `keepInventory` is disqualifying. (Our stack uses keep-inventory. We'd fail those criteria today, and we say so up front.)

So this isn't a "we beat the game" post. It's a "here is what actually breaks when an AI runs unattended, and what we built to break less" post. Turns out that's the more interesting problem — and it generalizes way past Minecraft.

## Week one: the bot was killing itself

Every ~90 seconds, our bot would exit and rejoin the server. The culprit: a watchdog. When the pathfinder got confused, a safety timer decided the right response was to kill the process. The watchdog had become the fault — textbook cascading failure, the same anti-pattern that takes down data centers, discovered here in a dirt hut at coordinate (14, 115, -32).

Fix #1 of many: **watchdogs cancel, never kill.** Cancel the stuck action, cool down, continue. The process is innocent until the *task* is proven stuck.

## The client is a liar

Minecraft bots work from a locally cached copy of the world. Ours repeatedly teleported into walls and suffocated. The forensic trail took days and included: the client model ghosting blocks at freshly-loaded chunk edges; a 1.21 block type (leaf litter) that reads as air to the client but is solid to the server; and — our favorite — a jar/world version mismatch that produced ghost blocks *silently*, no errors anywhere.

Fix: **every safety-critical check now asks the server directly** via single-condition probes, because the answer format itself lies in aggregate (a failed multi-condition command returns an empty reply, which naive code reads as "pass" or "fail" depending on mood). Generalized rule, applies to any agent on any external system: **your model of the world is a cache. Verification goes through the authoritative channel, or it didn't happen.**

## Success is a claim; verification is a receipt

Three incidents, one pattern:
- The shelter reported "sealed" by probes that were themselves broken. Two nights later we found the bot roofless. (Verification code is code. It fails too. Test the tester.)
- The death-recovery routine cheerfully reported "recovered items" while the inventory stayed empty. The agent's memory then planned around a phantom kit.
- The collect-block function returned success without ever producing a drop — then re-targeted the same ghost tree forever, because from its perspective nothing had gone wrong.

The fix that changed everything: **every step advances only on an independent readback** — inventory NBT queries, block-state probes — never on a function's return value. When we shipped it, the bot earned a full stone tool set in one session: first verified tool progress of the project.

This is the quiet scandal of agent work: most "the agent did it" claims are transcripts, not receipts. We adopted the standard the Mindcraft community uses for *proof-of-win* videos — no cuts, no teleports, no human input, published recording — and pointed it at ourselves instead.

## The delete-key incident

We ran an admin command to *read* the bot's inventory. It was the wrong command — it *deletes* items. Both verified kits, gone. Twice.

We codified a doctrine ("never use X as a read"), and — more importantly — recognized the lesson: **a doctrine in a document is a reactive fix; a guard in the tooling is a proactive one.** A 20-line read-only wrapper with a verb allowlist makes the mistake structurally impossible. That's the whole retrospective in one sentence: *every diagnosis should end in a guard the next session cannot step around, not a paragraph it has to remember.*

## The scheduler wars

Reactive survival modes (flee, unstuck, self-preservation), an LLM self-prompting loop, and a deterministic mission runner all wanted to control one action queue. The resulting interrupt wars took four sessions to end. The end state: one owner of the bot at a time, with a lease and a log — the single-writer principle, arrived at the expensive way.

Meanwhile the agent's own memory kept poisoning it: after a bad night, its notes read "pathfinding broken, collect oak_log manually," and the next model turn gamely complied with the bad advice. Memory that the agent can rewrite is an echo chamber. The right design — still on our roadmap, recommended three times, because *us too* — is durable goals outside the model's context, and failure notes that get quarantined rather than reread as truth.

## Where we actually are

Verified, readback-confirmed: full stone toolkit earned in-game, night survived unattended, crash-recovery that restarts mid-mission and rewinds itself when prerequisites vanish. Known limits, stated plainly: no planner exists beyond stone tier; navigation is teleport-assisted admin infrastructure (which we now count in an assist ledger rather than gloss over — the field distinguishes privileged access from human assistance, and so do we); the local model is a 3B-active MoE, which the literature says cannot drive open-ended planning — which is precisely why our architecture doesn't ask it to.

Small model + verified skill library + goal graph is the honest path for hobby compute, and it's the path the research backs: DECKARD showed decomposition quality beats model size; Voyager's skill library is externalized competence; GITM's agents never let the LLM write code, only compose vetted actions. We reinvented pieces of each, in production, in the wrong order, and can now cite the papers agreeing with our scars.

## What we'd do differently (the list that matters)

1. **A trusted-source rule on day one** — server is truth, client is cache. Would have deleted a week of suffocation deaths.
2. **An eval harness before features** — seeded worlds, scripted scenarios, programmatic checkers. Twelve sessions of "watch the logs and see" is how you get a fix cycle measured in days instead of minutes. (We prescribed this on day seven and didn't do it until this post made us admit it. Watch out for that — knowing the right practice and shipping it are different skills.)
3. **Patch management from day one** — hand-edits inside node_modules are silent time bombs. (This one we've since closed: patch-package plus a clean-clone `npm ci` verification now gates every dependency change.)
4. **Systemd units for every service** — our recovery script is good; fifteen-minute unit files are better.
5. **Circuit-breakers on recovery paths** — a recovery loop that assumes the thing it's recovering from will go right is an incident amplifier. Ours once looped a brain-dead bot back into a pillager camp 226 times.
6. **Decide your honesty policy before anyone asks** — assisted vs. autonomous runs, defined and logged in advance, so your claims survive your success.

## Build the pyramids to build Egypt

Kolby Nottingham — the Mindcraft project's first author — set a community goal in September 2025: beat Minecraft with Mineflayer bots, solved in chunks (obsidian, portal, blaze rods, stronghold, dragon), with a public no-cuts proof standard. He calls the pathfinder "essentially the AI": fix it and every other behavior rises.

That's the part of this project we're proudest of joining rather than owning. The bot didn't beat the game. What it did, over two weeks of very educational disasters, is build the machine that turns each disaster into a guard — progress-based watchdogs, receipt-not-transcript verification, single-writer ownership, server-truth probes, honest recovery messages that can say "VERIFIED STILL LOST."

Any agent that runs unattended long enough will meet every one of these failures. The only question is whether you meet them once, or every time.

---

**Appendix:** the full field taxonomy — seven compounding failure chains and the guard ledger mapping every diagnosis to the structure that closes it — lives in [FAILURE-TAXONOMY.md](FAILURE-TAXONOMY.md).

*System: Paper 1.21.11 · Mindcraft (MIT) fork · Mineflayer · FastFlowLM serving a Qwen MoE on an AMD XDNA2 NPU · deterministic mission runner with inventory-readback verification. No cloud services were consulted; the total inference bill was $0.*
