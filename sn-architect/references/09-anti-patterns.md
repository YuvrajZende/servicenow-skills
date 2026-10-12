# ServiceNow Architecture Anti-Patterns & Traps (`09-anti-patterns.md`)

> Common architectural mistakes, why they fail, and verified correct patterns.

---

## 1. The `gr.getValue()` vs `gr.field_name` Object Pointer Trap

### Why It Fails
In ServiceNow JavaScript (Rhino / ECMAScript), accessing `gr.field_name` does **not** return a primitive string or number. It returns a reference to a Java `GlideElement` object bound to the active `GlideRecord` cursor.

When looping through records and storing `gr.field_name` into an array, every element in the array receives a pointer to the **exact same** mutable object. By the end of the loop, every item in your array reflects the value of the **last record** processed.

### Code Comparison

```javascript
// ❌ WRONG: Pushes GlideElement object references; all array elements end up referencing the cursor's LAST record!
var userList = [];
var gr = new GlideRecord('sys_user');
gr.addActiveQuery();
gr.setLimit(3);
gr.query();
while (gr.next()) {
    userList.push(gr.user_name); // Trapped! Pushing live GlideElement reference
}
// Tested on ServiceNow (Scripts - Background):
// gs.info(JSON.stringify(userList)); -> ["user_3", "user_3", "user_3"]
// Every entry holds a reference to the same GlideElement instance updated by gr.next()!

// ✅ RIGHT: Returns a primitive JavaScript string
var userList = [];
var gr = new GlideRecord('sys_user');
gr.addActiveQuery();
gr.setLimit(5);
gr.query();
while (gr.next()) {
    userList.push(gr.getValue('user_name')); // Safe! Returns string primitive
}
// Result: userList = ['user1', 'user2', 'user3', 'user4', 'user5']
```

### The Trap Beyond Arrays

Pushing into an array is only the most infamous symptom of retaining `GlideElement` references. Because `gr.field_name` returns a `GlideElement` object rather than a JavaScript primitive, reading it directly in assignments, comparisons, and logic causes subtle bugs:

- **Direct storage into variables or objects:** Assigning `var val = gr.field_name` (or setting an object property `obj.name = gr.field_name`) stores the `GlideElement` object reference, not the string value. In a loop, every stored property will point to the cursor's final record value once iteration completes.
- **Concatenation and coercion:** Concatenating `gr.field_name + ''` relies on implicit Java-to-JavaScript string conversion, which can behave inconsistently depending on whether the code runs in scoped scripts, global scripts, or server-to-client JSON serialization.
- **Empty & existence checks:** Rather than relying on implicit JavaScript truthiness (`if (gr.field_name)` or `if (!gr.field_name)`), use official platform APIs designed specifically for field existence checks: `gr.field_name.nil()`, `gs.nil(gr.field_name)`, or compare primitive values using `!gr.getValue('field_name')`.

**Rule:** When reading field values for logic, comparisons, array/object storage, or returns, always extract the primitive value via `gr.getValue('field_name')`. Reserve dot-access (`gr.field_name`) for assigning values (`gr.field_name = val`) and invoking GlideElement methods (`.nil()`, `.getDisplayValue()`, `.canRead()`).

---

## 2. `current.update()` in Before Business Rules

### Why It Fails
A Before Business Rule executes synchronously *before* the database write commits. Calling `current.update()` inside a Before Business Rule triggers a new, recursive update transaction on the same record. This spawns an infinite loop until the transaction engine hits recursion limits and terminates.

### Code Comparison

```javascript
// ❌ WRONG: Triggers recursive update loop
(function executeRule(current, previous /*null when async*/) {
    if (current.priority == 1) {
        current.urgency = 1;
        current.update(); // CRITICAL BUG: Recursive loop exhaustion!
    }
})(current, previous);

// ✅ RIGHT: Mutate the in-memory object directly; platform commits it automatically
(function executeRule(current, previous /*null when async*/) {
    if (current.priority == 1) {
        current.urgency = 1; // Correct! Automatically saved during normal commit
    }
})(current, previous);
```

---

## 3. Client-Side GlideRecord

### Why It Fails
Calling `new GlideRecord('table')` in a Client Script makes a synchronous HTTP request from the user's browser back to the ServiceNow database. This freezes the user's browser UI, degrades performance, and bypasses server-side validation layers.

### Code Comparison

```javascript
// ❌ WRONG: Synchronous client-side GlideRecord freezes the browser UI
function onChange(control, oldValue, newValue, isLoading) {
    if (isLoading || newValue === '') return;
    
    var gr = new GlideRecord('sys_user');
    gr.get(newValue); // FREEZES UI until network request completes
    g_form.setValue('email', gr.email);
}

// ✅ RIGHT: Asynchronous GlideAjax with callback function
function onChange(control, oldValue, newValue, isLoading) {
    if (isLoading || newValue === '') return;
    
    var ga = new GlideAjax('UserLookupUtils');
    ga.addParam('sysparm_name', 'getUserDetails');
    ga.addParam('sysparm_user_id', newValue);
    ga.getXMLAnswer(function(response) {
        var data = JSON.parse(response);
        if (data && data.email) {
            g_form.setValue('email', data.email);
        }
    });
}
```

---

## 4. The Empty ACL Trap

### Why It Fails
An ACL record created without roles, without conditions, and without a script is called an **Empty ACL**. Instead of denying access, an empty ACL evaluates to `true` for **any authenticated user**. If created on an administrative or sensitive table, it inadvertently grants read or edit access to all users across the enterprise.

### Pattern Comparison

