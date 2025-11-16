# CLAUDE.md

Guidance for Claude Code instances working on this CK3 mod.

---

## Core Philosophy

**Merges succeed. Builds succeed. Mods break.**

This repository never modifies `base/`, so `git merge base/1.XX.X` always succeeds cleanly with zero conflicts. But that proves nothing about mod compatibility.

**The standard: COMPREHENSIVE INVESTIGATION through critical thinking.**

Not following scripts. Not checking lists. **THINKING** about what could break.

---

## The Fundamental Problem

### Architecture Creates Invisible Risk

```
Other mods:
  - Modify base/ files
  - Git merge → conflicts when CK3 updates
  - Conflicts = signals of problems

This mod:
  - base/ = vanilla reference (never modified)
  - mod/ = all custom content
  - Git merge → always succeeds
  - NO signals when things break
```

### Real Breakage Is Silent

**Example from git history:**

CK3 1.13.0.4 changed 21,363 files. Paradox rewrote the entire scheme system:
- Removed 48 modifiers (power/resistance model)
- Added 24 modifiers (phase duration model)
- Completely different semantics

If this mod had been using the old system:
- ✅ Git merge: success
- ✅ Build: success
- ✅ Game load: success
- ✅ No error messages
- ❌ **8 perks completely non-functional**

Players would notice "schemes aren't faster" but couldn't diagnose why. No errors. No crashes. Just broken features.

**The mod survived because it was proactively updated (see CHANGELOG v1.10).**

### Your Responsibility

After EVERY CK3 update, you must answer:

