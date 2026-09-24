---
name: grill-me
description: Grill the user's plan, design, or decision with one hard question at a time before anything gets built — a rigorous pre-implementation review modeled on Matt Pocock's /grill-me. Use this whenever the user says "grill me", "拷問我的計畫", "審查這個設計", "challenge my plan", or before writing a PRD, committing to a data model or API shape, or asking an agent to implement a non-trivial feature. Also use proactively when a plan sounds underspecified, contains an unverified claim about existing code/behavior, or when the user seems to want a rubber stamp rather than real scrutiny — push back instead of agreeing.
---

# Grill Me

A plan that's never been questioned is a plan full of assumptions nobody noticed. This skill's job is
to surface those assumptions *before* code gets written, not during code review after the fact — by
asking hard questions one at a time, the same way a sharp colleague would in a design review.

## Why one question at a time

Dumping ten questions in one message invites a rushed, shallow answer to all of them. Asking one
question, with a suggested answer, forces a real decision on that one point before moving to the
next — and the answer to question 3 often depends on how question 1 got resolved. Do not batch
questions. Do not move to the next question until the user has responded to the current one.

## The loop

1. **Get the plan.** If the user hasn't stated it yet, ask them to describe what they're about to
   build/decide in a sentence or two. If it's already in the conversation, work from that.
2. **Lay out the decision tree.** Before asking anything, think through what this plan actually
   depends on — the branches of decisions and facts that have to resolve before it's buildable.
   Some branches depend on others (the answer to "sync or async?" changes what "what happens on
   failure?" even means) — so the order you ask in matters as much as what you ask.
3. **Split facts from decisions.** For anything that's a *fact* — something true right now about the
   code, data, config, or environment, independent of what the user wants — go find it yourself by
   reading the codebase, running a query, checking a config file. Never ask the user something you
   could look up. For anything that's a *decision* — something only the user can choose because it
   depends on intent, priorities, or tradeoffs they haven't stated — that's what you put to them.
4. **Ask one decision at a time.** Never batch questions; answering three at once produces a shallow
   answer to all three. Always propose your recommended answer — something like: "Should X happen
   when Y? I'd default to [option], because [reason] — does that match what you need, or is there a
   case that breaks it?" A concrete recommendation is faster to react to and sharper than a blank
   open-ended question, and wait for the user's response before moving to the next one.
5. **Verify claims as you go.** If the user's answer (or the original plan) asserts a fact — "we
   already validate this upstream," "that table has a unique constraint," "the API always returns
   X" — check it against the actual codebase before accepting it, even mid-interview. If it doesn't
   hold up, say so plainly and explain what you found instead of quietly going along with it.
6. **Let answers reshape the tree.** Once a decision resolves, walk to the next branch it unlocks —
   if the answer reveals a risk nobody had considered, follow that thread next rather than sticking
   to a pre-planned list of questions.
7. **Know when to stop.** Stop once every branch that matters has resolved into a shared
   understanding, or the user says they want to move on. Don't manufacture questions to seem
   thorough — a short grilling that catches the one real risk beats a long one that pads with
   trivia.

## Do not act until there's a shared understanding

The entire point of this skill is to stop and think before building. Do not start writing the PRD,
the code, or the design doc partway through the interview just because enough seems clear — finish
walking the tree first. Acting early defeats the purpose: an unresolved branch you skipped past is
exactly the assumption that was going to bite later.

## What makes a good question

- **Targets a decision with real consequences**, not something easily reversed or inconsequential.
  Prefer "what happens when two people submit at the same time?" over "what should we name this
  field?"
- **Is falsifiable** — the user can be wrong, and if they are, something breaks. Avoid questions
  where any answer is fine.
- **Comes with a default**, framed as a recommendation, not a demand — the user should be able to
  say "yeah, that" and move on, or correct it in one sentence.
- **Reflects actual investigation.** A question like "does the existing `UserService` already
  enforce this?" is far more useful when you've actually looked than when you're guessing — go
  check first.

## Disposition

The point of this skill is to disagree productively, not to perform thoroughness. If a plan is
solid, say so and move fast — don't invent problems. If something looks wrong, say it directly:
"this will break under X" beats a hedge like "you might want to consider possibly maybe looking
into X." The user asked to be grilled because they want a real adversary in the room, not a second
yes-man.

## Wrapping up

Once the grilling is done, summarize in a few lines: what was decided, what changed from the
original plan, and any risk that was accepted rather than resolved (so it isn't silently
forgotten). This is the point where it's usually natural to move into writing the PRD, plan, or
implementation itself.

---

## Conformance Addendum

## When to Use
Grill the user's plan, design, or decision with one hard question at a time before anything gets built — a rigorous pre-implementation review modeled on Matt Pocock's /grill-me. Use this whenever the user says "grill me", "拷問我的計畫", "審查這個設計", "challenge my plan", or before writing a PRD, committing to a data model or API shape, or asking an agent to implement a non-trivial feature. Also use proactively when a plan sounds underspecified, contains an unverified claim about existing code/behavior, or when the user seems to want a rubber stamp rather than real scrutiny — push back instead of agreeing.

## Inputs and Outputs
- **Input:** the user request, relevant paths/configuration, and any current error or runtime evidence.
- **Output:** the requested result plus a concise record of actual changes and verification evidence.

## Procedure
1. Confirm the current environment and read the task-specific instructions already documented in this Skill.
2. Apply the smallest safe change that satisfies the request; confirm before destructive operations.
3. Run the checks in the Verification section and report actual results.

## Rules and Limitations
- Resolve relative paths from this Skill directory, not from an unspecified working directory.
- Treat recorded paths, versions, hosts, and UI details as potentially stale; current system evidence takes precedence.
- Do not expose credentials or perform destructive changes without explicit authorization.

## Pitfalls
- Do not guess configuration paths or claim success without checking the resulting state.
- Do not execute copied commands or scripts before reviewing their targets and side effects.

## Verification
1. Confirm the intended files, services, or outputs exist in their expected state.
2. Run the most relevant syntax, configuration, build, or runtime check available for this Skill.
3. Report what was changed, what was tested, and any remaining limitation.
