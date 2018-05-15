# A review of C++ `<chrono>` design

This document reviews the overall design of the C++ `<chrono>` library. There is
an insane amount of documentation available in the [Bibliography][bibliography]
section, this review cannot cover it all.

## Overview of the C++ standard library time facilities

The C++ standard library time facilities is a collection of libraries that build
on top of each other:

* `<ratio>`: compile-time rational-numbers arithmetic: `ratio<Numerator,Denominator>`
* `<chrono>`: clocks, time points, durations
* `<date>`: additional durations (days, weeks, months, years), additional time
  points (system time, local time), calendar types (`year_month_day`, ...),
  partial calendar types (`weekday`, `month_weekday`, ...)
* calendar libraries: build on top of `date`:
  * `<iso_week>`: iso-week date calendar (Gregorian)
  * `<julian>`: Julian calendar
  * `<islamic>`: Iaslamic calendar
  * ...
* `<tz>`: time-zone support: complete parser of the IANA data-base using the
  types of `<date>` and `<chrono>`. It adds a data-base type with facilities to
  query relevant information (like leap seconds), it extends `<chrono>` with new
  clock types (`utc`, `tai`, and `gps`), it provides a time-zone aware time
  point `zoned_time<Duration>`.
  
This document focuses on `<chrono>` and considers interactions with the other
libraries of the "time" stack.

## Overview of `<chrono>`

The `<chrono>` library is built on top of `<ratio>` (compile-time rational
numbers: `std::ratio<N, D>`), and exposes the following components:

* duration: a span of time, defined as some number of ticks of some time unit. 
* clocks: a starting point (epoch from now on) and a tick rate. 
* time point: a duration since the epoch of a clock
* io: formatting and parsing utilities

### `I/O`

Many types in the time facilities provide the following generic I/O operations:

* `operator<<` performs stream output
* `from_stream` parses from a stream according to the provided format 
* `to_stream` outputs into a stream according to the provided format 

## Durations

The `duration<Representation, TickPeriodInSeconds>` class template represents a
time interval, where

* `Representation`: is an arithmetic type used to store a tick count
* `TickPeriodInSeconds`: is a compile-time rational constant representing the
  number of seconds per tick

The only thing the `duration` stores is an internal count of ticks. This count
has type `Representation`, and the `duration` type has `sizeof(Representation)`.
The `TickPeriodInSeconds` is used to convert between different `duration` types.

If `Representation` is a floating point number type then the duration can
represent fractions of ticks.

The API of `duration` provides the following:

* `count()`: returns the count of ticks
* `zero`: constant, zero-length duration
* `min`: constant, duration with the lowest possible value
* `max`: constant, duration with the largest possible value
* binary arithmetic operators: `D2 operator+(D0,D1)`,`D2 operator-(D0,D1)`, ...
  `+=`, `-=`, ... where the two types involved don't need to be equal, and the
  result does not need to be equal to any of them
* comparisons: `bool operator==(D0,D1)`, `!=`, `<`, `<=`, ... where the two
  types involved don't need to be equal
