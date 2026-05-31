# Date, Time & Time Zones

## The most error-prone corner of the BCL

Date/time handling causes more subtle production bugs than almost anything else: ambiguous `DateTime.Kind`, time-zone math, daylight-saving transitions, untestable `DateTime.Now`. This file gives you the decision rules: which type to use when, how to store time correctly, and how to make time testable.

---

## The types at a glance

| Type | Represents | Use for |
|---|---|---|
| `DateTimeOffset` | A point in time **with UTC offset** | **Timestamps** — the default for "when did this happen" |
| `DateTime` | A date+time with a `Kind` (Utc/Local/Unspecified) | Legacy; use carefully, prefer UTC |
| `TimeSpan` | A **duration** / elapsed time | Intervals, timeouts, differences |
| `DateOnly` (.NET 6+) | A calendar **date**, no time | Birthdays, due dates, schedules |
| `TimeOnly` (.NET 6+) | A **time of day**, no date | Opening hours, alarms |
| `TimeZoneInfo` | A time zone (with DST rules) | Converting between zones |
| `TimeProvider` (.NET 8+) | An abstraction over "now" | **Testable** time |

**Default recommendation**: use **`DateTimeOffset`** for timestamps, **`DateOnly`/`TimeOnly`** when you genuinely have only a date or a time, **`TimeSpan`** for durations, and inject **`TimeProvider`** instead of calling `DateTime.Now` directly.

---

## `DateTime` and the `Kind` trap

A `DateTime` carries a `Kind` flag — `Utc`, `Local`, or `Unspecified` — but it's easy to lose track of, and arithmetic/serialization can silently misinterpret it:

```csharp
DateTime.Now;          // Kind = Local   (the machine's local time)
DateTime.UtcNow;       // Kind = Utc      (UTC)
new DateTime(2026, 5, 22);   // Kind = Unspecified  ← dangerous: is it UTC? local? unknown!
```

The danger: an `Unspecified` `DateTime` read from a database or parsed from a string has no zone information, so `.ToLocalTime()`/`.ToUniversalTime()` guess, and comparisons across machines go wrong. **This ambiguity is why `DateTimeOffset` exists** — it always carries the offset, removing the guesswork.

```csharp
// ✗ — what zone is this? Comparing/converting it is a coin flip.
DateTime parsed = DateTime.Parse("2026-05-22T14:30:00");   // Unspecified

// ✓ — unambiguous instant
DateTimeOffset ts = DateTimeOffset.Parse("2026-05-22T14:30:00Z");  // explicit UTC
```

---

## `DateTimeOffset` — the timestamp default

`DateTimeOffset` is a point in time plus its offset from UTC. It's unambiguous and the right choice for "when did X happen":

```csharp
DateTimeOffset now = DateTimeOffset.UtcNow;       // recommended for "now"
DateTimeOffset local = DateTimeOffset.Now;         // local time + the machine's offset

DateTimeOffset created = order.CreatedAt;
TimeSpan age = DateTimeOffset.UtcNow - created;    // correct regardless of zones

// Compare two events from different zones — DateTimeOffset compares the instant correctly
bool aFirst = eventA < eventB;
```

It does **not** store the *named* time zone (only the offset at that moment), so it can't tell you "Eastern Time" — for that you need `TimeZoneInfo`. But for recording and comparing instants, it's ideal.

---

## Storing time correctly

The rule: **store UTC, convert to local only for display.**

```csharp
// Store:
order.CreatedAt = DateTimeOffset.UtcNow;                       // persist UTC instant
string forDb = order.CreatedAt.ToString("O", CultureInfo.InvariantCulture); // ISO 8601 round-trip

// Read back (round-trips exactly):
var back = DateTimeOffset.Parse(forDb, CultureInfo.InvariantCulture, DateTimeStyles.RoundtripKind);

// Display to a user in their zone:
TimeZoneInfo userZone = TimeZoneInfo.FindSystemTimeZoneById("America/New_York");
DateTimeOffset userLocal = TimeZoneInfo.ConvertTime(order.CreatedAt, userZone);
```

- Persist as **ISO 8601 ("O")** with **`InvariantCulture`** — sortable, unambiguous, locale-independent (CSharpBook Ch13 §08).
- Databases: prefer columns that store the instant/offset (`timestamptz` in Postgres, `datetimeoffset` in SQL Server).
- Convert to a user's zone **only at the presentation layer**.

---

## `TimeZoneInfo` and IANA IDs

```csharp
TimeZoneInfo.GetSystemTimeZones();                           // all zones on this OS
var tz = TimeZoneInfo.FindSystemTimeZoneById("Europe/Istanbul");  // IANA id
DateTimeOffset converted = TimeZoneInfo.ConvertTime(utc, tz);
bool isDst = tz.IsDaylightSavingTime(someInstant);
```

Modern .NET understands **IANA time-zone IDs** (`America/New_York`, `Europe/London`) on all platforms (it maps to Windows IDs automatically on Windows since .NET 6). Prefer IANA IDs for portability.

