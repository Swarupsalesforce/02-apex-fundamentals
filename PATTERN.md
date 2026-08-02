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
**⚠️ A Map has no order.** `keySet()` returns keys in no promised sequence. So "the **first** X that…" must loop the original **List**, never the map:
The map answers *"how many?"*; only the list knows *"which came first?"* Looping the keySet gives the right answer on your test word and a silent wrong answer later.
Seed the verdict (`result = 'None'`) **before** the loop; only a discovery overwrites it.

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

### Interview shapes (Days 9–10 drills)

| Need | Shape |
|---|---|
| Parents that HAVE children | `WHERE Id IN (SELECT AccountId FROM Contact)` — semi-join |
| Parents with NO children | `WHERE Id NOT IN (SELECT AccountId FROM Contact)` — anti-join |
| Top N children per parent | `(SELECT Id FROM Cases ORDER BY CreatedDate DESC LIMIT 5)` — subquery takes its **own** ORDER BY + LIMIT |
| Total per parent, incl. zeros | **impossible in one query — SOQL has no LEFT JOIN.** Aggregate → `Map<Id, Decimal>` → `containsKey ? get : 0` in Apex |

**The fork:** *records* per parent → **subquery**. *A number* per parent → **GROUP BY**.
**Semi-join trap:** the inner SELECT names the **lookup field** (`AccountId`), never `Id` — comparing Contact Ids to Account Ids matches nothing and throws nothing.
**Clause order is fixed:** `SELECT → FROM → WHERE → ORDER BY → LIMIT`. Swapping the last two is a compile error, and it's a deliberate interview trap.
**You CAN filter on a field you didn't SELECT.** The restriction is that you can only *modify* fields you queried.

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

---

## PART 3.5 — TEST CLASSES (Day 9)

### 10. Arrange → Act → Assert
**Cue:** "I need a test for…" — always this shape, every time.
```apex
@isTest
private class AccountTriggerTest {
    @isTest
    static void tierIsGoldAtSixMillion() {
        // ARRANGE — build data (setup DML goes HERE, outside the brackets)
        Account acc = TestDataFactory.createAccount('Test Corp', 6000000);

        // ACT — only the operation under test sits inside
        Test.startTest();
        insert acc;
        Test.stopTest();

        // ASSERT — RE-QUERY first, then assert
        Account result = [SELECT Customer_Tier__c FROM Account WHERE Id = :acc.Id];
        Assert.areEqual('Gold', result.Customer_Tier__c, 'Revenue 6M should be Gold');
    }
}
```
**`Assert.areEqual(expected, actual, message)` — expected FIRST.** Backwards order produces a failure message that lies.

### 11. Test data factory
```apex
@isTest
public class TestDataFactory {                          // public — tests call it
    public static Account createAccount(String name, Decimal revenue) {
        return new Account(Name = name, AnnualRevenue = revenue);   // RETURN, don't insert
    }
    public static List<Account> createAccounts(Integer count, Decimal revenueStep) {
        List<Account> accs = new List<Account>();
        for (Integer i = 1; i <= count; i++) {
            accs.add(new Account(Name = 'Test Acct ' + i, AnnualRevenue = revenueStep * i));
        }
        return accs;
    }
}
```
**Why:** two new required fields on Account = one edit instead of forty.
**Return, don't insert** — the test needs to control *when* the DML fires (it's often the thing being tested).
**Parameter names must match reality:** `revenueStep`, not `revenue`, when the body multiplies.

### Reading a method signature
`public static Account createAccount(String name, Decimal revenue)`
| Part | Question |
|---|---|
| `public` / `private` | who may call it |
| `static` | callable off the class name, no instance needed |
| `Account` | what it hands back (`void` = nothing) |
| `(...)` | what the caller must supply |

### The six tests every trigger needs
1. single record, happy path · 2. every branch · 3. **exact boundary** (proves `>=` vs `>`) · 4. **bulk 200** · 5. the update path · 6. null / sad path