* `D1 duration_cast(D0)`: converts a duration from one type to another. See
  [docs](http://en.cppreference.com/w/cpp/chrono/duration/duration_cast) for
  more info about truncation, `NaN` to integer, etc.
* `floor`, `ceil`, `round`: like their floating-point number equivalents
* `abs` (provided for signed representation types only): like the floating-point equivalent
* I/O facilities

The library defines the following type aliases of `duration` using `ratio`:

* `nanoseconds duration</*signed integer type of at least 64 bits*/, ratio<1, 1000000000>>`
* `microseconds duration</*signed integer type of at least 55 bits*/,ratio<1, 1000000>>`
* `milliseconds duration</*signed integer type of at least 45 bits*/, ratio<1, 1000>>`
* `seconds = duration</*signed integer type of at least 35 bits*/, ratio<1,1>>`
* `minutes = duration</*signed integer type of at least 29 bits*/, ratio<60, 1>>`
* `hours = duration</*signed integer type of at least 23 bits*/, ratio<3600, 1>>`
* `days  = duration</*signed integer type of at least 25 bits*/, ratio<86400, 1>>`
* `weeks  = duration</*signed integer type of at least 22 bits*/, ratio<604800, 1>>`
* `months = duration</*signed integer type of at least 20 bits*/, ratio<2629746, 1>>`
* `years = duration</*signed integer type of at least 17 bits*/, ratio<31556952, 1>>`

And user-defined literals for these types: `h` for `hours`, `min` for `minutes`,
etc. This allows `auto three_hours = 3h;`.

Becuase the duration type is generic, users can easily define their own durations:

```c++
constexpr auto year = 31556952ll; // seconds in average Gregorian year
using microfortnights = std::chrono::duration<float, std::ratio<14*24*60*60, 1000000>>;
using nanocenturies = std::chrono::duration<float, std::ratio<100*year, 1000000000>>;
```

### clocks

The `Clock` concept specifies the interface of all clocks:

* `Clock::rep` is the arithmetic type used in the internal representation of `Clock::duration`
* `Clock::period` is the type of a compile-time rational number, that is, a
  `std::ratio<N, D>` specifying the tick period of the clock in seconds
  (`N`/`D`).
* `Clock::duration` is the duration type of the clock which is just an 
  instance of `std::chrono::duration<Clock::rep, Clock::period>`
* `Clock::time_point` is the time point of the clock which is just an instance
  of `std::chrono::time_point<Clock>` or `std::chrono::time_point<OtherClock, Clock::duration>`.
  
* `Clock::is_steady` is a compile-time constant that is `true` if `t1 <= t2`
  (where `t2` is produced by a call to `now` that happens after the call to
  `now` that produces `t1`) is always true and the time between clock ticks is
  constant, otherwise it is `false`.
  
* `Clock::now() -> Clock::time_point` returns a time-point representing the
  current point in time.

The type trait `is_clock` can be used to query whether a type is a `Clock`.

There are different kinds of clocks:

* `steady` aka monotonic: see `Clock::is_steady` above
* `realtime`/`async_progress`/`systemwide`: wall clock time from the system-wide realtime clock

The `realtime` clock property is severely unspecified:

* there does not seem to be a way to query whether a clock is a `realtime` clock
* only `realtime` clocks are guaranteed to be advanced while their associated thread sleeps
* there does not seem to be a definition of "associated clock thread" in the library

However, `realtime` clocks are critical for for timeouts. If the clock is not
`realtime`, a thread that sleeps might never be awaken because the clock is
never advanced.

The C++11 version of the library contained three clock types:

* `system_clock`: wall clock time from the system-wide realtime clock 
  * provides `to_time_t`/`from_time_t` functions thatmap it to C's `time_t`.
* `steady_clock`: monotonic clock that will never be adjusted 
* `high_resolution_clock`: the clock with the shortest tick period available
  which might be an alias to either `system_clock` or `steady_clock`.

The `<chrono>` library was updated in C++20 with clocks that provide time-zone
support: 

* `utc_clock`: Clock for Coordinated Universal Time (UTC) 
   * `from_sys`/`to_sys` functions converting time points from/to `system_clock`.
* `tai_clock`: Clock for International Atomic Time (TAI) 
   * `from_utc`/`to_utc` functions converting time points from/to `utc_clock`.
* `gps_clock`: Clock for GPS time 
   * `from_utc`/`to_utc` functions converting time points from/to `utc_clock`.
* `file_clock`: Clock used for file time 
   * `from_utc`/`to_utc` functions converting time points from/to `utc_clock`.
   * `from_sys`/`to_sys` functions converting time points from/to `system_clock`.
* `local_t`: pseudo-clock representing local time 

@HowardHinnat mentioned in [this
discussion](https://groups.google.com/a/isocpp.org/forum/#!topic/std-discussion/D_S47zTnNmw):

> I would not be opposed to deprecating high_resolution_clock, with the intent
> to remove it after a suitable period of deprecation. The reality is that it is
> always a typedef to either steady_clock or system_clock, and the programmer is
> better off choosing one of those two and know what he’s getting, than choose
> high_resolution_clock and get some other clock by a roll of the dice.

While that particular discussion did not appear to achieve consensus it shows
that exposing a "clock with the shortest tick period available" is not something
that can be solved by just simply adding a new clock type, since that can result
in a type that is hard to use correctly in practice.

The [`<chrono>` clock survey][chrono_clock_survey] shows that:

* `high_resolution_clock` has the same period as `steady_clock` in all major
  platforms (Linux, Windows, and MacOSX).
* all major platforms use a 64-bit integer type as the internal representation
  type of the duration type.

### time point

The `time_point<Clock, Duration>` class template represents a point in time as a
`Duration` from the `Clock`'s epoch, so that `sizeof(time_point<Clock,
Duration>) == sizeof(Duration)`.

The API is:

* `time_since_epoch`: returns the time point as duration since the start of its clock
* `+=`,`-=`: modifies the time point by a `Duration` of the same type as that of the `time_point`.
* `++`,`--`: increments/decrement the duration by one tick
* `+`, `-`: works on:
    * time points returning durations: `(time_point<C,D0>, time_point<C,D1>) -> D2` 
    where `D2` is a `Duration` that is appropriate for representing the
    result of the operation (it is obtained via `common_type_t<D0,D1>`)
   * time points and durations (and vice-versa): `(time_point<C,D0>, D1) -> time_point<C,D2>`
* `min`/`max`: constants returning the time points corresponding to the smallest and largest durations
* `time_point_cast(time_point<SameClock, DifferentDuration>)`: converts the time
  point to another one on the same clock but with a different duration type.
* `floor`, `ceil`, `round`: perform the respective operaiton on the `Duration`.

#### `<date>` extensions

The C++ `<date>` library adds the following `time_point` aliases:

* `sys_days = time_point<system_clock, days>`
* `sys_seconds = time_point<system_clock, seconds>`

The C++ `<date>` library distinguishes between time-points with "serial layout",
like `time_point`, and time-points with "field layout", like the following
time-point types with a one-day resolution:

* `year_month_day`
* `year_month_weekday`
* `year_month_day_last`

IIUC all these types represent "time-points" but the library does not appear to
have a `TimePoint` concept though.

The `<date>` library documentation explains that both layouts are good at some
operations and bad at others. The different layouts expose only those operations
that they are good at, and these must interoperate via conversions. The [`date`
algorithms][date_algorithms] documentation describes efficient implementations
of these conversions. For example (taken verbatim from the `<date>`
documentation):

* Field types are good at returning the values of the fields. Serial types
  aren't (except for weekdays). So `year_month_day` has accessors for `year`,
  `month` and `day`. And `sys_days` does not.

* Field types are good at month and year-oriented arithmetic. Serial types
  aren't. So `year_month_day` has month and year-oriented arithmetic. And `sys_days`
  does not.

* Serial types are good at day-oriented arithmetic. Field types aren't. So
  sys_days has day-oriented arithmetic. And `year_month_day` does not. Though one
  can perform day-oriented arithmetic on the day field of a `year_month_day`, with
  no impact on the other fields.

* To efficiently compute a day of the week, one first needs to compute a serial
  date. So weekday is constructible from `sys_days`.

The `<date>` library also provides facilities to construct time-points with
one-day resolution, like `auto date = 2015_y/mar/22;`. Also, this library allows
users to construct invalid time-points, and provides an `ok` method to check
whether time-point is valid.

The `<date>` library uses 32-bit integers to represent minutes, but this
overflow after 4000 years. It might have made more sense to use 64-bit integers for these.

#### `<tz>` extensions

The `<tz>` library introduces a `zoned_time` time-point.

## Bibliography
[bibliography]: #bibliography

### General

* [0] Howard Hinnant's [website][howard_web], containing many documents about
  the rationale and design of `date` and `tz`
* [1] The C++ [`date` and `tz` libraries][date], its `readme` contains many
  links to the relevant documentation, talks, and design documents.
  
### `<chrono>`
  
* [2] [N2661: A Foundation to Sleep On - Clocks, Points in Time, and Time
  Durations][chrono_proposal] is the ISO proposal for C++11's `<chrono>` 

* [3] [`<chrono>` utilities][chrono_utilities]
* [4] [`<chrono>` I/O][chrono_io] - i/o facilities and `floor`, `round`, and `ceil`.
* [5] [N3531: User-defined Literals for Standard Library Types (version 3)][udl_paper]
  added `std::chrono::duration`'s suffixes `h`, `min`, `s`, `ms`, `us`, `ns` in
  inline namespace `std::literals::chrono_literals` to C++14.
* [6] [P0092R1: Polishing `<chrono>`][chrono_cpp17] includes the `<chrono>`
  changes for C++17, which add alternative rounding modes for `durations` and
  `time_points` (`floor`, `ceil`, `round`), and it adds `abs` for signed duration types.
* [7] [`<chrono>` clock survey][chrono_clock_survey]
* [8] [A `<chrono>` tutorial][a_chrono_tutorial]

### `<date>`

* [ ] [`date`][date_description]
* [ ] [N3344: Toward a Standard C++ `Date` Class][towards_date]
* [ ] [`date` algorithms][date_algorithms]

### `<tz>`

* [ ] [`tz`][tz_description]

[howard_web]: https://howardhinnant.github.io/
[date]: https://github.com/HowardHinnant/date
[chrono_proposal]: http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2008/n2661.htm
[chrono_utilities]: https://howardhinnant.github.io/duration_io/chrono_util.html
[chrono_io]: https://howardhinnant.github.io/duration_io/chrono_io.html
[udl_paper]: http://www.open-std.org/JTC1/SC22/WG21/docs/papers/2013/n3531.pdf
[chrono_cpp17]: http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2015/p0092r1.html
[chrono_clock_survey]: http://howardhinnant.github.io/clock_survey.html
[a_chrono_tutorial]: https://schd.ws/hosted_files/cppcon2016/d8/%20chrono%20Tutorial%20-%20Howard%20Hinnant%20-%20CppCon%202016.pdf
[date_description]: https://howardhinnant.github.io/date/date.html
[towards_date]: http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2012/n3344.pdf
[date_algorithms]: http://howardhinnant.github.io/date_algorithms.html
[tz_description]: https://howardhinnant.github.io/date/tz.html
