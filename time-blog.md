# Time, TimeTZ, Timestamp, and TimestampTZ in PostgreSQL

At Cockroach Labs, we've spent a lot of time get our SQL semantics to match
PostgreSQL as much as possible - all so that you can use the awesome PostgreSQL
language with a powerful, distributed backend that CockroachDB can provide!
Getting this right involves getting the
[Time](https://www.cockroachlabs.com/docs/stable/time.html),
[TimeTZ](https://www.cockroachlabs.com/docs/stable/time.html),
[Timestamp](https://www.cockroachlabs.com/docs/stable/timestamp.html) and
[TimestampTZ](https://www.cockroachlabs.com/docs/stable/timestamp.html)
[types](https://www.postgresql.org/docs/current/datatype-datetime.html)
right - which has proven quite the quest! Numerous bugs have been filed and
fixed - and with it, our mastery of these types in PostgreSQL (and the Go time
library) has increased!

In this blog post, we'll explore some intricacies we've found with these data
types in PostgreSQL, and look at how we reproduce some of these features using
Go and its ecosystem. We'll share our recommendations for using these types
along the way.

Are you excited to learn about time in the PostgreSQL world? Join us as a
companion in our TARDIS (**T**ime **a**nd **R**elated **D**ata **T**ypes **I**n
**S**QL) and hop into an adventure into the world of time. Allons-y!

## Session Time Zones
When you open a connection to PostgreSQL (or CockroachDB), you will have a
session between you (the client) and the database (the server). The session will
contain a time zone, which you can set with the `SET TIME ZONE` command for
using [PostgreSQL](https://www.postgresql.org/docs/current/sql-set.html).

By default, the PostgreSQL shell (psql) will try to connect and set the session
to the local time zone as defined by your `timezone` setting in
`postgresql.conf`, whereas CockroachDB will default to UTC. We can observe this
with `CURRENT_TIMESTAMP`, which returns the current timestamp in the default
session time zone:

```
otan=# -- Using a PG shell, we can see Australia/Sydney time by default.
otan=# show time zone;
     TimeZone
------------------
 Australia/Sydney
(1 row)

otan=# select CURRENT_TIMESTAMP;
       current_timestamp
-------------------------------
 2023-03-16 16:35:20.703644+11
(1 row)
```

You can specify your time zone to be a location (which is daylight savings
aware), or a UTC offset. This will change the `CURRENT_TIMESTAMP` to be in the
time zone your session is set to:

```
otan=# SET TIME ZONE 'Australia/Sydney';
SET
otan=# SELECT CURRENT_TIMESTAMP;
      current_timestamp
------------------------------
 2023-03-16 16:37:00.21676+11
(1 row)

otan=# SET TIME ZONE '-11';
SET
otan=# select CURRENT_TIMESTAMP;
       current_timestamp
-------------------------------
 2023-03-15 18:37:06.880169-11
(1 row)
```

Note your ORM or driver may have different default behaviour than the psql shell
or the CockroachDB shell but should allow you to set a default time zone.

Setting your session time zone will affect some time operations in some fun and
exciting ways (which is not necessarily what you want as a developer!) which
we'll explore soon.

## POSIX Time Offsets
In the above examples, we have used `SET TIME ZONE` with an integer offset
(`-11`) or a location (`Australia/Sydney`). However, PostgreSQL also supports
the syntax for `SET TIME ZONE` to have `GMT` or `UTC` at the front, e.g. `SET
TIME ZONE 'UTC+3'`. Let's see how it behaves:

```
otan=# SET TIME ZONE 'UTC+3';
SET
otan=# SELECT CURRENT_TIMESTAMP;
       current_timestamp
-------------------------------
 2023-03-16 02:38:23.467396-03
(1 row)
```

Hold on a second, why does it have a time zone of `-3` when setting the time
zone to `UTC+3`?

It turns out that this is because having UTC or GMT in front uses the [POSIX
definition for time
zones](https://www.postgresql.org/docs/current/datatype-datetime.html#DATATYPE-TIMEZONES),
which defines zones to be hours west of the GMT line. In essence, the sign is
reversed from the time zone structure we know and love, where positive integers
represent that the timestamp is east of the GMT line (also known as ISO8601
standard).

Note that the POSIX standard also applies if you just add colon separators to
the offset in `SET TIME ZONE`:

```
otan=# SET TIME ZONE '+3:00';
SET
otan=# SELECT CURRENT_TIMESTAMP;
       current_timestamp
-------------------------------
 2023-03-16 02:40:30.983731-03
(1 row)
```

The takeaway here is that integers are treated as ISO8601 format, locations do
what you expect and anything else is POSIX standard.

Thinking this is a little weird? We've got another surprise with this coming
later with `AT TIME ZONE`!

## TIMESTAMP and TIMESTAMPTZ

The TIMESTAMP (also known as `TIMESTAMP WITHOUT TIME ZONE`) and TIMESTAMPTZ
(also known as `TIMESTAMP WITH TIME ZONE`) types stored as a 64-bit integer as a
microsecond offset since 1970-01-01 in CockroachDB and as a 64-bit integer
microsecond offset since 2000-01-01 in PostgreSQL by default.

As both data types are stored using only 64-bit integers, it is important to
note that neither **store** any time zone information. 

So where does TIMESTAMPTZ get the "time zone" display from? The session time
zone! As a corollary, that means storing the TIMESTAMPTZ and fetching the
results from a different session time zone results in a different printed
result, with the same equivalent UTC offset underneath.

Let's look storing a TIMESTAMP / TIMESTAMPTZ at `Australia/Sydney` and fetching
it from the `-3` time zone:

```
otan=# SET TIME ZONE 'Australia/Sydney';
SET
otan=# CREATE TABLE time_comparison(t TIMESTAMP, ttz TIMESTAMPTZ);
CREATE TABLE
otan=# INSERT INTO time_comparison values (CURRENT_TIMESTAMP, CURRENT_TIMESTAMP);
INSERT 0 1
otan=# SELECT t, ttz FROM time_comparison;
             t              |              ttz
----------------------------+-------------------------------
 2023-03-16 16:44:53.614697 | 2023-03-16 16:44:53.614697+11
(1 row)

otan=# SET TIME ZONE '-3';
SET
otan=# SELECT t, ttz FROM time_comparison;
             t              |              ttz
----------------------------+-------------------------------
 2023-03-16 16:44:53.614697 | 2023-03-16 02:44:53.614697-03
(1 row)
```
The time we've inserted these entries as` 2023-03-16 16:44:53.614697` with the
session time zone being `+11` (from `Australia/Sydney`). The TIMESTAMP column
does not "store" the time zone, so drops that information.

When we switch time zones to `UTC-3`, only the `ttz` column will change its
result by transforming the time into the `-3` offset, which is 14 hours behind,
hence now displaying `2023-03-16 02:44:53.614697-03`.

If you're still confused, don't worry — it's confusing to us too. Here's how
we think about it:

* **TIMESTAMP is just a date and time - nothing else**. It does not store or
  behave like differently depending on the session time zone. It may help to
  conceptualise it as "behaving like a UTC timestamp".
* **TIMESTAMPTZ stores date and time, but it displays timestamps and performs
  operations based on the session time zone**. You've seen the "displays
  timestamp" bit already — we'll get to the "performs operations" component
  later.

### Parsing
Let's parse the opening ceremony of the [Sydney
Olympics](https://en.wikipedia.org/wiki/2000_Summer_Olympics) in PostgreSQL with
using a Californian time zone:

```
otan=# SET TIME ZONE 'america/los_angeles';
SET
otan=# SELECT '2000-09-15 19:00'::TIMESTAMP, '2000-09-15 19:00'::TIMESTAMPTZ;
      timestamp      |      timestamptz
---------------------+------------------------
 2000-09-15 19:00:00 | 2000-09-15 19:00:00-07
(1 row)
```

So parsing a timestamp string without any time zone information in the string
will automatically default to the current time zone for TIMESTAMPTZ.

Let's append Sydney's time zone in September (+11:00) into the string before we
cast:
```
otan=# SET TIME ZONE 'america/los_angeles';
SET
otan=# SELECT '2000-09-15 19:00+11:00'::TIMESTAMP, '2000-09-15 19:00+11:00'::TIMESTAMPTZ;
      timestamp      |      timestamptz
---------------------+------------------------
 2000-09-15 19:00:00 | 2000-09-15 01:00:00-07
(1 row)
```

As we see above:
* For TIMESTAMPs, we've got `19:00` — same as the input but we've ignored the time
zone offset. This can be significant for us as `19:00-07` is different to the
value it is stored as — `19:00+00`.
* For TIMESTAMPTZ, we've got `01:00`, which looks way off. Remember — TIMESTAMPTZ
is a UTC offset, which displays in the session time zone. We've parsed
TIMESTAMPTZ with `+11:00` (which is parsed as ISO8601 format unlike SET TIME
ZONE) but we are displaying TIMESTAMPTZ in the time zone`-07:00` as we have set
the TIME ZONE to be in California. With an 18 hour time difference, that places
the opening ceremony at 1am local time in California, as you can see from the
output.

American readers, youse must have been tired watching the Olympics Games in 2000!

### Casting
Let's look at a few casts between TIMESTAMP and TIMESTAMPTZ with the time
[one of the greatest cricket test match finishes of all time](https://www.espncricinfo.com/series/england-tour-of-australia-2006-07-227733/australia-vs-england-2nd-test-249223/full-scorecard):

```
otan=# SET TIME ZONE 'Australia/Adelaide';
SET
otan=# -- case 1: timestamp -> timestamptz
otan=# SELECT '2006-12-05 17:00'::TIMESTAMP::TIMESTAMPTZ;
        timestamptz
---------------------------
 2006-12-05 17:00:00+10:30
(1 row)

otan=# -- case 2: timestamptz -> timestamp
otan=# SELECT '2006-12-05 17:00'::TIMESTAMPTZ::TIMESTAMP;
      timestamp
---------------------
 2006-12-05 17:00:00
(1 row)
```

When we cast from a TIMESTAMP to a TIMESTAMPTZ in Case 1, we have to set the
time zone of `+10:30`. However, this actually changes the UTC time, as
"displaying it correctly" in the current session time zone involves subtracting
`10:30` from the underlying offset value.

Conversely, when we convert from TIMESTAMPTZ to TIMESTAMP in Case 2, TIMESTAMP
represents an absolute UTC value but without the time zone information. As such,
we would have to add `10:30` to the underlying offset value for the correct
equivalent value to make this equal to the value `17:00`.

This behaviour gets tricky, especially when the data is stored and fetched in a
session with another session time zone whilst expecting the cast to TIMESTAMP to
give you the same result:

```
otan=# SET TIME ZONE 'Australia/Adelaide';
SET
otan=# CREATE TEMP TABLE timestamptz_table (val timestamptz);
CREATE TABLE
otan=# INSERT INTO timestamptz_table values ('2006-12-05 17:00'::TIMESTAMP);
INSERT 0 1
otan=# SELECT * from timestamptz_table;
            val
---------------------------
 2006-12-05 17:00:00+10:30
(1 row)

otan=# SELECT val::TIMESTAMP from timestamptz_table;
         val
---------------------
 2006-12-05 17:00:00
(1 row)

otan=# SET TIME ZONE 'America/Chicago';
SET
otan=# SELECT val::TIMESTAMP from timestamptz_table;
         val
---------------------
 2006-12-05 00:30:00
(1 row)
```

In the above example, we changed the time zone above from `+10:30` to Chicago's
`-06:00`, so we have a `+16:30` time difference. Since we cast to TIMESTAMP
using the TIMESTAMPTZ data type, we evaluate it as of the session time zone at
the point of evaluation, hence getting `2006-12-05 00:30:00` when casting it in
the `America/Chicago` timezone.

Note: if you want a result to always evaluate to the same TIMESTAMP no
matter the session time zone, use the `AT TIME ZONE` syntax discussed below.

#### Writing these cast operations in Golang
For many of these operations, we need to move timestamps to a different time
zone but whilst keeping the same timestamp underneath.  Go does not natively do
this, as [time.In](https://golang.org/pkg/time/#Time.In) only changes the
location, but has the same UTC offset underneath. This means that something at
`10:00` cast to a `+3` time zone without any offset changes would report as
`13:00+3` instead of `10:00+3`. This was a source of quite a few of our time
bugs!

To do this correctly, we have to do an awkward dance: read the second argument
from [`Zone`](https://pkg.go.dev/time#Time.Zone) to get the zone offset in
seconds from the timezones before and after and then use
[`time.Add`](https://pkg.go.dev/time#Time.Add) to subtract that duration offset
to the new time value.

Example code [(see it in action using the Go playground)](https://go.dev/play/p/gjXWI8AX-_0):
```go
func KeepTimeOffsetInNewZone(t time.Time, loc *time.Location) time.Time {
	afterTime := t.In(loc)
	_, beforeOffsetSecs := t.Zone()
	_, afterOffsetSecs := afterTime.Zone()
	timeDifference := afterOffsetSecs - beforeOffsetSecs
	return afterTime.Add(-time.Duration(timeDifference) * time.Second)
}
```
### Precision
TIMESTAMP and TIMESTAMPTZ supports microsecond precision. However, there is an
option for "rounding" of fractional digits for the seconds component.

To do this, we can specify a number between 0 and 6 inclusive in parenthesis
after TIMESTAMP and TIMESTAMPTZ, e.g. `TIMESTAMP(3)` for TIMESTAMP in millisecond
precision allowing 3 fractional digits, or `TIMESTAMPTZ(0)` for TIMESTAMPTZ with
no fractional digits. This will get the values rounded to the specified
precision.

Let's look at examples using the time when [Super Over rules were
unfair](https://en.wikipedia.org/wiki/2019_Cricket_World_Cup_Final):
```
otan=# -- rounded up
otan=# SELECT '2019-07-14 17:00:00.545454'::TIMESTAMP(0);
      timestamp
---------------------
 2019-07-14 17:00:01
(1 row)
otan=# -- rounded down
otan=# SELECT '2019-07-14 17:00:00.545454'::TIMESTAMP(1);
       timestamp
-----------------------
 2019-07-14 17:00:00.5
(1 row)
otan=# -- rounded up
otan=# SELECT '2019-07-14 17:00:00.545454'::TIMESTAMP(3); 
        timestamp
-------------------------
 2019-07-14 17:00:00.545
(1 row)
otan=# -- maximum precision and is default
otan=# SELECT '2019-07-14 17:00:00.545454'::TIMESTAMP(6); 
         timestamp
----------------------------
 2019-07-14 17:00:00.545454
(1 row)
```
#### Writing precision in Go
The equivalent functionality in Go is available using
[time.Round](https://pkg.go.dev/time#Time.Round) in the time library.

### Functions and Operators
[Functions](https://www.cockroachlabs.com/docs/stable/functions-and-operators.html)
(e.g. `extract`, `date_trunc`) and
[Operators](https://www.cockroachlabs.com/docs/stable/functions-and-operators.html#operators)
(e.g. `=`, `+`, `-`, `>`) are fairly easy to comprehend with TIMESTAMP. However
they are a little nuanced with TIMESTAMPTZ, causing a variety of bugs internally
in CockroachDB. As we alluded to earlier, it's important to remember that
TIMESTAMPTZ performs operations in the session time zone.

Let's look at some time operators, with timestamps around the release of the
movie Crocodile Dundee:
```
otan=# SET TIME ZONE 'America/New_York';
SET
otan=# -- Case 1
otan=# SELECT '1986-09-26 10:00'::TIMESTAMP = '1986-09-26 10:00-04'::TIMESTAMPTZ;
 ?column?
----------
 t
(1 row)

otan=# -- Case 2
otan=# SELECT '1986-09-26 10:00'::TIMESTAMP = '1986-09-26 09:00-05'::TIMESTAMPTZ;
 ?column?
----------
 t
(1 row)

otan=# -- Case 3
otan=# SELECT '1986-09-26 10:00'::TIMESTAMP < '1986-09-26 10:00-05'::TIMESTAMPTZ;
 ?column?
----------
 t
(1 row)

otan=# -- Case 4
otan=# SELECT '1986-09-26 10:00'::TIMESTAMPTZ + '1 day'::interval;
        ?column?
------------------------
 1986-09-27 10:00:00-04
(1 row)
```

Remember — in all cases, the TIMESTAMP is converted to the TIMESTAMPTZ of the
current session time zone, which is `1986-09-26 10:00-04` in New York.

Taking a look at each case:
* Case 1: the added `-04` for TIMESTAMP when converting to TIMESTAMPTZ means this result
 is true.
* Case 2: they are both internally the same time offset, and hence they are equal.
* Case 3: `10:00-05` can be thought of as `11:00-04`, which is strictly higher, hence
returning true.
* Case 4: adding an interval of 1 day changes the date to be incremented by 1.

Let's look at `extract` function:
```
otan=# SET TIME ZONE 'America/New_York';
SET
otan=# -- Case 1
otan=# SELECT extract('hour' from '1986-09-26 10:00'::TIMESTAMP);
 extract
---------
      10
(1 row)

otan=# -- Case 2
otan=# SELECT extract('hour' from '1986-09-26 10:00-04'::TIMESTAMPTZ);
 extract
---------
      10
(1 row)

otan=# -- Case 3
otan=# SELECT extract('hour' from '1986-09-26 10:00-06'::TIMESTAMPTZ);
 extract
---------
      12
(1 row)
```

From the shell output above:
* Case 1: The hour can be extracted as `10` directly.
* Case 2: As the timestamp provided is in the same time zone as the session time
zone, the calculation does not need any time zone conversions, but the
underlying UTC offset has a different hour. Thankfully, Go's time.Hour operator
(and Minute, Second, Month, etc.) takes into account what location the time is
in, so extracting the hour is straightforward.
* Case 3: What happened here? Remember — TIMESTAMPTZ performs operations in the
session time zone. If we were to run `SELECT '1986-09-26
10:00-06'::TIMESTAMPTZ`, we'd see `1986-09-26 12:00-04` as `-04` is the session
time zone. As such, when moving it to from `-02` to `-04`, the time becomes
`12:00` — and since we perform the operation in the session time zone — extract
will hence return `12`.

### AT TIME ZONE

AT TIME ZONE will convert a TIMESTAMPTZ to a TIMESTAMP at a given time zone, or
a TIMESTAMP to a TIMESTAMPTZ at a given time zone (which will be transposed into
the session time zone).

It is worth noting that `TIMESTAMP AT TIME ZONE <zone>` and T`IMESTAMPTZ AT TIME
ZONE <zone>` are inverses of each other.  This can be useful if you expect users
to cast from TIMESTAMPTZ to TIMESTAMP or vice versa from different session time
zones.

Let's look at the real examples using the release date of the hit song,
[Friday](https://en.wikipedia.org/wiki/Friday_(Rebecca_Black_song)):

```
otan=# SET TIME ZONE 'Australia/Sydney';
SET
otan=# -- Case 1
otan=# SELECT '2011-03-14 10:00:00'::TIMESTAMPTZ AT TIME ZONE 'Asia/Tokyo';
      timezone
---------------------
 2011-03-14 08:00:00
(1 row)

otan=# -- Case 2
otan=# SELECT '2011-03-14 10:00:00'::TIMESTAMP AT TIME ZONE 'Australia/Sydney';
        timezone
------------------------
 2011-03-14 10:00:00+11
(1 row)

otan=# -- Case 3
otan=# SELECT '2011-03-14 10:00:00'::TIMESTAMP AT TIME ZONE 'Asia/Tokyo';
        timezone
------------------------
 2011-03-14 12:00:00+11
```
From the above:

* Case 1: we are switching from `Australia/Sydney` time to `Asia/Tokyo` time, which is 2 hours behind.  
  This involves moving from 10am to 8am, but underneath removing the UTC time zone offset to the absolute
 "TIMESTAMP" value of 8am.
* Case 2: we have a TIMESTAMP which we wish to convert to Sydney time. This is straightforward -
  add the UTC offset to the time, which when we display in the session time zone of
  `Australia/Sydney` will still be 10am.
* Case 3: Case 3 gets interesting. We've added the `Asia/Tokyo` time zone offset to the underlying offset,
  but remember we display this offset at the session time zone. With the two-hour time difference, that means we see
  12pm in the afternoon when displaying this operation with a session time zone of Australia/Sydney.

#### POSIX standard goes out the window with AT TIME ZONE

Remember the surprise earlier with the POSIX standard being used for strings
when using `SET TIME ZONE`? In `AT TIME ZONE`, omitting the UTC/GMT prefix and
having just a bare integer offset (e.g. `+3`, `-3`, `3`) also behaves as POSIX
timestamp (unlike `SET TIME ZONE` where integers were special and behave as
ISO8601).

Let's look at an extreme example:
```
otan=# SET TIME ZONE '+3';
SET
otan=# select '2011-03-14 10:00:00'::TIMESTAMP AT TIME ZONE '+3';
        timezone
------------------------
 2011-03-14 16:00:00+03
(1 row)
```

We would expect the above case to be `2011-03-14 10:00:00+03:00` if it were
ISO8601 when using AT TIME ZONE. However, as `+3` is POSIX for AT TIME ZONE, it
really means "at time zone 3 hours **west** of GMT", to be displayed as "3 hours
**east** of UTC", hence adding 6 hours to the result.

### Daylight Savings
Now you may be wondering when to use a location versus when to use an absolute
offset. Offsets already seem tricky given how it uses POSIX style offsets.

This may help you decide — **word locations can infer changing time zone information**.
Let's look at a dates which traverse time zones in Chicago:

```
otan=# SET TIME ZONE 'America/Chicago';
SET
otan=# SELECT '2010-11-06 23:59:00'::TIMESTAMPTZ;
      timestamptz
------------------------
 2010-11-06 23:59:00-05
(1 row)

otan=# SELECT '2010-11-07 23:59:00'::TIMESTAMPTZ;
      timestamptz
------------------------
 2010-11-07 23:59:00-06
(1 row)
```

With the daylight savings boundary change, we can see that the time zone offset
changes. That's neat, isn't it?

You may now be wondering — how are these time zone changes encoded? [IANA
maintains a database](https://www.iana.org/time-zones) of daylight savings
changes for each time zone (some of which date back a very long time).

But which version of this database do we use? Well:
* CockroachDB will use a copy of this database that is installed within your computer. This is the default behaviour of Go.
* PostgreSQL ships out its own copy of tzdata every release.

This means that time and daylight savings behaviours can change between
computers when using CockroachDB if a newer version of the IANA database is on
your system.

This is [currently tracked for a future fix](https://github.com/cockroachdb/cockroach/issues/31978),
and is one of the reasons we recommend always using UTC as your session time
zone.

#### Interval Math with Daylight Savings

Let's look at how daylight savings impacts interval math:

```
otan=# SET TIME ZONE 'America/Chicago';
SET
otan=# -- Case 1
otan=# SELECT '2010-11-06 23:59:00'::TIMESTAMPTZ + '24 hours'::interval;
        ?column?
------------------------
 2010-11-07 22:59:00-06
(1 row)

otan=# -- Case 2
otan=# SELECT '2010-11-06 23:59:00'::TIMESTAMPTZ + '1 day'::interval;
        ?column?
------------------------
 2010-11-07 23:59:00-06
(1 row)

otan=# -- Case 3
otan=# SELECT '2010-11-06 23:59:00'::TIMESTAMPTZ + '1 month'::interval;
        ?column?
------------------------
 2010-12-06 23:59:00-06
(1 row)
```
Huh — are there not 24 hours in a day? INTERVALs in PostgreSQL are represented
as "months", "days" and "seconds", which plays a role in what we see here. Let's
see how this applies to the cases above:

* Case 1: When adding "seconds" to TIMESTAMPs (which is represented by units that
are not days, i.e. 24 hours still uses seconds until it overflows), we add real
world seconds. This is done in Go by using [time.Add](https://pkg.go.dev/time#Time.Add).
* Case 2: When adding "days",  we're just putting any math straight into the date 
fields, preserving the same time even as we cross daylight savings barriers. This is handled in Go by
[time.AddDate](https://pkg.go.dev/time#Time.AddDate).
* Case 3: Similar to case 2, but we add months instead of days.

#### Y2K38 - a (now resolved) Golang problem

Does `2038-01-19 03:14:07` spark any Y2K vibes? This is the time is `2147483647`
(max int32) seconds after the unix offset of `1970-01-01`, known as the [Y2K38
date](https://en.wikipedia.org/wiki/Year_2038_problem).

This was an issue in
[Go when handling tzdata past 2038](https://github.com/golang/go/issues/36654)
which has been resolved as of Go 1.14. Before Go 1.14, we also did timezones
incorrectly after 2038 as we were doing what Go did!

```
otan=# SET TIME ZONE '-9';
SET
otan=# SELECT '1947-12-13 13:00+11'::TIMESTAMPTZ AT TIME ZONE 'UTC+3';
      timezone
---------------------
 1947-12-12 23:00:00
(1 row)
```

### What do we recommend?

Like others, CockroachDB recommends usage of TIMESTAMPTZ as being to operate in
the correct timezone and parsing timezone fields is valuable.

However, we recommend always setting the session time zone to UTC. This allows
the user to not worry about not losing time zone information whilst parsing,
whilst allaying concerns that if a user decides to use session time zones that
they perform with intended daylight-savings aware behaviour.

### Time Twister

Feeling confident about time? Feeling you may be turning half human, half time lord?

Try explain the following behaviour:

```
otan=# SET TIME ZONE '-9';
SET
otan=# SELECT '1947-12-13 13:00+11'::TIMESTAMPTZ AT TIME ZONE 'UTC+3';
      timezone
---------------------
 1947-12-12 23:00:00
(1 row)
```


#### Answer
The session time zone is set at `-9`. This means `1947-12-13 13:00+11` would be
translated to `1947-12-12 17:00:00-09`.

Now UTC+3 is POSIX standard, meaning it really means 3 hours west of UTC (`-3`
in ISO8601). This is six hours ahead of the session time zone of `-9`, hence
translating to `23:00:00`. Since `AT TIME ZONE` translates a TIMESTAMPTZ to a
TIMESTAMP, the time zone data is gone.

And if you are curious, 1947-12-13 is the date of the [infamous Mankad
incident](https://en.wikipedia.org/wiki/Run_out#Running_out_a_batsman_%22backing_up%22_(Mankad)).
Now totally legal!

## TIME and TIMETZ

TIME (also known as TIME WITHOUT TIME ZONE) and TIMETZ (also known as TIME WITH
TIME ZONE) both only store the time of day component of a TIMESTAMP. But:

* TIME is still encoded with 8 bytes, representing microseconds since midnight.
* TIMETZ is encoded with 12 bytes, with 8 bytes representing microseconds since
midnight and 4 bytes for storing the time zone offset in seconds **west** of UTC
(the opposite of what we're "used to" when talking time zones). Unlike
TIMESTAMPTZ, the current session time zone is not taken into account (except for
parsing) and since it only stores time zone offsets (not actual location like
`Australia/Sydney`), it does not encode daylight savings information.

Let's look at using `CURRENT_TIME` (the equivalent of `CURRENT_TIMESTAMP`):

```
otan=# CREATE TABLE timetz_example (t time, ttz timetz);
CREATE TABLE
otan=# INSERT INTO timetz_example VALUES (CURRENT_TIME, CURRENT_TIME);

INSERT 0 1
otan=# SELECT t from timetz_example;
        t
-----------------
 23:25:38.691729
(1 row)

otan=#  SELECT ttz from timetz_example;
        ttz
--------------------
 23:25:38.691729-07
(1 row)

otan=# SET TIME ZONE 'Australia/Sydney';
SET
otan=# select t from timetz_example;
        t
-----------------
 23:25:38.691729
(1 row)

otan=# select ttz from timetz_example;
        ttz
--------------------
 23:25:38.691729-07
(1 row)
```

As you can see, changing the time zone does not affect table results. Since
TimeTZ stores the offset, they stay the same between session time zones shifts.

When performing interval math with time, times past `23:59:59.999999`,
automatically overflows back to `00:00:00` as there is no "date" component
(but there is `24:00:00` time - more on that later).

```
otan=# select '10:00'::time + '14 hours'::interval;
 ?column?
----------
 00:00:00
(1 row)

otan=# select '10:00+03'::timetz + '14 hours'::interval;
  ?column?
-------------
 00:00:00+03
(1 row)
```

### Parsing

Parsing TIME and TIMETZ largely behaves the same as the parsing for TIMESTAMP
and TIMESTAMPTZ:

* With TIME, any time zone offsets are ignored.
* With TIMETZ, the current session time zone appended to TIMETZ if none is
specified (which changes based on daylight savings). However, If a time zone
offset is specified for TIMETZ, it will use that instead. The time zone offsets
use the familiar ISO8601 standard.

```
otan=# SET TIME ZONE 'Australia/Sydney';
SET
otan=# SELECT '07:00'::time, '07:00'::timetz, '07:00-03'::time, '07:00-03'::timetz;
   time   |   timetz    |   time   |   timetz
----------+-------------+----------+-------------
 07:00:00 | 07:00:00+11 | 07:00:00 | 07:00:00-03
(1 row)
```

### Precision

Similar to TIMESTAMP/TIMESTAMPTZ, you can specify precision in parenthesis for
TIME/TIMETZ types, which round to specified precision of fractional digits for
the seconds component:

```
otan=# -- rounded up
otan=# select '17:00:00.545454'::time(0), '17:00:00.545454+03'::timetz(0);
   time   |   timetz
----------+-------------
 17:00:01 | 17:00:01+03
(1 row)

otan=# -- rounded down
otan=# select '17:00:00.545454'::time(1), '17:00:00.545454+03'::timetz(1);
    time    |    timetz
------------+---------------
 17:00:00.5 | 17:00:00.5+03
(1 row)
```

### Casting

When casting TIME to TIMETZ, the TIME will get promoted to the time zone of your
current session. However, when casting TIMETZ to TIME, we will lose time zone
offset:

```
otan=# SET TIME ZONE 'Australia/Sydney';
SET
otan=# SELECT '10:00'::time::timetz, '10:00+03'::timetz::time;
   timetz    |   time
-------------+----------
 10:00:00+11 | 10:00:00
(1 row)
```

Losing the time zone offset and reinterpreting it in the session time zone can
be surprising, as casting that TIME back to TIMETZ will give you a different
result. In other words, casting the inverse of the inverse does not yield the
identity. You can see an example of this below:

```
otan=# SET TIME ZONE 'Australia/Sydney';
SET
otan=# SELECT '10:00+03'::timetz::time::timetz;
   timetz
-------------
 10:00:00+11
(1 row)
```

Here, our `+03` in our TIMETZ has been reinterpreted in the session time zone of
`+11` after casting it to TIME, which represents a wholly different result. If you
need inverses to match, AT TIME ZONE between TIME and TIMETZ will yield the
desired effects.

### Comparators and Ordering of TIMETZ

Consider the comparison of these two equivalent times in UTC -
`10:00+03` and `11:00+04`:

```
otan=# -- Case 1
otan=# SELECT '10:00+03'::timetz = '10:00+03'::timetz;
 ?column?
----------
 t
(1 row)

otan=# -- Case 2
otan=# SELECT '10:00+03'::timetz = '11:00+04'::timetz;
 ?column?
----------
 f
(1 row)

otan=# -- Case 3
otan=# SELECT '10:00+03'::timetz > '11:00+04'::timetz;
 ?column?
----------
 t
(1 row)

otan=# -- Case 4
otan=# SELECT '10:00+03:00'::timetz < '11:01+04:00'::timetz;
 ?column?
----------
 t
(1 row)
```

That's interesting — `10:00+03` and `11:00+04` are the same time in the real
world, but in the realm of TimeTZ one clearly has precedence over the other!

Recall that TimeTZ stores both microsecond offset AND offset representing
seconds **west** of UTC. If the microsecond offset is the same, then we compare
seconds **west** of UTC (i.e. the negative of the offset we see above).

As such:
* Case 1 demonstrates that the same UTC offset AND time zone offset being equal
means the result is equal, as expected.
* Case 2 demonstrates that despite having the same UTC offset, the results are NOT
equal since the offset TIME ZONE is not the same.
* Case 3 demonstrates that timezones "more west" (has a higher POSIX offset) have
precedence.
* Case 4 demonstrates that UTC offsets take precedence - since `11:01+04` is one
minute higher relative to UTC compared to `10:00+03`, it returns true

### 24:00:00 in TIME/TIMETZ

An interesting feature that is only supported for TIME and TIMETZ is `24:00:00`
time. This is a time that can be parsed, but you cannot use arithmetic to
achieve the value. Adding to `24:00:00` overflows the value back to `00:00:00`.

```
otan=# -- 24:00 time can be parsed as such
otan=# SELECT '24:00'::time;
   time
----------
 24:00:00
(1 row)

otan=# -- but trying to reach 24:00 via arithmetic overflows to 00:00.
otan=# SELECT '23:59'::time + '1 minute'::interval;
 ?column?
----------
 00:00:00
(1 row)

otan=# --— and adding to 24:00 time will overflow it to 00:00.
otan=# SELECT '24:00'::time + '1 second'::interval;
 ?column?
----------
 00:00:01
(1 row)

otan=# -- even adding 0 seconds will overflow.
otan=# SELECT '24:00'::time + '0 second'::interval;
 ?column?
----------
 00:00:00
(1 row)
```

#### Handling 24:00 time in Go

Unfortunately, Go's [time.Parse](https://pkg.go.dev/time#Parse) does not handle 24:00 —
so our TIME string parsers all have wrappers  to regex match and handle the 24:00 case.


### What do we recommend?
Whilst TIME and TIMETZ store only the time component, TIME takes the same amount
of space — and even more for TimeTZ. Furthermore, TIMETZ also does not keep
track of location with time offsets. `24:00` time is an interesting case that is
handled but may have limited practical uses.

It is worth also noting that PostgreSQL advises against using the TIMETZ data
type. TIMETZ was originally implemented to follow the SQL standard. See the note
just above [section 8.1 in PostgreSQL's own documentation](https://www.postgresql.org/docs/current/datatype-datetime.html#DATATYPE-DATETIME-INPUT).

So our recommendation is to use ... neither! If you need time, you are most likely
better off using TIMESTAMPTZ which takes care of these shortfalls in all these
cases. If time zone information is required, use a separate column to encode
that information.

## Conclusion

Each of the time types have interesting nuances:

* TIMESTAMP is a type with date and time that has no time zone information.
  Behaviour does not changed based on the session time zone.
* TIMESTAMPTZ is a type with date and time that has time zone behaviour dependent
on your session time zone and  is conscious of daylight savings). It never
persists time zone initialized with it. If you want to be safe, always use UTC
with it.
* TIME is a type with time of day info only.
* TIMETZ is a type with time of day and fixed time zone offsets. It does not
depend on the session time zone for computation, and is not daylight savings
aware.

We generally recommend to always use TIMESTAMPTZ with your session time zone set
to UTC. If you need time zone information, use a separate column to store this
information. However, at the end of the day, pick works for you!

Whew, that was confusing. Maybe I should [write a strongly worded letter](https://frinkiac.com/meme/S04E19/1034282.jpg?b64lines=VEhFUkUgQVJFIFRPTyBNQU5ZIApUSU1FIFRZUEVTIE5PV0FEQVlTLgpQTEVBU0UgRUxJTUlOQVRFIFRIUkVFLgpJIEFNIE5PVCBBIENSQUNLUE9ULg==)
to the folk who write the SQL Standard! In any
case, working with time data types is indeed interesting — and we haven't even
touched the INTERVAL type and [leap seconds](https://en.wikipedia.org/wiki/Leap_second)!

Did you enjoy our adventures and deep dive into the internals and nuances of
databases? As always, [we're hiring](https://www.cockroachlabs.com/careers/)!
