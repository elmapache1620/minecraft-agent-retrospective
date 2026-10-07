# A Field Taxonomy of AI-Agent Failures (from two weeks of unattended bot operation)

*Appendix to "What Two Weeks of Running an AI Minecraft Bot Taught Me About Reliability." Every entry here was a real incident in a real deployment: a Paper Minecraft server (1.21.11), a Mineflayer-based LLM agent, and a 3B-active MoE served on a local NPU. Status marks: **✅ closed-and-verified** (fix proven by independent readback) · **⚠️ mitigated** (reduced, not eliminated) · **❌ open** (diagnosed, not yet structurally guarded).*

The unit of analysis is not the bug. Forty-one bugs is a maintenance story. **Seven causal chains** is the actual story — each starts from one early decision or shortcut and pays interest for days.

---

## Part I — The seven compounding chains

### 1. Client-model trust → embed deaths → fork the world
**Chain:** teleport-based navigation made every landing depend on a placement check — but every check read the *client's cached world model*, which ghosts blocks at freshly-loaded chunk edges and read one 1.21 block type as air while the server treated it as solid. Each partial fix reduced deaths without removing them, because the one remaining post-landing check still trusted the client. The fixed home coordinates themselves had sunk into terrain, so even the health-guard teleport **home** buried the bot.
**Cost:** 30+ deaths in one day; ultimately a full world replacement.
**Guard:** one trusted-source rule, enforced in code: *the client model is a cache, never ground truth.* Every safety-critical probe goes through the authoritative channel (server-side block queries). Generalizes to any agent on any external system.
**Status:** ✅ (via server-truth probes) — but the rule should have been stated on day 1.

### 2. The recovery mechanism as kill-loop amplifier
**Chain:** death-recovery teleport was built with two unverified assumptions: the destination is safe, and pickup succeeded. When the model server crashed, the brain-dead bot died repeatedly and the *recovery itself* kept dropping it at a hostile-mob camp → **226 deaths in one lifetime.** Worse, the recovery routine asserted success without inventory verification, so the agent's memory planned around a phantom kit.
**Guard:** safety mechanisms need their own circuit breaker — max uses per episode, disable-on-brain-failure, counted in an assist ledger. A recovery loop that assumes the thing it's recovering from will go right is the classic incident amplifier (SRE watchdog-cascade, same shape).
**Status:** ✅ deaths stopped (honest recovery + keep-inventory); ⚠️ disable-on-brain-failure note never implemented.

### 3. Watchdog whack-a-mole
**Chain:** upstream watchdogs killed the process every ~90 seconds on pathfinder confusion — the watchdog *was* the fault. Fix: skip instead of kill → surviving reactive modes then kept interrupting mission work → interrupt wars across three subsystems; one suppression attempt caused two more deaths and was reverted.
**Guard:** re-derive what Erlang and Google SRE documented decades ago — trip on *no progress*, not on elapsed time; cancel-not-kill; restart budgets with quarantine; **one arbiter owns the actuator** (single-writer ownership with a lease and a grant log).
**Status:** ✅ ended in a mission lock; ❌ cost four sessions because each watchdog was patched separately instead of fixing the ownership question.

### 4. Destructive commands posing as reads
**The worst class, entirely self-inflicted:**
- An admin command run twice as an "inventory read" — **it deletes**. Destroyed the first verified tool set, twice.
- Broken probes (entity tags, empty command-chain replies) read as positive evidence → a false "shelter sealed" report that left the bot roofless through the mass-death night.
- The server returns an **empty reply for failed command chains** — overnight, every cell read "unstandable" and the probe code agreed.
- A destructive counting trick circulated as legit for a day before being re-banned.
**Guard:** *an observation tool that lies is worse than no tool; one that destroys is worse than a fire.* Read-only wrappers with verb allowlists; "empty reply = invalid" as a first-class tri-state (`unknown` is never success and never license-to-retry); single-condition probes only.
**Status:** ⚠️ doctrines exist; **no technical guard yet** — the lessons live in prose a future session has to remember. (This is the flagship example of the whole taxonomy.)

