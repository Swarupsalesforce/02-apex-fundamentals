# PATTERN CHEAT SHEET
**Owner:** Soumya · **As of:** Day 8 (Week 2)
Rule: every pattern here is one you have *built yourself*. Nothing on this page is theory.

---

## PART 1 — LOOP + MEMORY PATTERNS (Week 1)

All six are the same machine: **walk a collection, keep something in a box, answer a question at the end.** What differs is what's in the box.

### 1. Flag pattern — "is it true for ALL of them?"
**Cue:** prime check · "are all…" · "is any…" · "does it contain a bad one"
```apex
Boolean isPrime = true;                 // optimistic
for (Integer i = 2; i < n; i++) {
    if (Math.mod(n, i) == 0) {
        isPrime = false;
        break;                          // one witness is enough
    }
}
if (isPrime) { ... }                    // verdict AFTER the loop
```
**Gotchas:** verdict goes *outside* the loop (else it prints every pass) · seed optimistic, flip on witness
**Principle:** *not-X is proven by one witness; X is proven by exhausting the search.*

### 2. Champion / best-so-far — "which one is biggest?"
**Cue:** largest · smallest · most recent · highest revenue · latest CloseDate
```apex
Integer champion = nums[0];             // seed from a REAL element
for (Integer n : nums) {
    if (n > champion) champion = n;
}
```
**Gotchas:** never seed from `0` — that invents an answer that isn't in the data · `>` vs `>=` decides who wins a tie → comment your ruling
**In Salesforce:** "each Account's most recent Opportunity" = this, with dates.

### 3. Two-state boxes — "second largest"
**Cue:** second-highest · runner-up · top two · previous value
```apex
Integer first  = ...;                   // seed from first TWO real elements
Integer second = ...;                   // swap them if out of order
for (...) {
    if (n > first)       { second = first; first = n; }   // TRANSFER before crowning
    else if (n > second) { second = n; }                  // middle values
}
```
**Gotchas:** the transfer line must come *first*, or you overwrite the old champion and lose it · `else if`, not a second `if` · duplicates need a ruling

### 4. Unique list / Set membership — "remove duplicates"
**Cue:** distinct · unique · dedupe · "already seen?"
```apex
Set<String> seen = new Set<String>();
for (String s : items) {
    if (!seen.contains(s)) { seen.add(s); uniqueList.add(s); }
}
// or simply: mySet.addAll(myList);   ← the bouncer does it for free
```
**Gotchas:** normalize *before* the bouncer (`toLowerCase().trim()`) or `Bob@x.com` and `bob@x.com` both get in
**Two-sum version:** check `contains(partner)` **then** `add(num)` — adding first lets a number match itself.

### 5. Counting map — "how many of each?"
**Cue:** count by · frequency · how many per · tally
```apex
Map<String, Integer> counts = new Map<String, Integer>();
for (String k : keys) {
    if (counts.containsKey(k)) counts.put(k, counts.get(k) + 1);
    else                       counts.put(k, 1);
}
```
**Gotchas:** needs `if/ELSE` — two different actions · key on the thing that **repeats** (a tier, a domain) not on something unique (an Id)

### 6. Compound heartbeat — `Map<Key, List<Value>>` — "group them"
**Cue:** group by · for each X, its Y · bucket by
```apex
Map<String, List<Contact>> byAccount = new Map<String, List<Contact>>();
for (Contact c : contacts) {
    if (!byAccount.containsKey(c.AccountId)) {
        byAccount.put(c.AccountId, new List<Contact>());   // create bucket ONCE
    }
    byAccount.get(c.AccountId).add(c);                     // add ALWAYS runs
}
```
**Gotchas:** only the *create* step is conditional; the `add` is unconditional — this is what makes it different from #5 · `get(key)` returns the real List, not a copy, so `.add()` on it works

---

## PART 2 — SOQL (Day 6)

| Need | Shape |
|---|---|
| Filter by variable | `WHERE Name = :myVar` |
| Filter by collection | `WHERE Id IN :myIdSet` |
| Child → parent field | `SELECT Account.Name FROM Contact` |
| Parent → children | `SELECT Id, (SELECT Name FROM Contacts) FROM Account` |
| Count / group | `SELECT Industry, COUNT(Id) c FROM Account GROUP BY Industry` → read with `.get('c')` |
| Safe single record | query into a **List**, `isEmpty()` check, then `list[0]` |

