---
tagline: tweening for animation
---

## `local tw = require'tweening'`

A library for the management of gradual value changes for the purposes of
animation.

## Features

  * timing parameters: `duration`, `delay`, `speed`, `loop`, `backwards`,
  `yoyo`, `ease`, `way`, `offset`.
  * independent timing parameters for each target object and/or attribute.
  * timelines that can be nested, overlapping, and tweened.
  * stagger tweens: alternate end-values over a list of targets.
  * extensible attribute types, value converters and interpolation functions.
  * relative values including directional rotation.
  * no allocations while tweening.

## Tweens

### `tw:tween(t) -> tween`

Create a new tween. A tween is an object which repeatedly updates a value on
a target object _in time_ based on an interpolation function and timing
parameters.

__NOTE:__ `t` itself is turned into a tween (no new table is created). This
allows any method to be overriden by specifying it directly as a field of `t`.

### Timing model

__field__      __default__   __description__
-------------- ------------- -------------------------------------------------
`start`        current clock start clock (relative when in a timeline)
`duration`     `1`           duration of one iteration (can't be negative)
`ease`         `'quad'`      ease function `f(p) -> d` or name from [easing] module
`way`          `'in'`        easing way: `'in'`, `'out'`, `'inout'` or `'outin'`
`ease_args`    none          optional args to pass to the ease function
`backwards`    `false`       start backwards
`yoyo`         `true`        alternate between forwards and backwards on each iteration
`loop`         `1`           number of iterations (can be fractional; use `1/0` for infinite)
`delay`        `0`           start delay (on the first iteration; can be negative)
`speed`        `1`           speed factor; must be > 0
`offset`       `0`           progress at start
`running`      `true`        running or paused
`clock`        `start`       current clock when running, set by `update()`
`resume_clock` `clock`       current clock when paused, set by `update()`

__NOTE__: The difference between `start` and `delay` is that when in the
`delay` poriton, the tween is considered to be running, and the target value
is updated as such.

__NOTE__: Timing model fields can be changed anytime (there's no internal state).

__method__                   __description__
---------------------------- --------------------------------------------------
`total_duration() -> dt`     total duration incl. repeats (can be `inf`)
`end_clock() -> t`           end clock (can be `inf`)
`is_infinite() -> bool`      true if `loop` and/or `duration` are `inf`
`clock_at(P) -> t`           clock at total linear progress
`is_backwards(i) -> bool`    true if iteration `i` goes backwards
`total_progress([t]) -> P`   linear progress in `0..1` incl. repeats (can be `inf`)
`status([t]) -> status`      status: 'before_start', 'running', 'paused', 'finished'
`progress([t]) -> p, i`      progress in `0..1` in current iteration, and iteration index
`distance([t]) -> d`         eased progress in `0..1` in current iteration
`update([t])`                update internal clock and target value

#### Changing the state of the tween

__method__           __description__
-------------------- ---------------------------------------------------------
`pause()`            pause (changes `running`, prevents updating `clock`)
`resume()`           resume (changes `running` and `start`)
`seek(P)`            update target value based on total progress (changes `start`).
`stop()`             pause and remove from timeline
`restart()`          restart (changes `start`)
`reverse()`          reverse (changes `start` and `backwards`; `duration` must be finite)

__NOTE:__ The methods above change the timing model of the tween, which is
normally supposed to be immutable in order to make the tween stateless. This
is done to allow a tween that is part of a timeline to be paused and resumed
independently of other tweens and of the timeline. Use `clone()` to preseve
the initial definition of the tween.

### Animation model

__field__          __default__         __description__
------------------ ------------------- ---------------------------------------
`target`           (required)          target object
`attr`             (required)          attribute in the target object to tween
`from`             `target[attr]`      start value (defaults to target's value)
`to`               `target[attr]`      end value (defaults to target's value)
`type`             per `attr`          force attribute type
`interpolate`      default for `type`  `f(t, x1, x2[, xout]) -> x`
`value_semantics`  default for `type`  (see below)
`get_value() -> v` `target[attr] -> v` value getter
`set_value(v)`     `target[attr] = v`  value setter

__NOTE:__ Animation model fields are read/only. See below for how to add
attribute type matching rules and interpolators.

### Relative values

`start_value` and `end_value` can be given in the format `'<number>'`,
`'<number><unit>`, `'<operator>=<number>'` or `'<operator>=<number><unit>'`,
where operator can be `+`, `-` or `*` and units can be one of:

__unit__     __description__
------------ -----------------------------------------------------------------
`%`          percent, converted to `0..1`
`deg`        degrees, converted to radians
`cw`         rotation: clockwise
`ccw`        rotation: counter-clockwise
`short`      rotation: shortest direction
`deg_cw`     rotation: clockwise in degrees, converted to radians
`deg_ccw`    rotation: counter-clockwise in degrees, converted to radians
`deg_short`  rotation: shortest direction in degrees, converted to radians
------------ -----------------------------------------------------------------

Examples: `+=10deg`, `25%`. Unit converters are extensible (see below).

__NOTE:__ `cw` and `ccw` assume that increasing angles rotate the target
clockwise (like cairo and other systems where the y-coord grows from top
to bottom).

__NOTE:__ Don't combine relative rotations wuth `cw` and `ccw`, it's
confusing. `+=90deg` means "rotate the target another 90 degrees clockwise",
while `90deg_cw` means "rotate the target to the 90 degrees mark going
clockwise".

### Misc.

__method__                   __description__
---------------------------- --------------------------------------------------
`tween:clone() -> tween`     clone a tween
`tween:totarget() -> obj`    convert to tweenable object

### `tween:clone() -> tween`

Clone a tween in its current state.

### `tween:totarget() -> obj`

Create a proxy object for the tween with the additional tweenable field
`total_progress`. Other tweenable properties like `speed` remain accessible.

## Timelines

### `tw:timeline(t) -> tl`

Create a new timeline. A timeline is a list of tweens which are updated in
parallel (or one after another, depending on their `start` time). The list
can also contain other timelines, resulting in a hierarchy of timelines.

__NOTE:__ `t` itself is turned into a timeline (no new table is created).

### Timelines are tweens

A timeline is itself a tween, containing all the fields and methods of the
timing model of a tween (but none of the fields and methods of the animation
model since it's not animating a target object, it's updating other tweens).
This means that a timeline can be used to _tween_ the _total progress_ of its
child tweens instead of just updating them on the same clock, since it has a
`distance` resulting from its timing model. It also means that the timeline
can be itself tweened on its _total progress_ (which only works if the
timeline is _not_ infinite).

### Tweening other tweens

Setting `tween_progress = true` on a timeline switches the timeline into
tweening its child tweens on their _total progress_ (so tweening a temporal
value) instead of just updating them on the same clock. In this mode, the
child's `start` and `duration` are ignored: instead, the timeline's `distance`
is interpolated over the child's total progress.

### Timeline-specific fields and methods

__field__        __default__ __description__
---------------- ----------- -------------------------------------------------
`tweens`         `{}`        the list of tweens in the timeline
`ease`           `'linear'`  (a better default for the `tween_progress` mode)
`duration`       `0`         auto-adjusted when adding tweens
`auto_duration`  `true`      auto-increase duration to include all tweens
`auto_remove`    `true`      remove tweens automatically when finished
`tween_progress` `false`     progress-tweening mode

__method__                  __description__
--------------------------- --------------------------------------------------
`add(tween[, start])`       add a tween
`replace(tween[, start])`   replace/add a tween
`remove(tween|attr|target)` remove matching tweens recursively
`clear() -> true|false`     remove all tweens (non-recursively)
`status()`                  adds `'empty'`

### `tl:add(tween[, start]) -> tl`

Add a new tween to the timeline, set its `start` field and its `timeline`
field, and, if `auto_duration` is `true`, increase timeline's `duration`
to include the entire tween. When part of a timeline, a tween's `start`
is relative to the timeline's start time. If `start` is not given, the
tween is added to the end of the timeline (when the timeline's duration is
infinite then the tween's start is set to `0` instead).

### `tl:replace(tween[, start]) -> tl`

Add a tween or replace an existing tween with the same target and attr.

### `tl:remove(tween|attr|target) -> true|false`

Remove a tween or all tweens with a specific attribute or target object
recursively. Returns true if any were removed.

### `tl:clear() -> true|false`

Remove all tweens from the timeline. Returns true if any were removed.

## The tweening module

### `tw() -> tw`

Create a new `tweening` module. Useful for extending the `tweening` module
with new attribute types, new interpolators or different ways of getting the
wall clock without affecting the original module table.

### Attribute types and interpolators

### `tw.type.<name|pattern|match_func> = attr_type`

Tell tweening about the type of an attribute, eg.
`tw.type['_color$'] = 'list'`

### `tw.interpolate.<attr_type> = function(d, x1, x2[, xout]) -> x`

Add a new interpolation function for an attribute type.

### `tw.value_semantics.<attr_type> = false`

Declare an interpolation function as having reference semantics. By default
interpolation functions have value semantics, i.e. they are called as
`x = f(d, x1, x2)`. If declared as having reference semantics, they are
instead called as `f(d, x1, x2, x)` and are expected to update `x` in-place
thus avoiding an allocation on every frame if `x` is a non-scalar type.

### Value parsers

### `tw:parse_value(s, relative_to, attr_type, attr) -> v`

Parse a value.

### The wall clock

### `tw:clock([t|false]) -> t`

Get/freeze/unfreeze the wall clock. With `t` it freezes the wall clock such
that subsequent `tw:clock()` calls will return `t`. With `false` it unfreezes
it such that subsequent `tw:clock()` calls will return `tw:current_clock()`.

Freezing is useful for creating multiple tweens which start at the exact same
time without having to specify the time.

### `tw:current_clock() -> t`

Current monotonic performance counter, in seconds. Implemented in terms of
[time.clock()][time]. Override this in order to remove the dependency on the
[time] module.

### Easing

### `tw:ease(ease, way, p, i) -> d`

Implemented in terms of [easing.ease()][easing]. Override this in order to
remove the dependency on the [easing] module.