# Worked example

A full session on a vague brief, showing the shape: investigate first, round with recommendations
attached, pushback, an ungrillable found early, and a short close. Abridged — a real session has more
questions per round — but the mechanics are complete.

**The brief:** *"We spend a lot on Snowflake and leadership wants it down. I'm thinking we build an
internal tool to find waste. Can you help me plan it?"*

Note what did *not* happen: no plan was drafted. That brief contains at least a dozen unmade
decisions, and a roadmap written on top of it would have looked authoritative and been aimed at a
guess.

---

## Investigate first

> Before I ask you anything — I looked at what I could reach.
>
> Your repo has no cost tooling yet, so this is greenfield. `ACCOUNT_USAGE` is queryable from the
> role in your profile, so measurement is not blocked. And Snowflake bills warehouse-seconds rather
> than query-seconds, which means idle time is billable and invisible in per-query views — that is
> usually the largest recoverable line, so I am assuming it is in scope unless you say otherwise.
>
> What I could not find: your actual spend, whether you are on a capacity commitment, and who has
> been asked to deliver this. Those need you.

Four questions removed from the interview by twenty minutes of reading. That is the trade this
section exists to make.

---

## Round 1

> First, my understanding: **leadership wants Snowflake spend reduced, and you are proposing to build
> tooling to find the waste.** Correct me if the goal is really "stop the growth" rather than "cut
> the bill" — different programme.
>
> **The ask**
>
> 1. What is the annual spend, and does your number reconcile to the invoice?
>    → *Suggest: get this exact before anything else. Estimates that do not tie out get challenged
>    the first time finance looks, and then nothing you produce is trusted.*
>
> 2. Is the target a percentage, an absolute number, or "make growth stop"?
>    → *Suggest: percentage against a reconciled baseline. Absolute targets break when the business
>    grows and you end up penalised for someone else's success.*
>
> 3. Who asked for this, and what happens to them if it does not land?
>    → *Suggest: this determines your real deadline more than any stated one.*
>
> **The money**
>
> 4. Is there a capacity commitment, and when does it end?
>    → *Suggest: assume yes at your scale. If so, cutting consumption may save zero cash this year —
>    the commit just burns down slower. This changes what you promise.*
>
> 5. Has anyone in finance agreed how a saving gets recognised?
>    → *Suggest: settle this in week one, not month nine. Programmes die here.*
>
> **The tool**
>
> 6. Build, buy, or both? Commercial platforms exist and claim 10–20% on compute.
>    → *Suggest: buy observability, build attribution and enforcement. Vendors cannot own your
>    tagging taxonomy or your dbt conventions, and that is where the durable savings are.*
>
> 7. What is your appetite for the tool *changing* things versus only reporting them?
>    → *Suggest: report-only for the first quarter. Earn the mandate before you spend it.*
>
> Push back if any of these are the wrong questions, and skip anything you cannot answer yet.

Seven questions, each answerable now, none depending on another's answer. Note that #6 asks
build-vs-buy *before* any design work — asking "what should the architecture be?" first would have
presumed the build.

---

## What came back

> 1. ~$40M. Number's from the Snowflake console, hasn't been reconciled.
> 2. 20% by end of Q2.
> 3. CFO asked. VP Data owns it. Honestly not sure what happens if we miss.
> 4. Yes there's a commit, don't know the terms.
> 5. No.
> 6. Build. We've been burned by vendors, it's political, don't push.
> 7. **Disagree with your suggestion.** We have three months, report-only wastes the first one.

Answer 7 is the most valuable thing in the session — a rejected recommendation revealing a real
constraint. Answers 4 and 5 have just made the deadline questionable, and that has to be said.

---

## Round 2

> Two things changed, one of which I need to press on.
>
> **The commitment problem.** You have a commit with unknown terms and no agreement on how savings
> are recognised. That combination can mean a 20% consumption cut produces **zero** cash saving by
> Q2, and you would have delivered real engineering and still missed. This is the single largest
> risk to the programme and it is not an engineering risk.
>
> 8. Can you get the commit terms — amount, end date, rollover — this week?
>    → *Suggest: this is the highest-value hour available to you right now. Everything downstream
>    depends on it and nobody else will do it.*
>
> 9. If the answer is "no cash saving until renewal", does the CFO still want this?
>    → *Suggest: ask directly. Framed as "credits avoided now, cash at renewal" it is usually still
>    a yes — but you want that on the record in month one, not month nine.*
>
> **On report-only — you are right and I was wrong.** Three months does not allow a quiet quarter.
> Revised: skip report-only, but restrict early changes to the levers that revert in seconds and
> touch no data — auto-suspend, cluster minimums, scaling policy, statement timeouts. You get
> savings in week two and the blast radius stays near zero.
>
> 10. Does that split work, or do you need to touch storage and retention too?
>     → *Suggest: stay off retention entirely for now. Lowering it destroys history immediately and
>     is the one change here you cannot undo.*
>
> 11. Who can approve a warehouse config change without a change-advisory cycle?
>     → *Suggest: if the answer is "nobody quickly", that process is your real constraint, not the
>     engineering.*
>
> **Build, noted — not relitigating.**
>
> 12. Two engineers for three months, or is this someone's 20%?
>     → *Suggest: 20% will not hit 20% by Q2. Worth saying to the VP now rather than in month two.*

Round 2 does four things worth copying: it escalates a risk that only became visible after round 1,
it **concedes a point explicitly and changes its recommendation**, it drops a settled question rather
than relitigating, and it turns a vague deadline into a resourcing question.

---

## The ungrillable, named

> One thing we cannot settle by talking: **whether your top warehouses can actually be downsized.**
> That depends on spill, queueing and load percent in your account, and no amount of discussion
> substitutes for the query. It is roughly twenty minutes of work against `QUERY_HISTORY` and
> `WAREHOUSE_METERING_HISTORY`.
>
> It does not block planning — the programme shape holds either way. But it does block committing to
> a number, because "20%" is currently a target, not an estimate.

State it, size it, say whether it blocks. Then stop asking.

---

## The close

> **The commitment.** Build in-house, two engineers, three months. Week one is reconciling the
> baseline and getting the commit terms. Changes start in week two, restricted to config levers that
> revert in seconds and touch no data. Retention and anything that destroys data is out of scope
> this quarter. Report credits avoided and dollars realised as two separate numbers from the start.
>
> **Load-bearing assumptions.** That the commit terms permit savings to be recognised before
> renewal — currently unknown and the largest risk. That warehouse config changes can be approved in
> days rather than weeks. That two engineers are genuinely available.
>
> **Unresolved.** Whether 20% is achievable — needs the spill and utilisation queries. Whether the
> CFO accepts credits-avoided as the Q2 measure — needs a conversation.
>
> **What changed.** You came in planning a waste-finding tool. The binding constraint turned out not
> to be finding waste but proving the saving is real, which is a contract question, not an
> engineering one. The tool is still worth building; it is no longer the first thing.

Four sentences per section. If the close needs more than that, the session was not decisive enough —
and note that the last line is the point of the whole exercise.

---

## What made this session work

- **Investigation removed four questions** before the interview started.
- **Recommendations on every question** meant seven questions took one reply, not seven exchanges.
- **A rejected recommendation** (report-only) surfaced the real constraint and the interviewer
  conceded plainly rather than defending the position.
- **A risk was escalated when it appeared**, not saved for the summary.
- **The ungrillable was named and sized** instead of being talked around.
- **The conclusion inverted the brief** — which is what a grilling is for. If the plan at the end
  matches the plan at the start, nothing was tested.
