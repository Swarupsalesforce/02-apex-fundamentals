# DECISIONS
**Owner:** Soumya · Started Day 20, backfilled from Days 8–19

Every design ruling, with the alternative rejected and the reason. These are interview answers — "why this and not that" is the question, and this file is the answer sheet.

**Format:** what I chose · what I rejected · why · when the answer would flip.

---

## TRIGGER DESIGN

**Before-trigger for tier assignment** *(Day 8)*
Rejected: after-trigger with DML. Setting a field on the record being saved needs no DML — the assignment rides along with the save. After-trigger would mean a second write for no reason.
*Flips when:* the field belongs on a different record.

**One-line trigger, logic in a handler** *(Day 10)*
Rejected: logic inside the trigger file. Triggers can't be called by anything but DML, so a Batch job could never reuse the logic. Salesforce also gives no order guarantee between multiple triggers on one object, so one trigger per object is forced — and that file has to be a switchboard, not a workshop.
*Flips when:* never, for production code.

**Recalculate tier on every update, no `oldMap` change detection** *(Day 10)*
Rejected: comparing old and new revenue to skip unchanged records. The calculation is zero SOQL and zero DML, so skipping saves nothing. It's also self-healing — if someone hand-edits the tier field, the next save corrects it.
*Flips when:* the work is expensive (a query, a callout, an email).

**Handler methods take collection parameters, never read `Trigger.new` internally** *(Day 10)*
Rejected: reaching for `Trigger.new` inside the logic method. A method that does that **can only ever be called by a trigger** — no Batch job, no scheduled class, no button, no direct unit test has a `Trigger.new`. The parameter is what makes four callers possible.
*Flips when:* never.

**`rollUpContactCount(Set<Id>)` rather than `(List<Contact>)`** *(Day 11)*
Rejected: passing the contacts. On delete there is no `Trigger.new` to pass — only `Trigger.old` has the parent Ids. A Set of affected parents works for insert, update, delete and undelete identically.
*Flips when:* the method needs the child records themselves, not just the parents'.

---

## ROLLUPS

**Collect parent Ids from both `Trigger.new` and `Trigger.old` on update** *(Day 11)*
Rejected: `Trigger.new` only. Reparenting moves a child from A to B — `Trigger.new` knows B, only `Trigger.old` knows A. Miss it and A keeps a phantom child forever, silently.
*Flips when:* never, for any rollup.

**Seed every affected parent at zero before the aggregate query** *(Day 11)*
Rejected: relying on the aggregate alone. Deleting the last child means the aggregate returns no row for that parent, so it never enters the map and never gets updated — the stale count survives. Only visible on the transition to zero, which is why it hides for months.
*Flips when:* never.

**`Map<Id, Account>` for two rolled-up fields, not two parallel maps** *(Day 17)*
Rejected: `Map<Id, Decimal>` plus `Map<Id, Date>`; also rejected a wrapper class. Building the Account record directly means one map, one seed loop, and `update map.values()` at the end.
*Flips when:* four or five fields, or when the values need logic attached — then a wrapper earns its place.

---

## VALIDATION

**Zero and negative revenue → null tier, not Bronze** *(Day 18, preserving Day 12)*
Rejected: letting zero fall through to Bronze. Bronze implies a real but small customer; zero revenue means unknown, which is different information. Preserved deliberately during the strategy refactor rather than changed by accident.
*Flips when:* the business says zero-revenue accounts should be worked as Bronze.

**Kept the negative-revenue branch in `assignTiers` after adding validation** *(Day 12)*
Rejected: deleting it as dead code. `assignTiers` is public — a Batch job or test can call it with records that never passed through validation. **A public method defends itself rather than trusting its callers.** Validation at the boundary, defensiveness in the logic.
*Flips when:* the method becomes private and has exactly one caller.

