---
name: rebartracker-workflow
description: Carter's work preferences and session rules for RebarTracker development. Use this skill whenever working on RebarTracker tasks in Claude Code or planning sessions in Claude.ai. Covers commit discipline, the [IMPLEMENT NOW] tagging system, separation of responsibilities between Claude and Carter, red flags that should pause work, communication style, and credential/push handling. Apply these rules automatically — do not ask Carter to repeat them each session.
---

# RebarTracker Workflow

Rules for how Carter and Claude work together on RebarTracker. Apply all of these automatically without being asked.

## Who does what

**Claude writes, Carter tests.**

- Claude Code writes code, runs git commands, commits, and pushes
- Carter tests in the browser and DevTools Console
- Claude never assumes a test passed without Carter explicitly confirming
- Claude never marks a bug as "verified" based on static analysis alone unless Carter has agreed that's sufficient for that specific case

**Carter is not a developer by trade.** Keep explanations direct and practical. Avoid jargon unless Carter uses it first. When explaining what a change does, describe the real-world behavior, not the code mechanics.

## Commit discipline

- **One commit per discrete fix.** Never bundle two unrelated fixes into one commit. If two things broke and both got fixed, that's two commits.
- **Push to master immediately after each commit.** Don't batch up commits and push at the end. One fix = one commit = one push.
- **Commit messages are descriptive.** Not "fix bug" — "Fix Bug 2: Split and Overflow dropdowns show all slots with disabled state". Future you should be able to read the log and know exactly what each commit changed.
- **Never leave debug console.log statements in committed code.** Remove all diagnostic logging before committing.
- **Syntax check before every commit.** Claude Code should catch syntax errors before they land on master.
- **Session-close marker commits.** At the end of each verified session, commit an empty marker: `git commit --allow-empty -m "Session N verified: [short description]" && git push origin master`

## Message tagging system

When Carter sends a message that contains plans, examples, or test steps alongside actual implementation requests, Claude Code must read the tag first and respond accordingly.

| Tag | Meaning | Claude's response |
|-----|---------|-------------------|
| `[IMPLEMENT NOW]` | This is a work order. Execute it. | Proceed with the task |
| `[DISCOVERY ONLY]` | Read and report findings. No code changes. | Report only, wait for explicit proceed |
| `[FOR CONTEXT]` | Background info. No action needed. | Acknowledge, hold for reference |
| `[VERIFY — no changes]` | Check something and report. Don't fix it. | Inspect and report |
| `[PAUSE]` | Stop what you're doing. Wait for instruction. | Stop immediately |

**If no tag is present:** use judgment. If the message is clearly a task ("fix this bug"), proceed. If it's ambiguous (numbered list that could be a plan OR tasks), ask for clarification before doing anything.

**Numbered lists are NOT automatically work orders.** A test matrix, a plan, a list of scenarios to consider — these look like tasks but are often context or verification steps for Carter to do in the browser. When in doubt, ask: "Is this list for me to implement, or is this a test plan for you to run?"

## Discovery before implementation

For any non-trivial change, run a discovery pass first:

1. Read the relevant code and report findings (function names, line numbers, current behavior)
2. State the proposed approach clearly
3. **Wait for Carter to say "proceed" or "go ahead" before writing any code**

Skip discovery for:
- Trivial one-line changes where the fix is unambiguous
- Explicitly scoped tasks where Carter has already described exactly what to change and where

Never skip discovery and jump to implementation on anything that involves:
- A new modal or UI section
- A new Firebase path or data schema
- Changes to how existing data is stored or read
- Any transfer, merge, or clear operation that modifies inventory

## Session separation — paste vs test

**Carter tests in the browser. Claude Code does not.**

When Claude Code finishes a commit, it gives Carter a test plan. That test plan is for Carter to execute — it is NOT a list of tasks for Claude Code to work on next.

Claude Code should clearly label the handoff:

> "Committed and pushed. Here's what to test in the browser before we move on: [test steps]"

After that, Claude Code waits. It does not proceed to the next task until Carter reports results.

**If Carter pastes a test matrix into Claude Code by mistake:** Claude Code should recognize it as a test plan (not a work order), tell Carter "This looks like a test plan for you to run in the browser — I'll wait for your results before doing anything," and stop.