```
❌ WRONG: Empty ACL Configuration
Table: x_acme_loan_request
Operation: read
Roles: [Empty]
Condition: [Empty]
Script: [Empty]
Result: EVERY logged-in user in the company can read all loan requests!

✅ RIGHT: Explicit Role-Gated ACL Configuration
Table: x_acme_loan_request
Operation: read
Roles: [x_acme_loan.fulfiller, x_acme_loan.admin]
Condition: [None or State-based]
Script: answer = true;
Result: Only authorized fulfillers and admins can read loan records.
```

---

## 5. Hardcoding Sys IDs

### Why It Fails
Hardcoding `sys_id` strings (e.g., `'a8f12b4097... '`) directly in scripts breaks when code is deployed between instances (Dev ➔ Test ➔ Prod). While metadata sys_ids may be preserved, data records (groups, categories, parent records) frequently have different sys_ids across environments.

### Pattern Comparison

```javascript
// ❌ WRONG: Hardcoded sys_id breaks across Dev / Test / Prod
if (current.assignment_group == 'd625fc6e4f3a8200b3c6... ') {
    // Logic breaks in production where group has a different sys_id
}

// ✅ RIGHT: Retrieve via System Property or query by immutable name
var tier2Group = gs.getProperty('x_acme_loan.tier2_assignment_group');
if (current.assignment_group == tier2Group) {
    // Clean, configurable, and environment-independent
}
```

---

## 6. The N+1 Nested GlideRecord Query Trap

### Why It Fails
Executing a second database query inside a `while (gr.next())` loop causes exponential database round-trips. If the outer query returns 1,000 records, the script executes 1,001 separate SQL queries, causing extreme database latency.

### Pattern Comparison

```javascript
// ❌ WRONG: N+1 queries executed in a loop
var incidentGr = new GlideRecord('incident');
incidentGr.addActiveQuery();
incidentGr.query();
while (incidentGr.next()) {
    var problemGr = new GlideRecord('problem');
    problemGr.addQuery('sys_id', incidentGr.problem_id);
    problemGr.query(); // 1,000 extra queries!
    if (problemGr.next()) {
        // process
    }
}

// ✅ RIGHT: Batch query or collect keys first using dot-walking or GlideAggregate
var incidentGr = new GlideRecord('incident');
incidentGr.addActiveQuery();
incidentGr.addNotNullQuery('problem_id');
incidentGr.query();

var problemIds = [];
while (incidentGr.next()) {
    problemIds.push(incidentGr.getValue('problem_id'));
}

var problemGr = new GlideRecord('problem');
problemGr.addQuery('sys_id', 'IN', problemIds.join(','));
problemGr.query(); // Exactly 1 query fetches all related records!
```

---

## 7. Naive `new Date()` Timezone Bugs

### Why It Fails
JavaScript's `new Date()` runs in the JVM/user session timezone, while ServiceNow stores all dates in UTC and displays them in the user's timezone. Mixing naive JS dates with Glide date fields produces off-by-hours records whose direction depends on the viewer's timezone.

```javascript
// ❌ WRONG: JVM-timezone date written into a UTC-backed field
var d = new Date(); // local JVM time, not UTC, not user time
current.u_expiry = d;

// ✅ RIGHT: GlideDateTime handles UTC storage and user display correctly
var gdt = new GlideDateTime();
current.u_expiry = gdt;
gdt.addDaysUTC(30); // explicit UTC arithmetic
```

---

## 8. `GlideRecord.setWorkflow(false)` Side Effects

### Why It Fails
`gr.setWorkflow(false)` silently suppresses ALL business rules, flows, notifications, and audit on the update. Used to "avoid recursion," it also skips validation, SLA engine updates, and notifications nobody knows they lost.

```javascript
// ❌ WRONG: silent suppression of every engine on this update
gr.assignment_group = tier2SysId;
gr.setWorkflow(false);
gr.update();

// ✅ RIGHT: scope the suppression narrowly and document it
gr.setWorkflow(false); // intentional: bulk backfill, BR recursion guard — reviewed
gr.autoSysFields(false); // also: preserve sys_updated_by/on semantics decision
gr.update();
```
**Rule:** every `setWorkflow(false)` must carry a comment naming what is being skipped and why; most uses should be replaced with a targeted BR guard condition instead.

---

## 9. Querying `sys_audit` Directly

### Why It Fails
`sys_audit` is a specialized, high-volume internal table. Direct GlideRecord queries against it are slow, unreliable for history reconstruction, and can be excluded from audit itself. History reconstruction belongs in the audited table's own records or dedicated APIs — not bespoke `sys_audit` scans (see `01-data-model.md` §6).

---

## 10. Trusting Client-Side Security

### Why It Fails
Client scripts, `g_form.setReadonly`, and hidden fields are *presentation*, not security. Any permission decision enforced only in the browser is bypassed by any REST call or crafted request. Server ACLs (`05-security-design.md`) are the only enforcement layer; client logic is UX.

---

## Authoritative Sources
- [ServiceNow Developer Guide: GlideRecord API (Scoped)](https://developer.servicenow.com/dev.do#!/reference/api/latest/server/no-namespace/c_GlideRecordScopedAPI)
- [ServiceNow Developer Guide: GlideElement API (Scoped)](https://developer.servicenow.com/dev.do#!/reference/api/latest/server/no-namespace/c_GlideElementScopedAPI)
- [ServiceNow Documentation: Server-side Scripting Best Practices](https://docs.servicenow.com/bundle/xanadu-application-development/page/script/server-scripting/concept/c_ServerSideScripting.html)