### Test gotchas
- **RE-QUERY after DML.** Your in-memory variable doesn't know what the trigger did. `Expected X, Actual null` → suspect a stale record *before* suspecting the code.
- **No assertion = not a test.** It's a coverage generator that will pass while the logic is wrong.
- **`Test.startTest()` RESETS governor limits** → setup DML must go before it, or scaffolding eats your budget.
- **Tests can't see org data — for repeatability, not limits.** Same result in every org, forever. Everything created is rolled back.
- **Never `SeeAllData=true`.** Never hardcode Ids.
- **75% is a floor, not a goal.** 80% with sharp assertions beats 100% with none. Test boundaries and sad paths; trust the middles.
- **Insert a List even when one record would do** — any multi-record insert catches `Trigger.new[0]`.
- **`Assert`, not `System.assertEquals`** (both work; `Assert` is current).
- Empty string `''` in a text field is **stored as null** — assign `null` and mean it.
- In tests, `Account a = [SELECT...]` is *preferred* — you want it to fail loudly. Opposite of production. 

### Casing offenders (Day 9's audit)
`Update` ×3 → `update` · `Public static` → `public static`

---

## PART 3.6 — TRIGGER HANDLER PATTERN (Day 10)

### 12. Handler + context routing
**Cue:** any trigger with more than ~5 lines, or a second requirement on the same object
> **The trigger decides *when*. The handler decides *what*.**

```apex
// AccountTrigger.trigger — ONE line, forever. Nobody edits this file again.
trigger AccountTrigger on Account (before insert, before update) {
    AccountTriggerHandler.run();
}
```
```apex
public class AccountTriggerHandler {          // public — the trigger calls it

    public static void run() {                 // the ONLY method touching Trigger.*
        if (Trigger.isBefore && Trigger.isInsert) assignTiers(Trigger.new);
        if (Trigger.isBefore && Trigger.isUpdate) assignTiers(Trigger.new);
    }

    public static void assignTiers(List<Account> accounts) {   // pure logic
        for (Account acc : accounts) { ... }   // no Trigger refs, no DML, no SOQL
    }
}
```

**Why it exists:**
- trigger logic can't be reused — a Batch job can't fire a trigger, but it *can* call `assignTiers(scope)`
- a trigger can only be tested through DML; a class method is called directly → **fast unit test, no limits consumed**
- one-trigger-per-object means one file for *everything* → it must be a switchboard, not a workshop
- new requirement = new handler method; the trigger never changes

**Rules:**
- logic methods take **parameters** (`assignTiers(Trigger.new)`), never reach for `Trigger.new` internally — otherwise only a trigger can call them and you've gained nothing
- the `(before insert, before update)` declaration stays on the **trigger** — Salesforce's contract, the handler can't change it
- `run()` looks redundant when both branches do the same thing. It won't by Day 13. It's **documentation that executes.**
- **refactor with tests running after every step** — a failure then names the step that broke it

### The testability dividend
```apex
@isTest
static void assignTiersSetsGoldAtSixMillion() {
    List<Account> accs = new List<Account>{ new Account(Name='A', AnnualRevenue=6000000) };
    AccountTriggerHandler.assignTiers(accs);        // no insert, no re-query, no limits
    Assert.areEqual('Gold', accs[0].Customer_Tier__c, 'Revenue 6M should be Gold');
}
```
Keep **both** kinds: DML tests prove the **wiring**, direct-call tests prove the **logic**.

### static vs instance — the carry-now rule
**static** when the method just does a job (handlers, factories, utilities) → `ClassName.method(args)`
**instance** when the object must remember data between calls → `new ClassName()` first
*(Week 3 makes this formal.)*

### Refactoring gotchas learned the hard way
- **comments are the first casualty of moving code** — the null ruling didn't survive the move; re-add rulings after any refactor
- a chain of `else if` with a final *condition* (not a bare `else`) leaves a hole → on **update** an unmatched record keeps its **old value**, silently stale. Bare `else` or explicit assignment.
- assign `null`, not `''` — the DB stores them identically, so `''` makes a direct-call test assert `''` while a DML test asserts `isNull`: **two tests telling different stories about one ruling**