### 5. Memory-as-transcript poisoning
**Chain:** agent memory was conversational state the agent rewrites. A seeded mission was erased after a death; after a bad night the notes read "pathfinding broken, collect logs manually" and the next model turn obediently complied with the bad advice in a runaway loop; stale gear claims contradicted the server.
**Guard:** state the agent may not rewrite lives **outside the model's context** — durable task-state file immune to memory writers; failure residue quarantined rather than reread as truth; invariants pinned at the prompt front. (Voyager's skill library and event-sourced agent state are the literature's versions of the same answer.)
**Status:** ❌ open. Mitigated by pinned facts; the structural fix was recommended three times and never built.

### 6. The admin-assist double edge
**Chain:** privileged server access became the universal crutch — teleport, sweeps, repair, probes, reads. Every protocol quirk on that channel (auth empty-body, socket rejection of one language's client, invisible output, 160-char truncation, renamed rules, delete-on-read commands) injected false data or destructive side effects into the control loop. The false seal, the false-solid reads, and the deleted kit all arrived through this channel — which is *also* the project's genuine differentiator, since research agents can't use admin assist.
**Guard:** a crutch you depend on for survival is infrastructure, and infrastructure needs its failure modes handled: null/tri-state reads (unknown ≠ failed), one persistent client instead of thousands of spawns per night, sweeps that don't delete your own drops, and a published assist ledger.
**Status:** ⚠️ partially handled per-quirk; ❌ ledger open.

### 7. Version-pin ripple
**Chain:** pinning the server jar one version below the world data produced ghost blocks that masqueraded as behavior bugs for days (silent, no errors anywhere). Aligning versions then inherited a fresh trap set: an incompatible plugin, new protocol quirks, and one block type that became the literal agent of the death-loop in chain 1.
**Guard:** *a version decision is a data-format decision.* Never downgrade the server under newer world data; each platform-upgrade boundary gets its own probe session before the agent is trusted to it.
**Status:** ✅ aligned; ❌ the standing probe-before-upgrade ritual isn't formalized.

---

## Part II — Guard ledger: diagnosis → structure

The taxonomy only matters if each row ends in a guard a fresh session **cannot step around** — a wrapper, a harness, a unit file, a ledger — instead of a paragraph the next session has to remember to read.

| Diagnosis | Structural guard | Status |
|---|---|---|
| Watchdog suicides | Progress-based watchdogs: cancel-not-kill, cooldown | ✅ shipped |
| Success-without-evidence | Every step advances only on independent readback (inventory NBT, block-state probes) | ✅ shipped — first verified tool set earned same day |
| Interrupt wars | Single-writer mission lock with log proof | ✅ shipped |
| Client-model lies | Server-truth probe layer; replaceable-tag checks | ✅ shipped |
| False-positive probes | Empty reply = invalid; single-condition AND-ed probes | ✅ shipped |
| Death-recovery lies | Honest recovery that can report "VERIFIED STILL LOST" | ✅ shipped |
| Delete-on-read commands | Read-only RCON wrapper with verb allowlist | ❌ 20 lines, not built |
| Post-death amnesia | Durable `task_state.json` outside model context | ❌ ~50 lines, recommended 3×, not built |
| Unbounded crash-restart masks PathStopped | try/catch at the crash site + restart budget | ❌ open |
| Reboot kills the stack | systemd units for all four services | ❌ recovery script stands in |
| Hand-edits in node_modules | patch-package + clean-clone `npm ci` verification | ✅ shipped (post-review) |
| "Assisted vs autonomous" ambiguity | Published assist ledger + honesty tiers | ❌ owner decision |
| Vibe-observation fix cycles | Seeded-world eval harness with programmatic checkers | ❌ the highest-ROI unbuilt item |

**The one-sentence thesis:** we became very good at diagnosing failures in hours (reactive excellence); the entire remaining gap is converting each diagnosis into a guard *before* the next session can meet it — because any agent that runs unattended long enough will meet every one of these failures, and the only question is whether you meet them once, or every time.
