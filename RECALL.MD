# RECALL CARD
One screen. Read before every problem. **This file does not grow** — PATTERNS.md is for detail.

---

## 1. THE THREE QUESTIONS — ask in this order, every time

**Q1. Which object's *save* should make this code run?**
→ that's where the trigger goes. Not the object named first in the sentence.
*Stamp a Case from its Account → Case fires. Count Contacts on an Account → Contact fires.*

**Q2. Am I writing to the record being saved, or to a different record?**
→ same record = **BEFORE**, just assign, no DML.
→ different record = **AFTER**, needs DML.

**Q3. Am I looking up ONE value, or gathering MANY?**
→ one → `Map<Id, Parent>` · lookup
→ many → group or aggregate · rollup

Answer those three and the design is decided. Everything else is syntax.

---

## 2. CUE WORDS → PATTERN

| The requirement says… | Reach for |
|---|---|
| "are all…" / "is any…" | flag + break, verdict after the loop |
| "largest" / "latest" / "most recent" | champion — seed from a real element |
| "unique" / "dedupe" / "already seen" | Set |
| "how many of each" / "count by" | counting map, `if/else` |
| "group by" / "for each X, its Y" | `Map<K, List<V>>` — create once, add always |
| "set the child's field from its parent's…" | collect Ids → query once → map lookup |
| "total / count / latest **on** the parent" | rollup — seed zeros first |
| "don't let" / "prevent" / "must have" | `addError()` — **but check validation rule first** |
| "only when X changes" | `Trigger.oldMap` |

---

## 3. SOQL — FIVE FORKS

| I want back… | Shape |
|---|---|
| parents, with children attached | subquery: `(SELECT … FROM Contacts)` |
| children, filtered by the parent | dot notation: `WHERE Account.Field__c = …` |
| parents that have **no** children | anti-join: `WHERE Id NOT IN (SELECT AccountId FROM Contact)` |
| a **number** per parent | `GROUP BY` + aggregate |
| to filter on that number | `HAVING COUNT(Id) > 3` |

**Semi-join inner query names the LOOKUP field** (`AccountId`), never `Id`.
**Clause order:** `SELECT → FROM → WHERE → GROUP BY → HAVING → ORDER BY → LIMIT`
**No LEFT JOIN exists.** Totals-including-zero = aggregate into a Map, default to 0 in Apex.

---

## 4. THE REVIEWER'S FIRST LOOK — before reading any logic

1. Any `[SELECT` inside a loop? → dies at **101**
2. Any `insert/update/delete` inside a loop? → dies at **151**
3. Does it loop `Trigger.new`, or assume one record?
4."trace it through — insert, update, delete, reparent, undelete, empty result, null aggregate, 200 records." Eight cases, and they catch the bugs that compile.

SOQL fails **first**. Check for the query before the DML.

---

## 5. MY SIX REPEAT ERRORS — pre-commit audit, 90 seconds

- [ ] **Key:** same field on `containsKey`, `put`, and `get`? Is it the thing that *repeats*, not the thing that's unique?
- [ ] **One or many:** is this variable a List? Then it has no `.Id` — it needs `[0]`.
- [ ] **Verdict placement:** is the answer being decided *inside* a loop that should decide it after?
- [ ] **Messages:** does every assertion message match the value it asserts?
- [ ] **Loops:** any query or DML inside one?
- [ ] **Casing:** `Integer String List Map System.debug` capital · `insert update if while null` lower

---

## 6. TWO SENTENCES WORTH KNOWING COLD

> **A loud bug gets fixed in an hour. A silent bug corrupts data for a month.**
> Reading a null doesn't throw — the crash comes later, on a different line. `Trigger.new[0]` doesn't throw — it silently does 1/200th of the job.

> **The best code is the code you didn't write.**
> Master-detail → roll-up summary. Fields on the record itself → validation rule. Apex only when it needs *other records'* data, or fires on delete.