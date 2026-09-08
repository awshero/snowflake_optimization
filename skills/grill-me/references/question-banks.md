# Question banks

Starting material, not a script. The real questions come from what the person says — these exist so
you can build a dependency order quickly instead of inventing structure under time pressure.

Each bank is ordered by **layer**, not importance. Layer 1 questions are answerable immediately.
Layer 2 questions only make sense once layer 1 is answered. That ordering is the frontier.

Before using any of these: check which you can answer yourself by reading the repo, querying the
data, or looking something up. Delete those. Ask only what requires their judgment.

## Contents
- [Cost and efficiency programmes](#cost-and-efficiency-programmes)
- [Platform and architecture decisions](#platform-and-architecture-decisions)
- [Build versus buy](#build-versus-buy)
- [New tool or product](#new-tool-or-product)
- [Cross-cutting questions worth asking about anything](#cross-cutting-questions-worth-asking-about-anything)

---

## Cost and efficiency programmes

**Layer 1 — what is actually being asked**
- What is the number today, and how confident are you in it? Does it reconcile to an invoice?
- Who asked for this, and what happens to them if it does not land?
- Is the goal a percentage, an absolute figure, or "make the growth stop"? These are different programmes.
- What is the deadline, and is it real or aspirational?
- What is the null option — what happens if you do nothing for a year?

**Layer 2 — how savings become real** *(needs: the number, the goal)*
- Is there a capacity commitment? When does it end?
- If consumption falls, does the invoice fall — or does the commit just burn down slower?
- Who signs off that a saving is real? Have you talked to them yet?
- Are you reporting credits avoided, dollars realised, or both? They diverge.
- Does the target account for growth? A flat bill against 40% more workload is a large win that looks like failure.

**Layer 3 — who does the work** *(needs: the goal, the realisation mechanism)*
- Does the platform team change other people's workloads, or do teams change their own?
- Can teams currently see their own spend? If not, that is the first deliverable, not an enabler.
- What is your appetite for automated change versus proposals a human approves?
- Which changes are you willing to make without an owner's sign-off, and which never?
- Who has been burned by a previous efficiency push, and what did it cost them?

**Layer 4 — durability** *(needs: ownership model)*
- What stops this drifting back in twelve months?
- What gets measured monthly after the programme ends, and by whom?
- If someone downsizes a warehouse and it causes an incident, what happens to them? *This one answer explains most estates.*
- What would make you stop the programme early?

**Commonly ungrillable here:** the actual spend decomposition, the real idle ratio, whether a
specific warehouse can be downsized. All need queries. Say so and move on.

## Platform and architecture decisions

**Layer 1 — the forcing function**
- What breaks if you keep the current design for two more years?
- Is this driven by cost, reliability, speed of delivery, or a compliance deadline? Rank them.
- Who is asking — and are they asking for this design, or for an outcome they think it delivers?
- What has already been decided that is not up for discussion?

**Layer 2 — constraints** *(needs: the forcing function)*
- What must keep working throughout, with no downtime?
- What is the migration budget in engineer-weeks, honestly?
- Which team owns the result afterwards, and have they agreed?
- What is the rollback if this is wrong six months in?

**Layer 3 — the shape** *(needs: constraints)*
- What is the smallest version that proves the approach?
- What are you deliberately not building, and how will you resist building it anyway?
- Which of your assumptions about scale or load is least evidenced?
- What does this make harder that is currently easy?

**Layer 4 — consequences** *(needs: the shape)*
- Who has to change how they work, and do they know yet?
- What new operational burden does this create, and who carries it at 3am?
- In two years, what will someone want to change, and how hard will this make it?

## Build versus buy

**Layer 1 — the requirement**
- What specifically must this do that nothing you have does today?
- Have you used any vendor in this space? What made you stop, or not start?
- Is the differentiating part the thing you would build, or the thing you would buy?

**Layer 2 — the honest comparison** *(needs: the requirement)*
- What is the total build cost including a year of maintenance, not just v1?
- Who maintains it when the person who built it leaves?
- What does the vendor do that you would not think to do?
- What would you have to give a vendor — data, access, integration surface — and is that acceptable?

**Layer 3 — the hybrid** *(needs: the comparison)*
- Which parts are genuinely specific to you, and which are commodity?
- Could you buy the commodity and build only the specific part? What breaks if you do?
- If you buy and it fails in eighteen months, what is the exit cost?

## New tool or product

**Layer 1 — the user**
- Who has this problem, specifically enough to name three of them?
- What do they do today instead? If the answer is "nothing", is the problem real?
- How did you learn they have it — did they say so, or did you infer it?

**Layer 2 — the wedge** *(needs: the user)*
- What is the one thing it must do well for someone to switch?
- What is the first version, and who is the first user by name?
- How would you know within a month that this is not working?

**Layer 3 — the shape of success** *(needs: the wedge)*
- If this works, what number moves, by how much, by when?
- What does it cost to run at ten users, and at a thousand?
- What is the failure mode that would make someone stop using it after a week?

## Cross-cutting questions worth asking about anything

Reach for these when a bank runs dry, or when an idea feels under-examined and you cannot say why:

- **The null.** What happens if you do nothing? Surprisingly often the honest answer is "not much",
  and that is worth knowing before committing a quarter.
- **The load-bearing assumption.** Which single belief, if false, collapses this? Have you tested it?
- **The dissenter.** Who disagrees with this, and what is their best argument? If nobody disagrees,
  either it is obvious or nobody has looked at it properly. Work out which.
- **The reversal.** How hard is this to undo? Cheap-to-reverse decisions deserve less grilling —
  spend the session on the one-way doors.
- **The second order.** If this works exactly as intended, what does that cause? Who adapts, and how?
- **The tell.** What would you see in three months that would tell you this was a mistake — early
  enough to matter?
- **The unstated success criterion.** What would make you privately consider this a failure even if
  the stated metrics were met?
