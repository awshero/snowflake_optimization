---
name: grill-me
description: >-
  Interviews the user about a loose idea until they can commit to it — investigating what it can
  answer for itself, then asking the rest in rounds with a recommended answer attached to every
  question. Use when someone says "grill me", "interview me", "poke holes in this", or "help me
  think this through", and also — importantly — when they bring a vague but consequential initiative
  and are about to receive a plan, document, or roadmap built on unexamined assumptions: a cost
  programme, a migration, a build-vs-buy call, a new tool, a target they are about to commit to
  publicly. Prefer grilling first over drafting first whenever the brief is loose and the commitment
  is expensive to reverse. Do not use it when the user has asked for a specific artifact they have
  already scoped — build that instead.
---

# Grill me

*Adapted from two public descriptions of the `/grill-me` skill:
[aihero.dev](https://www.aihero.dev/skills-grill-me), which describes stateless interviewing in
rounds over a question frontier, and
[alphamatch.ai](https://www.alphamatch.ai/blog/grill-me-skill-ai-prompt-code-design-2026), which
describes walking the design tree sequentially, investigating rather than asking where possible, and
attaching a recommended answer to every question. They disagree on batching — rounds versus one at a
time. This implementation takes rounds, because recommended answers make a batch cheap to respond
to, which is the objection sequential asking exists to solve. Neither original SKILL.md is
published; this is an independent implementation.*

## What this is for

A loose idea feels finished long before it is. Someone says "we should build a cost optimization
tool" and it sounds like a decision, but underneath it sit thirty unmade choices — who it serves,
what it must not do, what happens if it works, what the null option is. Draft a plan on top of that
and you get something confident, detailed, and built on sand. Every unmade choice becomes a silent
assumption, and the document's polish hides them.

Grilling makes those choices explicit *before* the artifact exists, by interviewing until the person
can defend each one. The output is not a document. **The output is that they can now commit.**

The failure this prevents is expensive: a plan that is internally coherent and aimed at the wrong
thing. Coherence is not evidence of correctness, and a well-written plan is *harder* to
course-correct than a rough one, because it looks finished.

## First: answer what you can without asking

Before composing a single question, work out which ones you can settle yourself. If the answer is
in the codebase, read it. If it is in the data, query it. If it is in the repo's history, git log
it. If it is a documented fact about a vendor or an API, look it up.

This matters more than it sounds. Every question you ask spends the person's attention, and
attention is the scarce resource in an interview — not time. A question you could have answered
yourself is worse than useless, because it burns that attention *and* signals you have not engaged
with the material. Twelve questions where four were answerable by reading is a session that feels
like an interrogation. Eight questions that genuinely require the person's judgment feels like
thinking together.

So the sequence is: **investigate, then ask what remains.** State briefly what you found and are
therefore not asking about — it shows your work, and it lets them correct you if what you found is
misleading or out of date.

The questions worth asking are the ones only they can answer: intent, constraints, appetite for
risk, what they will not accept, who has to be convinced, what happens politically if it fails.

## The core mechanic: frontier rounds

Ask in **rounds**, where each round is the complete *frontier* — every remaining question whose
prerequisites have already been answered.

```
Investigate   answer everything you can yourself
                 ↓
Round 1       ask every question that is answerable right now
                 ↓  their answers unlock the next layer
Round 2       recompute the frontier, ask all of it
                 ↓
Round 3       ...
                 ↓
Stop          the rest is ungrillable, or they can commit
```

Two rules make this work:

**Ask the whole frontier, not a sample of it.** Drip-feeding hides the shape of the decision and
lets someone answer without seeing what else is at stake. Six to twelve questions in a round is
normal. Seeing the full set is itself informative — the questions they find hard are where the idea
is thin.

**Never ask a question whose prerequisite is unanswered.** Asking "what's the pricing tier
structure?" before "are you charging for this at all?" forces a guess, and the guess anchors
everything downstream. If a question only makes sense given an answer you do not have, it belongs
in a later round. Building the dependency order *is* the analytical work of this skill.

Expect three to five rounds for a substantial decision. Fewer if the idea is more formed than it
looked; more if each answer opens genuinely new territory.

## Every question carries your recommended answer

Do not ask bare questions. For each one, state what you think the answer should be and why, in a
sentence.

This changes the interaction fundamentally, in three ways that all matter:

- **It makes a batch cheap to answer.** Reacting to a proposal is far less work than composing an
  answer from nothing. "Yes / no / actually it's X" moves fast, which is what makes rounds of ten
  questions tractable rather than exhausting.
- **It forces you to have a view.** You cannot recommend without reasoning, and having a position
  is the precondition for the disagreement that makes grilling useful. An interviewer with no
  opinions cannot push back.
- **It surfaces disagreement precisely.** When someone rejects your recommendation, you learn far
  more than a blank answer would have given you — you learn the constraint you did not know about.

Format each as the question, then the recommendation, compactly:

```
3. Who owns the number when a team's spend goes up?
   → Suggest: the team, not the platform group. Central ownership runs out of
     context and goodwill around month four. Push back if your org has no
     existing showback habit.
```

Mark confidence honestly. "I'd guess X but I'm reasoning from a general pattern, not your
situation" is more useful than false certainty, and it tells them which recommendations to scrutinise.

## Running a round

Open the first round by stating what you understood the idea to be, in one sentence, so a
misunderstanding surfaces immediately rather than in round three. Then say what you investigated and
what it told you.

Group questions by theme and number them so they can be answered by reference. Keep each short.

Close every round by inviting two things explicitly:

- **Pushback on scope.** "Tell me if any of these are the wrong questions." The person steering
  scope is a feature, not an interruption — they know things you do not.
- **Partial answers.** "Skip anything you can't answer yet and say why." A skip is information:
  it is either a gap or an ungrillable, and both change what you ask next.

## Pushing back

**If the person agrees with everything, the session has failed.** A grilling where every answer is
accepted produces confidence without validation — worse than no grilling, because now they are
certain.

Disagree when you have grounds. If an answer is vague, say so and ask the sharper version. If two
answers contradict, put them side by side and ask which one gives. If an assumption is load-bearing
and unexamined, name it as load-bearing and ask what happens if it is false.

Vague answers are the main thing to catch, because they feel like progress:

| They say | Press with |
|---|---|
| "We want to save money" | How much, against what baseline, by when, and who verifies it? |
| "It should be easy to use" | Who is the user, and what do they do today instead? |
| "We'll roll it out gradually" | What ships first, to whom, and what makes it stop? |
| "Leadership is supportive" | Supportive enough to fund what, and what would change their mind? |
| "We'll figure that out later" | Later than what? What are you deciding now that depends on it? |

Being agreeable is not kindness here. They asked to be grilled because they want the weak parts
found while they are still cheap to fix.

## Ungrillable questions

Some questions cannot be settled by talking, and continuing to ask them wastes the session. A
question is **ungrillable** when the answer depends on evidence nobody in the conversation has:

- It needs a measurement — a query against real data, a benchmark, a profile.
- It needs a prototype — you genuinely cannot know until something exists.
- It needs another person — a customer, a finance partner, a security owner.

Say so explicitly, state what would resolve it, and stop asking. Then work out whether the decision
can proceed without it. Often it can, under a stated assumption. Sometimes it cannot — and
identifying *that* is the most valuable thing the session can produce, because it converts "let's
plan this" into "we cannot plan this until we run one query," which saves weeks.

Listing the ungrillables is part of the output, not a failure of it.

## Keep the scope small enough to hold

A grilling degrades badly when the subject is too large — the dependency graph stops fitting in
working memory, questions start repeating, and later rounds lose the thread of earlier answers. If
the idea spans several loosely-related decisions, say so and propose splitting it, then grill the
one that unblocks the others first.

A good scope is one decision with its immediate consequences. "Should we build a cost tool
in-house?" is grillable. "How should we run our data platform?" is not — it is four decisions
wearing a trenchcoat.

## Knowing when to stop

Stop when any of these is true:

- **They can state the commitment** in a few sentences, with the trade-offs, and defend each part.
- **Everything remaining is ungrillable.** Further talking cannot help; the next step is evidence.
- **Returns have flattened.** A round produces no answer that changes anything downstream.

Do not pad to hit a round count. Three sharp rounds beats five with filler.

## The output

This skill is **stateless**. It produces no files, no plan, no roadmap. Creating an artifact at the
end quietly undoes the work: the person stops holding the decision and starts holding a document.

Close by reflecting back, briefly:

```
The commitment    what they decided, in their own terms, a few sentences

Load-bearing      the assumptions that, if false, collapse it — named plainly,
                  because these are what to watch

Unresolved        the ungrillables, and what would settle each one

Changed           what they now believe that they did not at the start
```

Keep it short. If the reflection runs long, the session was not decisive enough.

Then stop. If they want a plan built on the commitment, that is a separate request — and a far
better one than it would have been an hour earlier.

## What a good session feels like

Afterwards they should be able to say:

- They disagreed with at least one thing you asked or recommended.
- Questions arrived in few rounds with clear forward motion, not a trickle.
- Something surprised them — usually a decision they had made implicitly without noticing.
- They could now defend each choice to someone sceptical.

If none of those are true, the grilling was too polite to be useful.

## Question banks

Read the bank matching the idea's type. These are starting material, not a script — the real
questions come from what the person actually says.

- **`references/question-banks.md`** — banks for cost and efficiency programmes, platform and
  architecture decisions, build-vs-buy, and new tools or products, each with its dependency ordering
  so you can build the frontier quickly.
- **`references/worked-example.md`** — a full session on a vague cost-programme brief, showing what
  each round looks like, how recommendations are phrased, where pushback lands, and how ungrillables
  get identified. Read this if you are unsure how sharp the questions should be.