**Bronze contact limit counts the incoming batch, not just the stored rollup** *(Day 12)*
Rejected: reading `Count_Contacts__c` alone. That field is written by an after-trigger, so a before-trigger sees a value that excludes everything arriving in this transaction — 10 contacts inserted at once onto an empty account all see 0 and all pass.
*Flips when:* never, for any validation built on a rollup field.

**Bronze limit runs on insert only** *(Day 12)*
Rejected: insert and update. On update, existing contacts appear in `Trigger.new` and get counted as incoming on top of the stored count — blocking legitimate edits to contacts that were already there. Reparenting validation is a separate story, deliberately deferred.
*Flips when:* the incoming tally can distinguish new children from edits to existing ones.

**Treat null contact count as 0, don't skip the check** *(Day 12)*
Rejected: `if (count == null) continue;`. That guard was written to avoid a null crash but skipped the validation entirely — so a brand-new Bronze account, whose rollup field is still null, accepted unlimited contacts. Substituting a number lets the check run.
*Flips when:* null genuinely means "not applicable" rather than "none yet."

**`> 5` not `>= 5` after adding the incoming count** *(Day 12)*
The threshold moved when the arithmetic changed. 5 stored + 1 incoming = 6, which is over the limit; 4 + 1 = 5, which is at it. Off-by-one lives exactly at the boundary.

---

## CLASS DESIGN

**Query in the service, never in the wrapper** *(Day 15–16)*
Rejected: letting `AccountHealth` fetch its own data. The wrapper takes an Account it's handed, so it can be built from a trigger, a Batch scope, a query, or a test — and tested with no DML at all. The service is where SOQL belongs.
*Flips when:* never; a data holder that queries is untestable in isolation.

**`isUnderCovered()` lives on the wrapper, not in the service's `if`** *(Day 15)*
Rejected: writing the comparison where it's needed. "Bronze with fewer than 2 contacts" is business policy — one place owns it, so changing it to 3 is one line instead of hunting six files.
*Flips when:* the rule is used exactly once and will never move.

**Sort ascending on contact count — emptiest first** *(Day 16)*
`compareTo` returns negative when `this` has fewer contacts. The manager wants a worklist, and the emptiest accounts are the most urgent.
*Flips when:* the screen wants best-covered first — reverse the signs.

**Added `WHERE Customer_Tier__c = 'Bronze'` to the service query** *(Day 16)*
Rejected: querying all accounts and filtering in Apex. At 50,000 accounts, pulling everything back to discard most of it hits row limits. **Accepted cost:** the Bronze rule now lives in two places — the query and `isUnderCovered()`. A selector layer is the real fix.
*Flips when:* the policy changes to include Silver and someone edits one place but not the other.

**`throw` instead of returning null from `getHealthFor`** *(Day 17)*
Rejected: returning null. The caller asked for a specific account — if it isn't there, that's a broken assumption, not a valid answer. Returning null means the crash lands twenty lines later in the caller's code, pointing at the wrong line. A custom `AccountHealthException` also lets a caller handle *this* failure differently from a query or DML failure.
*Flips when:* emptiness is a legitimate outcome — `getUnderCoveredAccounts()` returns an empty list, and that's information, not a failure.

**Strategy interface over if/else for tier rules** *(Day 18)*
Rejected: `if (type == 'Partner')` inside `assignTiers`. **The if/else is better code for two fixed types** — one readable place, fewer files. What tipped it was one sentence in the requirement: more account types are coming. With strategies, a new type is a new file plus one line in the factory, and nothing existing is edited.
*Flips when:* the set of variants is fixed and small. Then if/else wins on readability.

---

## TESTING

**Unit tests call strategies directly with hand-built records** *(Day 19)*
Rejected: testing every threshold through DML. `calculateTier` takes an Account and returns a String — no query, no DML, no `Trigger.new`. So twenty threshold cases run in milliseconds with no governor cost. Creating 45 contacts to test three thresholds would take seconds and prove nothing extra.
*Flips when:* the behaviour under test *is* the database interaction.

