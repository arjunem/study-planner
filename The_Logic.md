# Study Planner — Schedule Logic

## How the Schedule is Built

The planner distributes course hours day by day from the start date, filling each day up to its available hours limit before moving to the next day.

- **Weekdays** (Mon–Fri) use the **Weekday Hours** setting
- **Weekends** (Sat–Sun) use the **Weekend Hours** setting
- Modules are placed in order; a module that doesn't finish in one day carries over to the next

---

## Example

| Setting | Value |
|---|---|
| Total course hours | 24h |
| Weekday hours/day | 2h |
| Weekend hours/day | 4h |
| Start date | Thu 23 Apr 2026 |

### Day-by-day breakdown

| # | Day | Date | Type | Hours | Balance |
|---|---|---|---|---|---|
| 1 | Thursday | 23 Apr | Weekday | 2h | 22h |
| 2 | Friday | 24 Apr | Weekday | 2h | 20h |
| 3 | Saturday | 25 Apr | Weekend | 4h | 16h |
| 4 | Sunday | 26 Apr | Weekend | 4h | 12h |
| 5 | Monday | 27 Apr | Weekday | 2h | 10h |
| 6 | Tuesday | 28 Apr | Weekday | 2h | 8h |
| 7 | Wednesday | 29 Apr | Weekday | 2h | 6h |
| 8 | Thursday | 30 Apr | Weekday | 2h | 4h |
| 9 | Friday | 1 May | Weekday | 2h | 2h |
| 10 | Saturday | 2 May | Weekend | 2h | 0h |

**Total: 10 study days**

> Saturday 2 May only needs 2h even though 4h are available — the course ends when the balance hits zero.

---

## Module Distribution

### Each module has its own hours

Modules are not equal in size. A plan might look like:

| Module | Hours |
|---|---|
| M1 — Intro | 1h |
| M2 — Core Concepts | 3h |
| M3 — Deep Dive | 5h |
| M4 — Practice | 2h |

The planner places them in order, filling each day to its limit before moving on.

---

### Case 1 — One day, multiple modules

When a module finishes before the day's hours are used up, the next module starts on the same day.

**Example:** Weekday = 4h, M1 = 1h, M2 = 3h

```
Thursday (4h available)
  ├── M1 — Intro        1h  [done]   rem = 3h
  └── M2 — Core Concepts 3h  [done]   rem = 0h
```

Both modules land on the same day. The day is fully consumed.

---

### Case 2 — One module, multiple days

When a module's hours exceed the day's limit, it is split. The remainder carries over to the next day. The module appears on both days in the calendar.

**Example:** Weekday = 2h, M3 = 5h

```
Monday    (2h available)  →  M3 starts    carry = 3h remaining
Tuesday   (2h available)  →  M3 continues carry = 1h remaining
Wednesday (2h available)  →  M3 finishes  carry = 0h, 1h spare
                              M4 starts using the spare 1h
```

---

### Case 3 — Mixed (split + overflow on the same day)

The two cases combine naturally. A day can finish a carried-over module **and** start one or more new modules if hours allow.

**Example:** Weekday = 4h, carry = 1h left on M3, M4 = 2h, M5 = 3h

```
Thursday (4h available)
  ├── M3 — finish carry   1h  [done]   rem = 3h
  ├── M4 — Practice       2h  [done]   rem = 1h
  └── M5 — Deep Dive      1h  [split]  rem = 0h  →  carry = 2h to Friday
```

---

## Status Rules

### Skip
A skipped day's hours are **carried forward** and appended to the next day that has remaining capacity within its daily limit.

**Example:** Wednesday (2h) is skipped.
- The remaining balance (6h → still 6h) must be absorbed by later days
- The next Saturday has 4h available but only 2h were originally scheduled
- The spare 2h capacity on that Saturday absorbs the skipped 2h → Saturday now uses its full 4h slot
- Course end date shifts by 0 days if absorbed, or by 1+ days if no capacity exists on the same day

---

## Key Invariants

1. `mi` (module index) advances **only when a module is fully consumed** — never mid-module
2. `carry` tracks **hours still remaining** on the current in-progress module
3. The `safe` counter caps the loop at 700 iterations to prevent infinite loops
4. Total scheduled hours always equals or exceeds total course hours (surplus shows as "Hours buffer")