## Red flags — stop and ask

These situations should always pause Claude Code and prompt a question before proceeding:

- **Design decision not explicitly approved.** If implementing a feature requires choosing between options (e.g. Option A vs B) that Carter hasn't picked yet — stop and ask.
- **Scope creep.** If the fix requires changing more than 3-4 related functions, flag it before proceeding. "To fix X I'll also need to touch Y and Z — is that okay?"
- **Unexpected behavior during discovery.** If the code doesn't match what was expected (function missing, different structure than assumed), report before attempting to adapt.
- **Large refactor opportunity.** If Claude Code sees a chance to clean up existing code significantly while doing something small — don't. Do the small thing cleanly. Note the refactor opportunity separately.
- **Any delete or clear operation on real data.** Flag what data will be removed and ask for confirmation. This includes clearing slots, deleting Area stacks, wiping Firebase paths, or anything that modifies `store`, `inventory`, `area1Stacks`, or `area22Stacks` in bulk.

## Push credential handling

The Mac's Keychain caches the GitHub PAT. After a successful manual push from Terminal, Claude Code's subsequent pushes work silently.

**If a push fails with "Device not configured" or similar auth error:**
1. Tell Carter: "Push failed — credentials not cached in this shell. Please run `git push origin master` once from your own Terminal window to refresh the Keychain, then I can push silently going forward."
2. Do NOT keep retrying the push. One notification is enough.
3. The commit is safe locally. Nothing is lost.

**If Terminal asks for username/password:**
- Username: `Cjwoods03`
- Password: the GitHub PAT (starts with `ghp_...`) — Carter inputs this, Claude never handles tokens

## Long prompt handling

Claude Code's input field has a display limit but accepts long text. If a prompt is long and Carter is worried it got cut off:
- Claude Code should address ALL sections of the prompt in its response
- If Claude Code only responds to part of a long prompt, assume truncation and ask Carter to paste the remaining section as a follow-up
- Carter can also save long prompts as a .txt file and tell Claude Code: "Read ~/Desktop/prompt.txt and execute it"

## Communication rules

- **Report before fixing.** For discovery passes, show what you found before touching any code.
- **One thing at a time.** Don't fix three bugs in one response. Fix one, commit, push, let Carter test, then move to the next.
- **Say what you changed.** After every commit, give a plain-English summary of what changed and why — not the code diff, the behavior difference.
- **Don't over-explain.** Carter doesn't need a paragraph about why a pattern exists. One sentence of context is enough unless Carter asks for more.
- **When something unexpected happens, say so.** Don't paper over surprises with a workaround that silently changes behavior. "I found X when I expected Y — here's what I did instead and why" is always better than silently adapting.

## What Claude Code should NOT do without being asked

- Restructure or reorganize existing code (refactor, rename variables, move functions) unless the task explicitly requires it
- Add error handling beyond what's needed for the specific fix
- Write unit tests or create test files
- Create new files other than the ones explicitly requested
- Change formatting, indentation, or code style in sections not being modified
- Suggest switching to a different architecture, framework, or tool
- Open the browser or navigate to the live site
- Make assumptions about what Carter "probably wants" for unspecified behavior — ask instead

## Playbook reference

Ten-session development roadmap lives at:
`/mnt/user-data/outputs/RebarTracker-ClaudeCode-Prompts.md`

When Carter says "run Prompt 2B" or "move to Session 3," locate that prompt in the playbook and execute it. When Carter says "move to the next session," the last prompt in the current session is always the verify/close-out prompt.

## Session-close reminder

At the end of every working session, Claude Code should proactively say:

> "Before we wrap — here's what landed this session so you can update the rebartracker-context skill:
> [list of commits with short descriptions]
> [any new function locations, schema changes, or architecture additions]
>
> Paste this to Claude Code to update the skill:
> `Edit ~/RebarTracker/.claude/skills/rebartracker-context/SKILL.md — under Session N in the session history, append: [content]. Commit with "docs: update context skill for Session N" and push.`
>
> Also re-upload the updated SKILL.md to Claude.ai Skills settings."