**Gotchas:** `Account a = [SELECT...]` throws when zero rows — always List + isEmpty · aggregate results are `AggregateResult`, not your object · you can only modify fields you **queried** · query only what you need

**Never:** SOQL inside a loop. Limit is **100 queries** per transaction.

---

## PART 3 — DML + TRIGGERS (Day 8)

### The timing rule — memorize this sentence
> **Setting a field on the record being saved → BEFORE.**
> **Touching other records → AFTER.**

Before = not written yet, so assignment rides along free, no DML.
After = already committed, `Trigger.new` fields are read-only, needs DML.

### 7. Bulk-safe before-trigger *(the safest trigger there is)*
```apex
trigger AccountTrigger on Account (before insert, before update) {
    for (Account acc : Trigger.new) {      // Trigger.new is a LIST — 1 or 200
        acc.Some_Field__c = computeFrom(acc);
    }
}
```
**Zero SOQL, zero DML inside.** This is SF-001.

### 8. Collect-then-DML
```apex
List<Task> toInsert = new List<Task>();
for (...) { toInsert.add(new Task(...)); }
if (!toInsert.isEmpty()) insert toInsert;   // ONE statement, after the loop
```

### 9. Change detection — `Trigger.oldMap`
```apex
for (Account acc : Trigger.new) {
    Account oldAcc = Trigger.oldMap.get(acc.Id);
    if (acc.AnnualRevenue != oldAcc.AnnualRevenue) { ... }
}
```
`oldMap` is a coat-check Salesforce hands you **pre-built**. Unavailable on insert.

### Trigger context quick table
| Variable | Holds | Available |
|---|---|---|
| `Trigger.new` | records being saved | insert, update |
| `Trigger.old` | previous versions | update, delete |
| `Trigger.newMap` / `oldMap` | `Map<Id, SObject>` | **not** in before-insert (no Ids yet) |
| `Trigger.isInsert` / `isBefore` | Boolean | always |

### DML gotchas
- **150 DML statements** per transaction → never DML in a loop
- `insert` needs no Id · `update`/`delete` need one
- Plain `insert list;` is **all-or-nothing**; `Database.insert(list, false)` saves the good ones
- Deletes go to the Recycle Bin (15 days)
- **One trigger per object** — order between multiple triggers is undefined
- Records in a List are the **same objects**, not copies — modifying via a loop modifies the original List

---

## PART 4 — THE TWO SILENT BUGS
Loud bugs get fixed in an hour. Silent bugs corrupt data for a month.

| Bug | What it does | Why it survives testing |
|---|---|---|
| **Reading a null** | assigns null happily; crashes *later*, on a different line | the line that breaks isn't the line that's wrong |
| **`Trigger.new[0]`** | sets the field on record 1, **silently skips 199** | works perfectly in a UI single-record test |

Reviewer's first question on any trigger: *does it loop `Trigger.new`?*

---

## PART 5 — METHOD (works on problems no cheat sheet covers)
1. **Paper and English first.** No code until the sentences are right.
2. **Restate the problem** in your own words.
3. **Predict the output** before running — a passing test you didn't predict taught you nothing.
4. **Smallest failing case.** Trace it by hand, one line at a time.
5. **Every ruling gets a comment.** Null, ties, empty input, duplicates.
6. **20 minutes stuck → ask.** 3 genuine attempts → assembly help.
7. **Casing audit before commit:** `Integer` `String` `List` `Map` `System.debug` capitalized · `insert` `update` `if` `while` `null` lowercase.

---

## NOT YET OWNED — arriving on schedule
Test data factory (Day 9) · handler pattern (Day 10) · collect-Ids-query-once-Map-lookup (Day 11) · rollups (Day 11) · static recursion guard (Day 11) · `addError()` validation (Day 12) · related-record automation (Day 13) · then partial success, upsert by external Id, wrapper classes, selector/service layers, custom metadata–driven logic, async selection, security enforcement, JSON, callouts, platform events, LWC patterns.

*Don't study this section. It arrives when it arrives.*