**Integration tests only insert and assert — never call a method directly** *(Day 19)*
Rejected: calling `assignTiers` by hand inside an integration test. That skips the very chain the test exists to prove, and re-tests logic the unit tests already cover. The point is that contact insert → rollup → account update → trigger → tier runs *by itself*.
*Flips when:* never — the moment you call a method directly, it stopped being an integration test.

**Assert on a re-query, not on the in-memory variable** *(Day 9)*
Rejected: asserting on the list you passed to `insert`. A before-trigger's changes do come back to your variable, so the test can pass while proving nothing about the database. An after-trigger's changes to *other* records never come back at all.
*Flips when:* the method mutates the list you handed it — then the list *is* the result.

---

## TEMPLATE FOR NEW ENTRIES

```
**What I chose** *(Day N)*
Rejected: the alternative. Why the choice wins.
*Flips when:* the condition that would change the answer.
```

One or two a day. The "flips when" line is the one interviewers care about — it proves the decision was reasoned rather than copied.

**Declined a shared TriggerHandlerBase** *(Day 20)*
Rejected: extracting run() into a virtual base class with hooks. Two handlers
means paying a file and splitting each class's story across two places to save
four if-lines. Not worth it yet.
*Flips when:* several objects need the same routing, or something cross-cutting
is needed in all of them — a recursion guard, a bypass switch for data loads,
run-once logic.

Day 23 quiz issue
"Strategies let you add a variant without editing code that already works."
"Interfaces promise behaviour on objects — static methods don't belong to objects."
"The object decides which code runs, not the variable's type. That's polymorphism."

Day 24 
batch reuses assignTiers with zero new business logic

Day 25
Thsings i missed Diffrence in between @future Queueable, batch and Schedulable

@future — you missed the one case where it's genuinely required. "Thin and few lines" is a preference, not a rule. The real one: you cannot make a callout from a trigger synchronously. Salesforce forbids it. So a trigger that needs to hit an external system must go async, and @future(callout=true) is the classic answer. Queueable also works, but this is the scenario where async isn't optional.

Queueable — the 50,000 limit isn't the boundary. Queueable runs in one transaction, so it's bound by all the normal limits: 100 SOQL, 150 DML, 10,000 records per DML statement, 6MB heap. It isn't "up to 50,000." It's "whatever fits in one transaction."

Batch — the trigger isn't a record count, it's transaction limits. You said "more than 50,000." Closer: when the work can't fit in one transaction. That might be 20,000 records with heavy processing, or 500,000 with light. What Batch buys is many transactions with fresh limits each, not a bigger single one.

Scheduled — correct, and worth adding the thing you'd say in an interview: Schedulable doesn't do work, it starts work. Almost every real one is three lines that kick off a batch.

Day 26

**50,000 row limit vs 10,000 DML limit — different ceilings** *(Day 26)*
Confused these in a quiz: 50,000 is the max rows a normal SOQL query can
return (what QueryLocator exists to bypass). 10,000 is the max records in
a single DML statement. Different limits, different purposes.

**Database.Stateful only when something must survive across chunks** *(Day 26)*
Instance variables reset to their initial value at the start of every batch
chunk — Salesforce creates a fresh object each time. Stateful keeps the same
object alive across all chunks. Needed for a running total or a list of
failures to report in finish(); skip it otherwise, since it costs overhead
to carry state through every transaction.

**No separate test class for TierChangeLogger (Queueable)** *(Day 26)*
Verified — not assumed — that AccountTriggerTest already updates an Account's
tier inside startTest/stopTest and asserts on the resulting Tier_Change_Log__c
record with real expected values (not just size()). That exercises
TierChangeLogger.execute() end to end. A dedicated test class would cover
the identical path twice.
*Flips when:* the Queueable gains logic an existing test doesn't exercise.