1. **What changed?** (Not "how many files" - what SYSTEMS changed)
2. **Does the mod touch those systems?** (Discover, don't assume)
3. **Are dependencies still valid?** (Existence AND semantics)
4. **Do features still work?** (Functional testing, not just loading)

Cannot be answered by running a script. Requires **investigation and judgment.**

---

## Patterns of Breakage: What Actually Goes Wrong

### Pattern 1: Semantic System Reworks

**What happens:** Game systems redesigned with different mechanics, not just renamed.

**Example:** Scheme rework (v1.13)
- OLD: Accumulate power vs resistance until threshold
- NEW: Progress through phases with time duration
- Change type: Complete reimplementation

**How it manifests in diffs:**
```diff
- *_scheme_power_add/mult       (48 modifiers removed)
- *_scheme_resistance_add/mult
+ *_scheme_phase_duration_add   (24 modifiers added)
+ *_enemy_scheme_phase_duration_add
```

Mass deletions + mass additions of different pattern = **rework, not rename**.

**Failure mode:**
- Old modifiers silently ignored
- Features don't work
- No error messages

**How to detect:**
- Read diffs for pattern changes
- Understand if semantics changed (not just names)
- Test features functionally

**Critical insight:** Name similarities can deceive. `scheme_power` → `scheme_phase_duration` looks like a rename but is a different mechanic entirely.

### Pattern 2: Property Renames Within Same System

**What happens:** Modifier properties renamed but semantics unchanged.

**Example:** County control rework (v1.12)
```diff
- monthly_county_control_change_*
+ monthly_county_control_growth_*
```

All 8+ variants renamed from `change` to `growth`.

**How it manifests:**
- Systematic pattern across many modifiers
- Same values, different property names
- Usually documented in patch notes

**Failure mode:**
- Old property names silently ignored
- County control bonuses stop working

**How to detect:**
- Search for old property names in mod
- Check if vanilla still uses them
- Look for systematic renames (if one changed, others might too)

**Critical insight:** Even "simple" renames can be systematic. If you find one, search for the pattern.

### Pattern 3: Value Scale Changes

**What happens:** Numerical scales rebalanced, affecting magnitude.

**Example:** Health system rebalance (v1.12)
```diff
invigorated_modifier = {
-  health = 1
+  health = 0.25
}
```

Suggests health scale changed ~4:1 ratio.

**How it manifests:**
- Same property names
- Different value ranges
- Consistent ratios across many modifiers

**Failure mode:**
- Mod modifiers become overpowered/underpowered
- Balance breaks
- Percentages may be way off

**How to detect:**
- Compare value distributions across versions
- Look for consistent scaling ratios
- Check defines for max/min health changes
- Sample many modifiers to find pattern

**Critical insight:** No syntax error, no name change. Must compare VALUES and understand scales.

### Pattern 4: Age-Specific Differentiation

**What happens:** Generic properties split into context-specific variants.

**Example:** Health aging (v1.12)
```diff
- health = -2
+ health = -2.5
+ child_health = -1
+ elderly_health = -2
```

**How it manifests:**
- Old property still exists
- New age-specific properties added
- Vanilla now uses both

**Failure mode:**
- Mod still works but behaves differently
- Effects don't scale with age as intended
- Subtle balance issues

**How to detect:**
- Check if vanilla added age/context suffixes
- See if similar properties differentiated
- Compare mod's simple properties to vanilla's complex ones

**Critical insight:** NOT breaking in obvious way. Old code works but less precisely.

### Pattern 5: Complete Removal Without Replacement

**What happens:** Features deleted entirely, no equivalent.

**Example:** Event modifiers purged (v1.13)
```diff
- medicine_student_modifier = { ... }
- overturned_old_barbarity = { ... }
- trepanning_learning_penalty_modifier = { ... }
```

**How it manifests:**
- Entire modifier blocks deleted
- No corresponding additions
- Usually old/unused content

**Failure mode:**
- Events applying removed modifiers fail
- References to removed modifiers ignored
- Effects fail silently or with errors

**How to detect:**
- Check if modifiers/traditions still defined in vanilla
- Search git log for removals
- Look for orphaned references in vanilla events

**Critical insight:** If vanilla removed it, probably obsolete. But must verify no mod features rely on it.

### Pattern 6: DLC Content Gating

**What happens:** Existing features now require DLC checks.

**Example:** Buildings DLC-gated (v1.12)
```diff
scriptorium_01 = {
  can_construct_potential = {
    has_building_or_higher = temple_01
+   scope:holder = { has_dlc_feature = legends }
  }
}
```

**How it manifests:**
- New `has_dlc_feature` conditions
- `dlc_feature = X` flags in definitions
- Parameters gated in ethos/traditions

**Failure mode:**
- Features broken for players without DLC
- Buildings can't be constructed
- Governments unavailable

**How to detect:**
- Search for new `has_dlc_feature` additions
- Check if mod references DLC-gated content
- Test without DLC if possible

**Critical insight:** Works fine if YOU have DLC during testing. Players without DLC get broken content.

### Pattern 7: Trigger Logic Inversions

**What happens:** Boolean trigger logic flips meaning.

**Example:**
```diff
- immune_to_corruption_trigger = yes
+ immune_to_corruption_trigger = no
```

Same trigger name, opposite boolean value expected.

**Failure mode:**
- Logic inverts
- Conditions trigger when they shouldn't
- Behaviors flip unexpectedly

**How to detect:**
- Read trigger definitions for boolean changes
- Check if calls match new semantics
- Test condition evaluation

**Critical insight:** Extremely subtle. Name unchanged, syntax valid, but semantics inverted.

### Pattern 8: File Reorganization

**What happens:** Files moved to new directory structures.

**Example:**
```
schemes/ → schemes/scheme_types/
court_positions/categories/ → court_positions/
```

**Failure mode:**
- Mod files at old paths orphaned
- Vanilla at new paths takes precedence
- Or vice versa (load order dependent)

**How to detect:**
- Check for deleted files (git log --diff-filter=D)
- Look for new subdirectories
- Verify mod file paths match vanilla structure

**Critical insight:** Load order and path resolution rules matter. Same filename, different path = unexpected behavior.

### Pattern 9: Trigger Signature Changes

**What happens:** Triggers become parameterized, signatures change.

**Example:**
```diff
- petition_trigger = { scope:vassal = { ... } }
+ petition_trigger = { $VASSAL$ = { ... } }
```

**Failure mode:**
- Trigger calls don't match signature
- Runtime errors or silent failures
- Parameter mismatches

**How to detect:**
- Search for $ parameters in trigger definitions
- Check if hardcoded scopes became parameters
- Verify all calls provide required parameters

**Critical insight:** Trigger name same, but HOW you call it changed. Must update all call sites.

### Pattern 10: Modifier Decimal Precision Changes

**What happens:** Display precision adjusted, affecting value interpretation.

**Example:**
```diff
monthly_piety_gain_mult = {
- decimals = 1
+ decimals = 2
}
```

**Failure mode:**
- Mod using 0.25 might now display as 0.25 instead of 0.3
- Or values get rounded differently
- UI display mismatch

**How to detect:**
- Check modifier definitions for decimal changes
- Compare how values display in game
- Check if tooltip formatting changed

**Critical insight:** Usually cosmetic, but can affect script logic if code checks formatted strings.

### Pattern 11: Localization Key Reorganization

**What happens:** Localization keys renamed or moved to different files.

**Example:**
```diff
- MOD_MONTHLY_CONTROL_CHANGE
+ MOD_MONTHLY_CONTROL_GROWTH
```

**Failure mode:**
- Tooltips show raw keys instead of translated text
- UI looks broken to players
- Non-English languages might break

**How to detect:**
- Check if modifier suffix/prefix keys changed
- Verify localization files for new keys
- Test in different languages

**Critical insight:** Functional behavior fine, but UI/UX broken. Players see `MOD_MONTHLY_CONTROL_CHANGE` instead of "Monthly Control".

---

## How to Investigate: The Thinking Process

### Question 1: What Changed in Base Game?

**NOT:** "How many files changed?"

**YES:** "What SYSTEMS were modified?"

**Investigation approach:**

```
1. Get directory-level breakdown
   Which subsystems touched? (culture/, modifiers/, scripts/)

2. For high-change directories, read actual diffs
   What patterns appear? (additions? removals? reworks?)

3. Classify changes by type
   - Additive (safe)
   - Renames (update references)
   - Removals (investigate why)
   - Reworks (semantic analysis needed)
```

**Critical thinking:**
- Many additions + few deletions = expansion (low risk)
- Balanced adds/deletes = refactor (medium risk)
- Pattern shifts (power→duration) = rework (high risk)
- New subdirectories = reorganization (check paths)

### Question 2: Does Mod Touch Changed Systems?

**NOT:** "Check if mod uses schemes"

**YES:** "Discover what mod currently uses, then compare against what changed"

**Investigation approach:**

```
1. Read mod files to understand structure
   What systems does it interact with?

2. Extract current dependencies dynamically
   What modifiers? What traditions? What script values?

3. Cross-reference with base game changes
   For each changed system, does mod use it?

4. Assess intersection
   High overlap = high risk of breakage
   No overlap = likely safe
```

**Critical thinking:**
- Don't assume you know what mod uses
- Extract fresh from current code
- Changed system + mod usage = investigate deeper
- Changed system + no mod usage = lower priority

### Question 3: Are Dependencies Still Valid?

**NOT:** "Do modifiers exist?"

**YES:** "Do dependencies exist AND mean what the mod expects?"

**Investigation approach:**

```
1. For each dependency, verify existence
   Still defined in vanilla?

2. If missing, investigate
   Renamed? Removed? Merged? Replaced?

3. If found, verify semantics
   Read definition - same meaning as before?
   Check properties (color, suffix, sign)
   Understand calculation changes

4. Cross-check with mod usage
   Does mod use it correctly for new semantics?
   Does perk intent match modifier meaning?
```

**Critical thinking:**
- Existence ≠ correctness
- Same name ≠ same semantics
- Must understand what things DO, not just that they exist
- Context matters (how mod uses it vs what it does)

### Question 4: Do Features Actually Work?

**NOT:** "Does mod load?"

**YES:** "Do perks do what they claim?"

**Investigation approach:**

```
1. Prioritize by risk
   What changed in base? What does mod use from that?

2. Design functional tests
   For each high-risk feature, how to verify?

3. Execute in-game
   Actually test, don't just theorize

4. Verify results match expectations
   Tooltips correct? Effects apply? Magnitude right?
```

**Critical thinking:**
- Loading proves nothing (broken modifiers load fine)
- Must test actual functionality
- Edge cases matter (test scaling, conditions, interactions)
- One perk working doesn't mean all work

---

## Using Agents for Comprehensive Investigation

### When to Use Agents

Agents help achieve thoroughness when **scale or complexity exceeds manual capacity.**

**Use agents for:**

1. **Systematic exploration across large codebases**
   - "Analyze all 5 major version transitions for breaking patterns"
   - "Catalog every dependency in mod and verify against base"
   - "Compare modifier semantics across 3 versions"

2. **Multi-step investigation workflows**
   - "If tradition_X missing, search git history, find replacement, compare semantics"
   - "For each changed file, determine if mod affected, assess risk"
   - "Extract dependencies, cross-reference with changes, prioritize issues"

3. **Parallel analysis of multiple systems**
   - Simultaneously investigate modifiers AND traditions AND script values
   - Check multiple version transitions concurrently
   - Analyze diverse breakage patterns at once

### How to Task Agents Effectively

**Provide context and thinking goals:**

```
DON'T: "Check if modifiers exist"
  → Agent runs grep, reports "yes", provides no insight

DO: "Verify mod modifiers against base, investigating ANY missing/changed.
     For each issue found, search git history to understand what happened.
     For each change, assess if semantic or syntactic.
     Report with evidence and analysis."
  → Agent investigates deeply, provides reasoning
```

**Specify evidence requirements:**

```
DON'T: "Verify compatibility"
  → Vague, agent might do superficial check

DO: "Verify compatibility by:
     1. Extracting all N dependencies (show commands used)
     2. Checking each against base (show results)
     3. For missing items: investigate git history (show commits)
     4. For changed items: compare old vs new (show diffs)
     Report: counts, evidence, file:line references"
  → Agent must show work, can't just claim "checked"
```

**Enable multi-dimensional analysis:**

```
AGENT 1: What changed in base game?
  - Identify modified systems
  - Classify change types
  - Assess risk levels

AGENT 2: What does mod currently use?
  - Extract all dependencies
  - Map to base game locations
  - Categorize by type

AGENT 3: Where do they intersect?
  - Cross-reference Agent 1 & 2
  - Identify affected features
  - Prioritize investigation

AGENT 4: How to verify?
  - Design test strategy
  - Create verification approach
  - Estimate effort required
```

### Agent Limitations

**Agents cannot:**

1. **Make design decisions**
   - "Should we use tradition_piracy or find different tradition?"
   - "Is this balance change acceptable?"
   - "Which redesign approach fits mod philosophy?"

2. **Exercise judgment on acceptable risk**
   - "Is this cosmetic change worth delaying release?"
   - "Do we need to test this edge case?"
   - "Is this good enough or should we investigate deeper?"

3. **Understand player impact**
   - "Will players notice this change?"
   - "Does this break existing saves?"
   - "Should we announce this?"

**Agents are tools for thoroughness. Humans make decisions.**

### Agent Quality Control

**VERIFY agent findings:**

```
Agent reports: "All 106 modifiers verified"

Your checks:
1. Spot-check 10 random modifiers yourself
2. Verify agent's extraction count matches your manual count
3. Ask agent to show evidence for specific claims
4. If numbers don't match: agent made mistakes
5. If agent can't show work: didn't actually investigate
```

**Red flags:**
- Round numbers without evidence
- Vague language ("mostly compatible")
- Missing file:line references
- Can't reproduce when asked
- Contradicts your spot-checks

**If agent is wrong:**
- Don't trust conclusions
- Investigate yourself
- Note what agent failed at
- Adjust future agent tasking

---

## Critical Thinking Framework

### Before Any Claim

**Ask yourself:**

**1. How do I know this?**
- What file did I read?
- What command did I run?
- What was the output?
- Can someone else reproduce this?

**2. Am I discovering or assuming?**
- Did I extract this from current code?
- Or am I using cached knowledge?
- When did I last verify?

**3. What would prove me wrong?**
- What evidence would contradict this?
- Did I check for that?
- What am I not seeing?

**4. Would this catch real breakage?**
- Would this detect the scheme rework?
- What if 10 traditions renamed at once?
- What if semantics inverted?

**5. Am I checking syntax or semantics?**
- Does it exist? (syntax)
- Does it work? (semantics)
- Does it do what we think? (intent)

### Evidence Standards

**Unacceptable:**
```
"Verified all modifiers"
"Performed comprehensive analysis"
"Thorough investigation"
```
→ Claims without evidence

**Minimum:**
```
"Verified 106 modifiers exist (extraction: grep pattern, results: /tmp/modifiers.txt)"
"Found 2 missing traditions: tradition_X, tradition_Y"
```
→ Specific, verifiable, evidence provided

**Excellent:**
```
"Verified 106 modifiers by:
 - Extraction: [command shown]
 - Results: 106 unique modifiers found
 - Verification: checked each against base modifier_definition_formats/
 - Missing: 0
 - Changed: 3 (details below)
 - Spot-check: manually verified 10 random selections
 - Evidence: /tmp/modifiers.txt (full list)"
```
→ Shows method, results, verification, evidence locations

### Report Quality

**Bad report:**
```
Checked compatibility. Looks good. No issues found.
```
→ Useless. No evidence. No detail. Can't verify.

**Good report:**
```
Compatibility Check for CK3 1.19.0:

Investigation:
- Base game changes: 847 files (modifier_definition_formats: 12 files, culture: 8 files)
- Read diffs: No major system reworks detected
- Extracted dependencies: 106 modifiers, 35 traditions, 5 ethos, 2 script values
- Verified against base/1.19.0: All found, no missing dependencies
- Semantic check: Reviewed changed modifiers, no semantic shifts detected
- Build: Success
- Functional test: Spot-checked 5 perks, all working

Issues: None

Conclusion: Compatible with CK3 1.19.0

Evidence: /tmp/verification_1.19.0/
```
→ Specific. Verifiable. Shows work done.

---

## Investigation Workflows

### Workflow 1: Minor Update (Fast Path)

**Scenario:** Small patch, few hundred files changed.

**Thinking process:**

1. **Quick recon**
   - Check which directories changed
   - If modifier_definition_formats/ and culture/ untouched → low risk
   - If script_values/ unchanged → low risk

2. **Spot verification**
   - Extract mod's 2 script values, check they still exist
   - Sample 10-20 modifiers, verify found
   - Sample 5 traditions, verify found

3. **Build test**
   - Run build, check for errors

4. **Quick functional test**
   - Load game, check one perk per legacy
   - Verify visibility conditions work

**If all clear → likely safe. Update version metadata.**

**Time: 30-60 minutes**

### Workflow 2: Major Update (Deep Investigation)

**Scenario:** Major CK3 update, thousands of files changed.

**Thinking process:**

1. **Comprehensive recon**
   - Analyze which systems changed heavily
   - Read diffs for high-change directories
   - Look for patterns (reworks, renames, removals)

2. **Complete dependency extraction**
   - Extract ALL mod dependencies fresh
   - No cached lists, no assumptions

3. **Systematic verification**
   - Check every dependency
   - For missing: investigate git history
   - For changed: compare semantics

4. **Semantic analysis**
   - For any changed systems mod uses
   - Read old vs new definitions
   - Understand if meaning changed

5. **Comprehensive testing**
   - Test all perks using changed systems
   - Verify visibility for all legacy tracks
   - Check edge cases and scaling

6. **Documentation**
   - Document what changed and why
   - Record investigation process
   - Note any issues and fixes

**Time: 4-8 hours (or use agents to parallelize)**

### Workflow 3: Agent-Assisted Deep Dive

**Scenario:** Complex update, want comprehensive coverage.

**Thinking process:**

1. **Design investigation**
   - What needs checking?
   - What questions need answering?
   - What evidence required?

2. **Task multiple agents in parallel**
   - Agent 1: Analyze base game changes
   - Agent 2: Catalog mod dependencies
   - Agent 3: Verify each dependency
   - Agent 4: Compare semantics
   - Agent 5: Design test strategy

3. **Synthesize findings**
   - Review all agent reports
   - Identify contradictions
   - Spot-check key claims

4. **Deep dive on issues**
   - Manually investigate anything agents flagged
   - Read actual code for critical changes
   - Make judgment calls on fixes

5. **Verify agent work**
   - Spot-check their claims
   - Run key commands yourself
   - Don't trust without verifying

**Time: 2-3 hours active (agents work in parallel)**

---

## The Meta-Skill: Learning to Investigate

### Pattern Recognition

**As you investigate multiple updates, you build intuition:**

**Dangerous signals:**
- File X changes >100 lines → likely rework
- Pattern Y appears: mass deletions + pattern shift → semantic change
- Same property renamed across N files → systematic refactor
- New suffix variants (X_child, X_adult) → differentiation
- New DLC flags everywhere → content gating

**Safe signals:**
- Only additions, no deletions → expansion
- New files in separate directories → new systems
- Decimal precision tweaks → cosmetic
- Comment changes only → documentation

**Build your own pattern library from experience.**

### Developing Investigative Instinct

**After several updates, you should be able to:**

1. **Quickly assess risk from diff stats**
   - Glance at changed directories → know priority
   - See file change counts → gauge investigation effort
   - Recognize dangerous files (scheme_definitions.txt = high risk)

2. **Identify change types from diff patterns**
   - Mass delete/add → rework
   - Systematic renames → refactor
   - Value changes → rebalance
   - Property additions → differentiation

3. **Know where to dig deeper**
   - If X changed, check Y too (related systems)
   - If modifier renamed, check related modifiers
   - If one tradition changed, check others in same file

4. **Design efficient verification**
   - High-risk systems get comprehensive checks
   - Low-risk systems get spot checks
   - Unchanged systems get build-only verification

**This instinct comes from repetition and reflection.**

### Questions That Promote Thinking

**When investigating, constantly ask:**

**"What am I not seeing?"**
- What systems did I not check?
- What types of changes might I miss?
- What assumptions am I making?

**"What would break this?"**
- If tradition X renamed, what breaks?
- If modifier Y removed, what breaks?
- If semantic Z inverted, what breaks?

**"How would I know if broken?"**
- What symptoms would appear?
- What tests would catch it?
- What wouldn't catch it?

**"Is this good enough?"**
- Did I verify thoroughly?
- What's my confidence level?
- What corners did I cut?
- Should I investigate deeper?

**"What's the worst case?"**
- If I'm wrong, what's the impact?
- How many players affected?
- How hard to fix later?
- Should I be more thorough now?

---

## Verification Depth: Choosing Your Level

### Level 1: Minimal (High Risk)

**What you do:**
- Check mod builds
- Load game, verify no errors
- Spot-check 1-2 perks

**What this catches:**
- Syntax errors
- Catastrophic breakage

**What this misses:**
- Silent failures (broken modifiers)
- Semantic changes
- Partial breakage
- Edge cases

**Appropriate when:**
- Tiny patch, only localization changes
- Personal use only
- Can fix later if issues found

**NOT appropriate for:**
- Public releases
- Major updates
- After system reworks

### Level 2: Standard (Medium Risk)

**What you do:**
- Read diffs for changed directories
- Extract and verify all dependencies
- Investigate missing items
- Test perks in changed systems
- Build and load test

**What this catches:**
- Missing dependencies
- Most renames
- Obvious semantic changes
- Major functional breakage

**What this misses:**
- Subtle semantic shifts
- Balance changes
- Edge case breakage
- Complex interactions

**Appropriate when:**
- Regular updates
- Moderate changes in base
- Public releases (with caveats)

**Time: 2-4 hours**

### Level 3: Comprehensive (Low Risk)

**What you do:**
- Deep analysis of all changed systems
- Semantic verification of dependencies
- Comprehensive functional testing
- Edge case validation
- Agent-assisted systematic checks
- Multi-version comparison

**What this catches:**
- Everything Level 2 catches
- Semantic shifts
- Balance implications
- Subtle edge cases
- System interactions

**What this might still miss:**
- Extremely rare edge cases
- Multi-mod interactions
- Obscure DLC combinations

**Appropriate when:**
- Major CK3 updates
- After known system reworks
- Critical releases
- Long time since last update

**Time: 4-8 hours (or 2-3 hours with agents)**

### Choosing Depth

**Ask:**

1. **How much changed?**
   - Huge update → go deep
   - Small patch → can be lighter

2. **What systems changed?**
   - Mod-critical systems → comprehensive
   - Unrelated systems → lighter

3. **What's at stake?**
   - Public release → thorough
   - Personal testing → can be lighter

4. **How long since last check?**
   - Months of updates → comprehensive
   - Recent verification → lighter

**No one-size-fits-all. Use judgment.**

---

## Example: Thinking Through an Update

**Scenario:** CK3 1.19.0 just released. Need to verify compatibility.

### NOT This (Script-Following):

```
✗ Run recon.sh
✗ Run verify_dependencies.sh
✗ Check if returns 0
✗ If yes: done
```
→ Mindless execution, no understanding

### YES This (Critical Thinking):

**Phase 1: Assess**
```
Questions:
- How big is this update?
  → Run: git diff --stat base/1.18.X..base/1.19.0 | tail -1
  → Interpret: 800 files = moderate, 8000 = major

- What systems changed?
  → Run: git diff --name-only ... base/game/common/ | cut -d/ -f4 | sort | uniq -c
  → Interpret: modifier_definition_formats 237 files = HIGH PRIORITY

- Is this a DLC or patch?
  → Check: base/.ck3-version.json for release info
  → Interpret: DLC = new systems, patch = fixes/balance
```

**Phase 2: Investigate**
```
Since modifier_definition_formats changed heavily:

Questions:
- What specifically changed?
  → Read: git diff ... modifier_definition_formats/00_definitions.txt
  → Look for: Pattern shifts, mass removals, property changes

- Do I see patterns like scheme rework?
  → Check: Are there systematic deletions of one pattern?
  → Check: Are there additions of different pattern?
  → Assess: Rework or just additions?

- Does mod use this system?
  → Extract: What modifiers does mod currently use?
  → Compare: Against what changed
  → Assess: Overlap = risk
```

**Phase 3: Decide**
```
Decision tree:

If massive rework detected:
  → Deep investigation required
  → Task agents for comprehensive analysis
  → Manual semantic verification
  → Extensive testing

If moderate changes:
  → Verify affected dependencies
  → Check semantic changes
  → Test changed features

If minor additions:
  → Quick dependency check
  → Build test
  → Light functional testing
```

**Phase 4: Verify**
```
Based on findings:

If high risk:
  → Test all perks in affected system
  → Verify all dependencies thoroughly
  → Check edge cases

If medium risk:
  → Test sample perks
  → Verify critical dependencies
  → Spot-check functionality

If low risk:
  → Build test
  → Quick functional check
  → Proceed
```

**Throughout: THINK, don't execute blindly.**

---

## Using History to Build Understanding

### Learn from Past Updates

**Review git history:**

```
What to look for:
- Which updates had breaking changes?
- What types of changes broke things?
- How were they detected?
- How were they fixed?
- What patterns repeat?
```

**Build mental model:**
- Scheme rework (v1.13) = semantic overhaul pattern
- Control rename (v1.12) = systematic refactor pattern
- Health aging (v1.12) = differentiation pattern
- DLC expansions = gating pattern

**Use this to prioritize:**
- If scheme files changed → HIGH PRIORITY (history of rework)
- If cultural files changed → MEDIUM (rarely break, but check)
- If trait files changed → LOW (mod uses 1 trait, stable)

### Recognize Cascading Changes

**Some changes imply other changes:**

**Example:**
```
If you see:
  modifier_definition_formats/00_scheme_definitions.txt changed heavily

Then also check:
  script_values/00_scheme_values.txt (defines scheme magnitudes)
  scripted_triggers/*scheme*.txt (scheme-related logic)
  schemes/scheme_types/ (scheme implementations)

Because scheme systems are interconnected.
```

**Build system knowledge:**
- What depends on what?
- What's interconnected?
- What changes together?

**Use this to expand investigation beyond obvious.**

---

## The Standard: Thoroughness Through Thinking

**Not about following steps.**

**Not about running scripts.**

**Not about checking lists.**

**About UNDERSTANDING:**
- What changed in base game (and why)
- What mod currently depends on (discovered, not assumed)
- How changes affect mod (semantically, not just syntactically)
- Whether features still work (tested, not theorized)

**About INVESTIGATING:**
- Reading diffs to understand patterns
- Researching missing items to find replacements
- Analyzing semantics to verify correctness
- Testing functionally to confirm behavior

**About EXERCISING JUDGMENT:**
- Assessing risk vs effort
- Making design decisions
- Choosing verification depth
- Deciding what's good enough

**This requires THINKING, not executing.**

**Agents help with scale. Reading helps with understanding. Testing helps with verification.**

**But only YOU can think critically, make decisions, and take responsibility for the results.**

**That's the standard. Meet it through thought, not scripts.**

---

## Quick Reference

### Verified Mod Stats (Run commands to get current values)

```bash
# Don't trust these numbers - verify yourself:

# Legacy tracks
grep -c "^bld_.*_legacy_track = {" mod/common/dynasty_legacies/more_legacies.txt

# Perks
grep -c "^bld_.*_legacy_[1-5] = {" mod/common/dynasty_perks/more_dynasty_perks.txt

# Modifiers (extraction pattern discovered from reading file structure)
grep -oP '^\t\t[a-z_]+ = ' mod/common/dynasty_perks/more_dynasty_perks.txt | \
  sed 's/^\t\t//; s/ = //' | sort -u | wc -l

# Traditions
grep -oP 'tradition_\w+' mod/common/dynasty_legacies/more_legacies.txt | sort -u | wc -l

# Ethos
grep -oP 'ethos_\w+' mod/common/dynasty_legacies/more_legacies.txt | sort -u | wc -l

# Script values
grep -oP '\w+_value' mod/common/dynasty_perks/more_dynasty_perks.txt | sort -u | wc -l
```

### Essential Investigation Commands

```bash
# Understand scale of changes
git diff --stat BASE_OLD..BASE_NEW | tail -1

# See which directories changed
git diff --name-only BASE_OLD..BASE_NEW base/game/common/ | cut -d/ -f4 | sort | uniq -c | sort -rn

# Read actual changes in critical file
git diff BASE_OLD..BASE_NEW -- base/game/common/path/to/file.txt | less

# Find when something removed/changed
git log BASE_OLD..BASE_NEW --all -S"search_term" --oneline

# See what changed in a directory
git diff --stat BASE_OLD..BASE_NEW -- base/game/common/directory/
```

### Critical Files for This Mod

```
HIGH PRIORITY (mod heavily depends on):
- modifier_definition_formats/00_definitions.txt
- modifier_definition_formats/00_scheme_definitions.txt (history of rework)
- culture/traditions/*.txt (35 traditions referenced)
- culture/pillars/00_ethos.txt (5 ethos referenced)

MEDIUM PRIORITY (mod uses, but stable):
- script_values/00_scheme_values.txt (2 values used)
- scripted_triggers/00_dynasty_triggers.txt (1 trigger used)

LOW PRIORITY (mod lightly depends on):
- traits/00_traits.txt (1 trait: witch)
- secret_types/00_secret_types.txt (1 secret: secret_witch)
```

### Breakage Pattern Quick Guide

| Pattern | Risk | Detection Method |
|---------|------|------------------|
| Semantic rework | CRITICAL | Read diffs for pattern shifts |
| Systematic renames | HIGH | Search for old property names |
| Value scale changes | MEDIUM | Compare value distributions |
| Age differentiation | MEDIUM | Check for context suffixes |
| Complete removal | HIGH | Verify existence |
| DLC gating | MEDIUM | Search for new dlc_feature |
| Logic inversions | HIGH | Read trigger semantics |
| File reorganization | MEDIUM | Check deleted files |
| Signature changes | HIGH | Verify parameter lists |
| Decimal precision | LOW | Check definition properties |
| Localization shifts | LOW | Verify key references |

---

## Deep Dives: Investigating Specific Patterns

### Deep Dive 1: Detecting Semantic Reworks

**Recognition signals:**
- Large file with many deletions AND many additions
- Different naming patterns (power→duration, change→growth)
- Property structure changes (new fields in definitions)

**Investigation process:**

1. **Identify scale of change**
   - How many lines added/removed?
   - If >100 lines changed: likely significant
   - Pattern: deletions ≈ additions = replacement

2. **Read the diff carefully**
   - What's being deleted? (old system)
   - What's being added? (new system)
   - Are names similar but different? (suggests related)

3. **Understand the conceptual shift**
   - OLD system: How did it work mechanically?
   - NEW system: How does it work now?
   - Can you auto-convert? Or need redesign?

4. **Assess mod impact**
   - Does mod use old system?
   - How many features affected?
   - Can they be salvaged or must be redesigned?

**Example thinking process:**

```
See in diff: 48 deletions of "*_power" and "*_resistance"
             24 additions of "*_phase_duration"

Think: This isn't adding features, it's replacing a system.
       Power/resistance suggests competitive calculation.
       Phase_duration suggests time-based progression.

       These are fundamentally different mechanics.

Check: Does mod use any power/resistance modifiers?
       If yes: CRITICAL - must understand new system and convert
       If no: LOW - mod unaffected

Decide: If affected, need to:
        1. Understand how new system works
        2. Find semantic equivalent to old effects
        3. Redesign perks for new mechanic
        4. Test thoroughly to verify equivalence
```

### Deep Dive 2: Systematic Rename Detection

**Recognition signals:**
- Same base name, different suffix/prefix
- Multiple files showing same pattern
- Usually documented in patch notes

**Investigation process:**

1. **Find one renamed item**
   - Mod references X, not found in base
   - Search for similar names
   - Find base now has X' (similar but different)

2. **Search for pattern**
   - If X renamed to X', are there related renames?
   - Search for similar naming patterns
   - Check if systematic across file

3. **Verify all instances**
   - List all mod references to old pattern
   - Update each to new pattern
   - Verify completeness

**Example thinking process:**

```
Mod uses: monthly_county_control_change_add
Not found in base.

Search: "county_control" in base files
Find: monthly_county_control_growth_add

Think: "change" became "growth"
       Is this just this one modifier? Or pattern?

Search: All "*_change_*" modifiers in base
Find: NONE remain - all became "*_growth_*"

Conclude: Systematic rename across entire control system

Action: Search mod for "*_change_*" pattern
        Update all to "*_growth_*" pattern
        Verify: grep to confirm none remain
```

### Deep Dive 3: Value Scale Investigation

**Recognition signals:**
- Same property names
- Significantly different value magnitudes
- Consistent ratio across multiple modifiers

**Investigation process:**

1. **Sample value comparisons**
   - Pick 10-20 modifiers using same property
   - Compare old vs new values
   - Calculate ratios

2. **Look for consistency**
   - Are ratios consistent? (all ~4:1, or varied?)
   - Does pattern hold across modifiers?
   - Any outliers that don't fit pattern?

3. **Find system change**
   - Check defines for scale changes
   - Look for max/min value changes
   - Read patch notes for rebalancing

4. **Assess mod impact**
   - What values does mod use?
   - Are they now overpowered/underpowered?
   - Do they need rescaling?

**Example thinking process:**

```
Sample health modifiers:
  OLD: health = 1.0, 2.0, 0.5
  NEW: health = 0.25, 0.5, 0.125
  Ratio: ~4:1 reduction

Check more modifiers:
  OLD: health = -3.0, -1.5
  NEW: health = -0.75, -0.375
  Ratio: Same ~4:1

Conclude: Health scale changed globally

Check defines:
  MAX_HEALTH: Was 10, now 2.5
  Confirms: 4:1 scale change

Mod uses: health = 0.5
Think: On old scale (max 10), this is 5% of max
       On new scale (max 2.5), this is 20% of max!

Action: Reduce mod values by 4:1 to maintain balance
        health = 0.5 → health = 0.125
```

### Deep Dive 4: DLC Gating Impact

**Recognition signals:**
- `has_dlc_feature` additions
- `dlc_feature = X` flags in definitions
- Content moving behind DLC checks

**Investigation process:**

1. **Identify what got gated**
   - Search diff for new `dlc_feature` flags
   - List affected buildings/governments/modifiers
   - Understand which DLC

2. **Check if mod uses gated content**
   - Does mod reference affected features?
   - Do mod features depend on DLC content?
   - Are there conditional paths?

3. **Assess player impact**
   - What % of players have this DLC?
   - Is feature core or optional?
   - Can mod work without it?

4. **Decide on approach**
   - Add has_dlc_feature checks?
   - Make feature optional?
   - Redesign without DLC dependency?
   - Document DLC requirement?

**Example thinking process:**

```
See in diff: court_grandeur_baseline_add = { dlc_feature = royal_court }

Mod uses: court_grandeur_baseline_add (in 3 perks)

Think: This modifier now requires Royal Court DLC
       Without DLC, these perks won't apply this bonus

Question: Do players need Royal Court for mod to work?
Answer: NO - other perks still work
        YES - these 3 perks partially broken without DLC

Options:
1. Leave as-is (mod description: "Enhanced with Royal Court")
2. Add has_dlc_feature checks (graceful degradation)
3. Replace with non-DLC bonuses (remove dependency)

Decide: Option 1 - document DLC enhancement
        Rationale: Court grandeur is nice-to-have, not essential
                   Many players have Royal Court
                   Simpler than conditional logic
```

### Deep Dive 5: File Reorganization Handling

**Recognition signals:**
- Git log shows file deletions
- New subdirectories appear
- Similar filenames at different paths

**Investigation process:**

1. **Find reorganization**
   - Check for deleted files: `git log --diff-filter=D`
   - Check for new directories: `git diff --name-status`
   - Compare directory structures

2. **Understand load order implications**
   - Does mod override files at old paths?
   - Will vanilla at new paths take precedence?
   - Are there conflicts?

3. **Test file resolution**
   - Which file loads in game?
   - Does mod's version or vanilla's load?
   - Is behavior as expected?

4. **Update mod structure if needed**
   - Move mod files to match vanilla structure
   - Or remove mod overrides if vanilla path changed
   - Verify after change

**Example thinking process:**

```
See: schemes/*.txt deleted
     schemes/scheme_types/*.txt added

Mod has: No scheme overrides (doesn't touch scheme files)

Conclude: No impact - mod doesn't override those files

But think: Does mod REFERENCE scheme files?
          Check: Any scripted_effect calls to schemes?
          Check: Any on_action hooks for scheme events?

If yes: Verify references still work with new paths
If no: No action needed
```

---

## Agent Tasking for Diverse Scenarios

### Scenario 1: Cultural System Investigation

**When:** culture/traditions/ or culture/pillars/ changed significantly.

**Agent task:**

```
Investigate cultural system changes in CK3 update BASE_OLD→BASE_NEW

Focus:
1. Tradition analysis:
   - Which traditions were removed/renamed/changed?
   - Read actual diffs for each changed tradition file
   - For each change, determine type (rename vs redesign vs removal)

2. Ethos analysis:
   - Any ethos changed?
   - Any new ethos parameters added?
   - Any DLC gating on cultural features?

3. Mod cross-reference:
   - Mod uses 35 traditions (extract fresh from mod files)
   - Mod uses 5 ethos (extract fresh)
   - Which of mod's dependencies were affected?

4. Replacement search:
   - For each missing tradition, search for semantic equivalent
   - Compare old vs new definitions
   - Assess if replacement suitable

Deliver:
- List of ALL cultural changes (with git commits)
- Cross-reference of mod usage vs changes
- Recommended updates for mod (specific file:line changes)
- Confidence assessment (how similar are replacements?)

Evidence: Show diffs, show git log, show definitions compared.
```

**Why this task design:**
- Focused on cultural system (not trying to check everything)
- Requires reading actual changes (not just existence checking)
- Demands cross-referencing and judgment
- Must provide evidence

### Scenario 2: Modifier Semantic Analysis

**When:** modifier_definition_formats/ changed, need to verify semantics.

**Agent task:**

```
Analyze modifier semantic changes in BASE_OLD→BASE_NEW

Investigation:
1. Changed modifier detection:
   - Read full diff of all modifier_definition_formats/ files
   - List modifiers that changed properties (color, suffix, decimals, etc.)
   - List modifiers that changed definitions substantively

2. Mod usage analysis:
   - Extract all 106 modifiers mod uses
   - For each: determine usage context (which perk, what intent)
   - Example: "Romantic" perk uses courting_scheme_* → intent is to help romance

3. Semantic verification:
   - For each modifier mod uses that CHANGED:
     a. Read OLD definition (what it meant before)
     b. Read NEW definition (what it means now)
     c. Compare: Same semantics? Or different?
     d. Check: Does mod usage align with new semantics?

4. Intent matching:
   - For each perk, does modifier still match perk intent?
   - Example: "Faster" perk using duration modifier → check sign is negative
   - Flag mismatches for review

Deliver:
- List of modifiers with semantic changes
- For each: old meaning vs new meaning
- Perk-by-perk analysis: does usage match intent?
- Flagged issues requiring human judgment

Show work: Include modifier definitions, explain reasoning.
```

**Why this task design:**
- Focused on SEMANTICS, not just existence
- Requires understanding perk intent
- Demands reading and comparing definitions
- Flags issues for human review (doesn't auto-fix)

### Scenario 3: Multi-Version Pattern Analysis

**When:** Want to understand what TYPES of changes break mods (learning for future).

**Agent task:**

```
Analyze 5 version transitions to identify breakage patterns

Versions to analyze:
- 1.11.5 → 1.12.5 (Peacock → Scythe)
- 1.12.5 → 1.13.0 (Scythe → Basileus)
- 1.13.0 → 1.14.3 (Basileus → Traverse)
- 1.14.3 → 1.15.0 (Traverse → Crown)
- 1.15.0 → 1.16.0 (Crown → Chamfron)

For EACH transition:
1. Quantify changes (files, directories, areas)
2. Identify change TYPES (additions, renames, removals, reworks)
3. Find examples of EACH pattern type
4. Assess what would have broken mods using old system

Focus on DIVERSITY:
- Don't just analyze schemes 5 times
- Find different patterns across different systems
- Look at modifiers, cultures, scripts, traits, etc.

Deliver:
- Pattern taxonomy (types of changes observed)
- Real examples from each transition
- How each pattern manifests in diffs
- What detection method works for each
- Which transitions were risky vs safe

Goal: Build comprehensive understanding of CK3 update patterns.
```

**Why this task design:**
- Covers multiple versions (not just one)
- Demands finding diverse patterns
- Builds general knowledge (not specific to one system)
- Helps develop investigation instinct

### Scenario 4: Dependency Impact Tree

**When:** Want to understand cascading effects of changes.

**Agent task:**

```
Map dependency relationships and cascading impacts

Investigation:
1. Mod dependency extraction:
   - All modifiers, traditions, script values, triggers, etc.
   - For EACH, determine what IT depends on
   - Example: modifier uses script_value, script_value uses define

2. Base game system mapping:
   - What base systems does each dependency touch?
   - What's interconnected?
   - Example: scheme modifiers → scheme values → scheme mechanics

3. Cascading change analysis:
   - If X changes, what else might change?
   - If script_value Y changed, what modifiers use it?
   - If trigger Z signature changed, what calls it?

4. Impact tree construction:
   - For each mod feature, map dependency chain
   - Identify single points of failure
   - Assess cascade risk

Deliver:
- Dependency graph (what depends on what)
- For each mod feature: dependency chain depth
- High-risk dependencies (many things depend on them)
- If X breaks, what breaks in cascade?

Use this to: Prioritize verification (high-impact dependencies first).
```

**Why this task design:**
- Builds system understanding
- Identifies critical dependencies
- Shows cascading risks
- Helps prioritize investigation effort

---

## Investigation Case Studies

### Case Study 1: Missing Tradition Investigation

**Situation:** Mod references `tradition_practiced_pirates`, not found in new base.

**Thinking process:**

**Step 1: Confirm missing**
```
grep -r "tradition_practiced_pirates" base/game/common/culture/traditions/
→ No results

Conclusion: Definitely not in current base
```

**Step 2: Search git history**
```
git log BASE_OLD..BASE_NEW --all -S"tradition_practiced_pirates" --oneline
→ Shows commit where it was removed/changed

Read commit message and diff
→ Understand what Paradox did
```

**Step 3: Find candidates**
```
Keywords: pirate, piracy, naval, raiding, coastal

Search: grep -r "pirat\|piracy" base/game/common/culture/traditions/
Find: tradition_piracy, tradition_fp1_coastal_warriors

Think: Which is semantic equivalent?
```

**Step 4: Compare definitions**
```
OLD (from git history):
  tradition_practiced_pirates = {
    character_modifier = { coastal_advantage = 5 }
    county_modifier = { naval_movement_speed = 0.1 }
  }

NEW option 1:
  tradition_piracy = {
    character_modifier = { coastal_advantage = 5 }
    county_modifier = { naval_movement_speed = 0.1, monthly_loot_mult = 0.15 }
  }

NEW option 2:
  tradition_fp1_coastal_warriors = {
    character_modifier = { coastal_advantage = 10 }
    # Different bonuses, more combat-focused
  }

Analysis:
- tradition_piracy: 95% equivalent, adds loot bonus
- tradition_fp1_coastal_warriors: Different focus (combat vs piracy)
```

**Step 5: Decision**
```
Question: Which should mod use?

Option A (tradition_piracy):
  + Closest semantic match
  + Same bonuses mod expected
  + Not DLC-gated
  - Adds unexpected loot bonus

Option B (tradition_fp1_coastal_warriors):
  + Still maritime/naval theme
  - Different bonuses (combat focus)
  - Requires FP1 DLC
  - Less direct replacement

Choose: tradition_piracy
Reason: Closest match, no DLC requirement, bonus alignment
```

**Step 6: Implementation**
```
Update: mod/common/dynasty_legacies/more_legacies.txt:95
Change: tradition_practiced_pirates → tradition_piracy

Test: Load game with Norse culture
      Check: Seafaring legacy shows ✓

Document: CHANGELOG entry explaining change
```

**Key insight:** This is JUDGMENT, not automation. Had to understand semantics, compare options, make design decision.

### Case Study 2: Decimal Precision Investigation

**Situation:** Modifier definitions show changed decimal values.

**Thinking process:**

**Step 1: Notice the change**
```
Diff shows:
  monthly_piety_gain_mult = {
-   decimals = 1
+   decimals = 2
  }

Think: What does this affect?
```

**Step 2: Understand implications**
```
decimals = 1 means: 0.1, 0.2, 0.3 precision
decimals = 2 means: 0.01, 0.02, 0.03 precision

Higher precision = more granular values allowed

Question: Does mod use this modifier?
```

**Step 3: Check mod usage**
```
grep "monthly_piety_gain_mult" mod/

Find: bld_tradition_legacy_3 uses monthly_piety_gain_mult = 0.15

Old display: 0.2 (rounded)
New display: 0.15 (exact)

Analysis: Better precision, now shows actual value
Impact: COSMETIC - tooltip display improved, no functional change
```

**Step 4: Assess criticality**
```
Question: Does this break anything?

Check: Is 0.15 still valid? Yes (within precision range)
Check: Does calculation change? No (just display)
Check: Could this affect script logic?
       Only if code checks formatted strings (unlikely)

Conclude: LOW RISK - cosmetic improvement
Action: No changes needed
```

**Key insight:** Not all changes matter. Some are improvements. Must assess critically.

### Case Study 3: Multi-System Investigation

**Situation:** Major update, many systems changed, need efficient verification.

**Thinking process:**

**Step 1: Identify high-risk areas**
```
Run: git diff --name-only BASE_OLD..BASE_NEW base/game/common/ | cut -d/ -f4 | sort | uniq -c

See:
  237 modifier_definition_formats  ← HIGH
  143 culture                       ← HIGH
   54 script_values                 ← MEDIUM
   12 traits                         ← LOW

Think: Can't thoroughly check everything
       Must prioritize based on mod usage and risk
```

**Step 2: Parallel agent investigation**
```
Launch 3 agents:

Agent A: Deep dive on modifier_definition_formats/
  - Read all diffs
  - Categorize changes
  - Cross-ref with mod's 106 modifiers
  - Flag issues

Agent B: Cultural system analysis
  - Check all 35 traditions
  - Verify all 5 ethos
  - Investigate any changes
  - Find replacements if needed

Agent C: Script values and supporting systems
  - Verify 2 script values
  - Check triggers, traits, secrets
  - Low priority but systematic

Run in parallel → save time
```

**Step 3: Synthesize results**
```
Agent A reports: 3 modifiers changed decimals (cosmetic), 0 missing
Agent B reports: 2 traditions renamed, found replacements
Agent C reports: All script values unchanged, triggers stable

Spot-check: Verify 5 claims from each agent
Result: Agents correct

Conclude: 2 tradition renames need fixing, rest OK
```

**Step 4: Manual investigation of issues**
```
For 2 renamed traditions:
- Read the actual diffs myself (don't just trust agent)
- Compare old vs new definitions
- Make judgment on replacements
- Update mod
- Test visibility
```

**Step 5: Targeted testing**
```
Test priority:
1. Seafaring legacy (affected by tradition renames) - HIGH
2. One scheme perk (Agent A said modifiers changed) - MEDIUM
3. Spot-check other legacies - LOW

Result: All working
```

**Key insight:** Agents handle scale, humans handle judgment. Combination is powerful.

---

## Developing Investigation Expertise

### Building Mental Models

**As you investigate updates, internalize:**

**System relationships:**
```
Schemes system connects:
- scheme_definitions.txt (modifier definitions)
- 00_scheme_values.txt (magnitude script values)
- schemes/scheme_types/*.txt (scheme implementations)
- scripted_triggers/*scheme*.txt (scheme logic)

If one changes significantly, check others.
```

**Change frequency patterns:**
```
Very stable (almost never change):
- Core ethos (bellicose, courtly, etc.)
- Basic traits (brave, craven, etc.)
- Core modifiers (health, prowess, skills)

Occasionally change:
- DLC traditions (renamed for clarity)
- Script values (rebalancing)
- Modifier decimals (precision tweaks)

Volatile (change frequently):
- Event modifiers (added/removed often)
- On_action event lists (constant additions)
- Activity content (new events)
```

**Risk assessment intuition:**
```
High-risk changes:
- >100 lines in critical file
- Pattern shifts in naming
- Mass deletions + different additions
- Property semantics (color, suffix) changes

Low-risk changes:
- Pure additions
- Comment changes
- Decimal precision tweaks
- Localization updates
```

### Pattern Library Development

**Maintain mental catalog:**

**Pattern:** Systematic refactor
```
Signals: Same base name, different suffix across many instances
Example: *_change_* → *_growth_*
Detection: Search for old pattern, find systematic replacement
Fix: Global find/replace after verifying pattern complete
```

**Pattern:** Semantic rework
```
Signals: Mass deletions + different additions, naming suggests mechanic change
Example: power/resistance → phase_duration
Detection: Read diffs for conceptual shifts
Fix: Redesign features for new mechanic (can't auto-convert)
```

**Pattern:** Feature gating
```
Signals: New dlc_feature flags, has_dlc_feature checks
Example: Court grandeur requiring Royal Court
Detection: Search for new DLC flags
Fix: Add checks, document requirement, or redesign
```

**Pattern:** Value rebalancing
```
Signals: Same names, different value ranges, consistent ratios
Example: Health 0-10 → 0-2.5
Detection: Statistical comparison of value distributions
Fix: Rescale mod values to match new scale
```

**Add to this library with each update investigated.**

### Investigation Anti-Patterns

**Things that feel thorough but aren't:**

**Anti-Pattern 1: Exhaustive existence checking**
```
✗ Check all 106 modifiers exist: ✓ All found!

Problem: Doesn't catch semantic changes
         Doesn't verify they work as expected
         False sense of security
```

**Anti-Pattern 2: Running pre-made verification scripts**
```
✗ ./verify_mod.sh
  ✓ All checks passed!

Problem: Script checks what script-writer thought to check
         Misses new types of breakage
         Becomes stale as game evolves
```

**Anti-Pattern 3: Trusting "no errors" as success**
```
✗ Game loads, no errors in log
  ✓ Must be compatible!

Problem: CK3 silently ignores unknown modifiers
         Broken features don't generate errors
         Missing dependencies invisible
```

**Anti-Pattern 4: Testing only what you expect to break**
```
✗ Scheme system changed, so test only scheme perks
  ✓ Scheme perks work!

Problem: Cascading changes might affect other systems
         Unexpected interactions possible
         Tunnel vision misses surprises
```

**Instead:**
- Verify semantics, not just existence
- Think about what to check, don't follow scripts
- Test functionality, not just loading
- Stay alert for unexpected breakage

---

## When Verification Reveals Problems

### Problem Type 1: Missing Dependency

**You found:** Tradition_X not in base game.

**Investigation steps:**

1. **Understand what happened**
   - Git log: Was it removed? Renamed?
   - Patch notes: Is it mentioned?
   - Similar names: Any likely replacements?

2. **Assess impact**
   - Where does mod use it?
   - What breaks if missing?
   - How critical is the feature?

3. **Find solution**
   - Replacement found: Compare semantics
   - No replacement: Can we redesign?
   - Feature obsolete: Remove from mod?

4. **Implement fix**
   - Update references
   - Test affected features
   - Verify nothing else breaks

5. **Document**
   - CHANGELOG: What changed and why
   - Commit message: Evidence and reasoning

### Problem Type 2: Semantic Mismatch

**You found:** Modifier exists but means something different now.

**Investigation steps:**

1. **Understand the change**
   - Read old definition (git show)
   - Read new definition (current base)
   - Compare: What's different?

2. **Analyze impact**
   - How does mod use this?
   - Does new meaning contradict intent?
   - Can we adapt or must redesign?

3. **Consider options**
   - Use different modifier that matches intent?
   - Adjust values for new semantics?
   - Redesign feature entirely?

4. **Test implications**
   - If we make change X, what happens?
   - Does it maintain balance?
   - Does it preserve intent?

5. **Choose and implement**
   - Make design decision
   - Update code
   - Test thoroughly

### Problem Type 3: Scale Mismatch

**You found:** Values that worked before now seem wrong.

**Investigation steps:**

1. **Identify the scale change**
   - Compare many values across versions
   - Find consistent ratio
   - Check defines for system maximums

2. **Assess mod values**
   - What values does mod use?
   - On old scale vs new scale?
   - Overpowered? Underpowered? Appropriate?

3. **Decide on approach**
   - Rescale to match vanilla changes?
   - Keep current (if still balanced)?
   - Use opportunity to rebalance?

4. **Verify broadly**
   - Don't just fix one value
   - Check all uses of that property
   - Ensure consistency

### Problem Type 4: Cascading Breakage

**You found:** One change breaks multiple things.

**Investigation steps:**

1. **Map the cascade**
   - What directly uses broken item?
   - What uses those items?
   - How far does it propagate?

2. **Prioritize fixes**
   - Fix root cause first
   - Or fix each broken item individually?
   - What's most efficient?

3. **Verify cascade resolution**
   - After fixing root, check all dependents
   - Ensure cascade resolved
   - No lingering breakage

**Example:**
```
script_value X changed value
  → Used by modifiers A, B, C
    → Used in perks 1, 2, 3, 4
      → Affects legacies Alpha, Beta

Fix script_value X
  → Verify modifiers A, B, C still work
    → Test perks 1, 2, 3, 4
      → Confirm legacies Alpha, Beta unaffected
```

---

## Advanced Topics

### Semantic Analysis Techniques

**Beyond "does it exist?"**

**Technique 1: Property semantics**

```
color = good   → Positive values are bonuses
color = bad    → Positive values are penalties

suffix = POSITIVE → High values shown as good
negative_suffix = NEGATIVE → Low values shown as good

no_difference_sign = yes → Don't show +/-, just value
```

**How to use:**
- Read modifier definition
- Understand display semantics
- Verify mod values have correct sign for intent

**Technique 2: Value magnitude analysis**

```
If vanilla uses:
  modifier = 0.01, 0.05, 0.10 (small precise values)

And mod uses:
  modifier = 5 (large integer)

Suggests: Scale mismatch or wrong unit
```

**How to use:**
- Sample vanilla usage of same modifier
- Compare value ranges
- If mod's values are outliers → investigate

**Technique 3: Calculation model understanding**

```
Modifier: X_per_Y_level

Check defines for:
- How many levels exist?
- What are thresholds?
- What's max level?

Then verify:
  Mod uses: +1 per level
  Max level: 5
  Max bonus: +5

  Is +5 balanced? Compare to other perks.
```

**How to use:**
- Understand underlying calculation
- Check if assumptions still valid
- Verify edge cases (min/max levels)

### Handling Ambiguous Changes

**When changes unclear:**

**Ambiguity 1: Is this a rename or redesign?**
```
Approach:
1. Read full definitions of old and new
2. Compare all properties, not just name
3. If properties identical → rename
4. If properties different → redesign
5. If unsure → test both interpretations
```

**Ambiguity 2: Are these related changes?**
```
See: modifier_A changed, modifier_B changed

Approach:
1. Check if both in same system (schemes, culture, etc.)
2. Look for similar naming patterns
3. See if one depends on the other
4. Read commit messages for context
5. If related → investigate as a set
```

**Ambiguity 3: Is this a problem or improvement?**
```
See: decimal precision increased

Approach:
1. Check mod's values still valid at new precision
2. Verify display improves (exact vs rounded)
3. Test if script logic affected
4. If beneficial → no action
5. If problematic → adjust
```

**When in doubt: Investigate deeper. Don't assume.**

### Cross-Version Comparison

**Understanding evolution:**

**Technique: Multi-version diff**
```
Compare modifier across 3 versions:

v1.11: scheme_power_add = { decimals = 0 }
v1.12: scheme_power_add = { decimals = 0 }
v1.13: (removed)
       scheme_phase_duration_add = { decimals = 0, color = bad }

Pattern: Stable for two versions, then reworked
Lesson: Even long-stable systems can be reworked
```

**Technique: Change acceleration detection**
```
Version changes in modifier_definition_formats/00_definitions.txt:

v1.11→1.12: 23 lines changed
v1.12→1.13: 487 lines changed  ← Spike!
v1.13→1.14: 12 lines changed

Pattern: Spike indicates major rework
Lesson: When you see spike, investigate thoroughly
```

**Technique: Pattern lifecycle tracking**
```
Track property naming over versions:

v1.10: *_change_*
v1.11: *_change_*
v1.12: *_growth_*
v1.13: *_growth_*

Pattern introduced: v1.12
Lesson: If you update from v1.11→v1.13, must adapt to this pattern
```

---

## Meta-Investigation: Investigating How to Investigate

### When Standard Approaches Insufficient

**Situation:** You've verified all obvious things, but something still feels wrong.

**Meta-questions:**

1. **What am I not checking?**
   - What types of dependencies exist that I didn't look for?
   - What base game systems exist that mod might implicitly rely on?
   - What indirect dependencies might I have missed?

2. **What assumptions am I making?**
   - Am I assuming mod only uses X, Y, Z types of dependencies?
   - Am I assuming certain files never change?
   - Am I assuming certain systems are stable?

3. **How would I discover unknown unknowns?**
   - What if there's a dependency type I don't know exists?
   - What if mod interacts with base in ways I haven't considered?
   - What if cascading changes affect things I didn't check?

**Approach: Exhaustive discovery**

```
1. Read EVERY file in mod/
   - Not just the obvious ones
   - Look for any base game references
   - Note anything unusual

2. Understand what each file DOES
   - Not just what it contains
   - How does game process it?
   - What base systems does it interact with?

3. Trace all interactions
   - Follow the data flow
   - What calls what?
   - What depends on what?

4. Challenge your mental model
   - Am I sure X doesn't affect Y?
   - Did I verify that assumption?
   - What if I'm wrong?
```

This is DEEP investigation. Not always necessary. But when standard verification feels insufficient, go deeper.

### Self-Correction During Investigation

**Recognize when you're being lazy:**

**Sign 1: Rushing to conclusions**
```
You think: "Only 200 files changed, probably fine"

Stop: Did you READ any of those 200 files?
      Or just assume low count = low risk?
```

**Sign 2: Trusting tools blindly**
```
You think: "Agent said all clear"

Stop: Did you verify agent's claims?
      Or just trust because it's convenient?
```

**Sign 3: Skipping semantic checks**
```
You think: "Modifiers exist, must work"

Stop: Did you verify they mean the same thing?
      Or assume existence = correctness?
```

**Sign 4: Light testing taken as proof**
```
You think: "Loaded game, looks fine"

Stop: Did you test actual features?
      Or just check for errors?
```

**Self-correction:**
- Catch yourself taking shortcuts
- Force yourself to verify
- Question your assumptions
- Go deeper when in doubt

---

## The Mindset: Passionate Developer

**What a passionate developer would do:**

### When Update Drops

**NOT:**
- Quick merge, quick build, ship it

**YES:**
- Read patch notes thoroughly
- Understand what Paradox changed and why
- Investigate impacts methodically
- Verify everything works
- Test comprehensively
- Only then release

### When Issues Found

**NOT:**
- Quick fix, move on

**YES:**
- Understand root cause
- Consider multiple solutions
- Choose best for players
- Test edge cases
- Document reasoning
- Learn for next time

### When Uncertain

**NOT:**
- Assume it's probably fine

**YES:**
- Investigate until certain
- Ask: What would prove me wrong?
- Check for that
- Only proceed when confident

### When Pressed for Time

**NOT:**
- Cut corners, hope for best

**YES:**
- Be honest about verification level
- Document what you didn't check
- Assess risk of unknowns
- Make conscious decision about release
- Or take the time needed

**The standard is thoroughness. No excuses.**

---

## Closing Thoughts

### What This Document Is

**A framework for thinking about compatibility verification.**

Shows you:
- Why clean merges prove nothing
- What types of changes break mods
- How to investigate systematically
- When to use agents for scale
- How to think critically about changes
- What standards to meet

### What This Document Is Not

**A step-by-step script to follow.**

Doesn't give you:
- Hardcoded verification script
- Specific commands to run in order
- List of modifiers to check
- Guaranteed process that always works

### Why the Difference

**Because every update is different.**

- Different systems change
- Different patterns appear
- Different investigation needed
- Different decisions required

**A script can't adapt. You must think.**

### The Core Message

**Git merge succeeds. Builds succeed. Tests might pass.**

**But is the mod actually working? Do features do what they should? Are there silent failures?**

**Only thorough investigation can answer this.**

**Not scripts. Not agents. Not shortcuts.**

**You. Thinking critically. Investigating thoroughly. Verifying ruthlessly.**

**That's the standard.**

---

**Remember: Think. Investigate. Verify. Document.**

**No scripts. No lists. Just thorough, critical thinking.**