### Daylight Saving Time pitfalls

DST transitions create **invalid** times (clocks skip forward — e.g., 2:30 AM may not exist) and **ambiguous** times (clocks fall back — 1:30 AM happens twice):

```csharp
tz.IsInvalidTime(dt);      // true if the wall-clock time doesn't exist (spring forward)
tz.IsAmbiguousTime(dt);    // true if it occurs twice (fall back)
```

This is exactly why you **store UTC** — UTC has no DST, no gaps, no ambiguity. Do all storage and math in UTC; apply zone conversion only for display, and handle invalid/ambiguous cases there.

---

## `TimeSpan` — durations

```csharp
TimeSpan timeout = TimeSpan.FromSeconds(30);
TimeSpan elapsed = endTime - startTime;
TimeSpan total = TimeSpan.FromHours(1) + TimeSpan.FromMinutes(30);
double ms = elapsed.TotalMilliseconds;

await Task.Delay(TimeSpan.FromSeconds(5));      // APIs increasingly take TimeSpan over int ms
```

`TimeSpan` is a duration (can be negative), independent of any calendar. Use it for timeouts, intervals, and time differences. Modern APIs accept `TimeSpan` (clearer than "milliseconds as int").

---

## `DateOnly` and `TimeOnly` (.NET 6+)

When you have **only** a date or **only** a time, don't abuse `DateTime` (with a bogus time/date component):

```csharp
DateOnly dob = new(1990, 3, 15);                 // a birthday — no time, no zone
DateOnly today = DateOnly.FromDateTime(DateTime.UtcNow);
int age = today.Year - dob.Year - (today < dob.AddYears(today.Year - dob.Year) ? 1 : 0);

TimeOnly opening = new(9, 0);                     // 09:00 — store hours, not a fake date
TimeOnly closing = new(17, 30);
bool isOpen = TimeOnly.FromDateTime(DateTime.Now).IsBetween(opening, closing);
```

These avoid the classic bug of storing a birthday as a `DateTime` and then having time-zone conversion shift it to the previous day.

---

## `TimeProvider` — making time testable (.NET 8+)

`DateTime.Now`/`DateTimeOffset.UtcNow` are **untestable** — you can't control them in a unit test. Inject **`TimeProvider`** instead:

```csharp
public class TokenService(TimeProvider clock) {
    public bool IsExpired(Token t) => clock.GetUtcNow() > t.Expiry;
}

// Production: TimeProvider.System
var svc = new TokenService(TimeProvider.System);

// Tests: FakeTimeProvider (Microsoft.Extensions.TimeProvider.Testing)
var fake = new FakeTimeProvider();
fake.SetUtcNow(new DateTimeOffset(2030, 1, 1, 0, 0, 0, TimeSpan.Zero));
var svc2 = new TokenService(fake);
Assert.True(svc2.IsExpired(expiredToken));
fake.Advance(TimeSpan.FromHours(1));   // move time forward deterministically
```

`TimeProvider` also abstracts timers (`CreateTimer`) and `Task.Delay`, so time-dependent async code becomes testable too. This is the modern replacement for static-clock mocking (CSharpBook Ch16 §03).

---

## Common gotchas

### `DateTime.Now` vs `UtcNow`

`Now` is local (zone-dependent, DST-affected, machine-dependent). Use **`UtcNow`** for timestamps and logic; `Now` only for local display.

### `Unspecified` Kind ambiguity

A `DateTime` without a clear `Kind` (from parsing/DB) leads to wrong conversions. Prefer `DateTimeOffset`; if using `DateTime`, keep everything UTC explicitly.

### Storing local time

Persisting local time breaks across zones and DST. Store UTC (ISO 8601 + invariant culture); convert for display only.

### Birthday/date stored as `DateTime`

Time-zone conversion can shift the day. Use `DateOnly`.

### `DateTime.Now` in business logic

Untestable. Inject `TimeProvider` and use `FakeTimeProvider` in tests.

### Naive DST math

Adding hours to a local `DateTime` across a DST boundary gives wrong results. Do math in UTC, convert at the edges.

---

## Summary

- Default to **`DateTimeOffset`** for timestamps (unambiguous — carries the offset), **`TimeSpan`** for durations, **`DateOnly`/`TimeOnly`** for date-only/time-only values.
- **`DateTime`'s `Kind`** (esp. `Unspecified`) is a trap; if you use `DateTime`, keep it UTC.
- **Store UTC** (ISO 8601 "O" + `InvariantCulture`); convert to a user's zone via **`TimeZoneInfo`** (IANA IDs) **only for display**. UTC has no DST gaps/ambiguity.
- Use **`UtcNow`**, not `Now`, for logic/timestamps.
- Inject **`TimeProvider`** (with `FakeTimeProvider` in tests) instead of calling `DateTime.Now` — the modern, testable way to handle time.

→ Next: [07-Globalization.md](07-Globalization.md)
