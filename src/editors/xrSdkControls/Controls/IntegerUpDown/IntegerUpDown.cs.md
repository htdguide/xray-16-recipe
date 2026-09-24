# src/editors/xrSdkControls/Controls/IntegerUpDown/IntegerUpDown.cs

> A spin box over whole numbers, with a held-button acceleration table, optional hexadecimal display, and a strict split between "the text" and "the value".

**Needs** — [`IntegerUpDownAccelerationCollection.cs`](IntegerUpDownAccelerationCollection.cs.md) · [`IntegerUpDownAcceleration.cs`](IntegerUpDownAcceleration.cs.md)
**Used by** — [`IntegerSlider.Designer.cs`](../IntegerSlider/IntegerSlider.Designer.cs.md) · [`IntegerSlider.cs`](../IntegerSlider/IntegerSlider.cs.md) · [`IntegerUpDownAcceleration.cs`](IntegerUpDownAcceleration.cs.md)
**Tier floor** — T3: text parsing, clamping and a timer-driven step; nothing device-facing.

## Purpose

The stock spin box in the host toolkit works in decimal fractions. The editor needs whole numbers, sometimes shown in hexadecimal (colour bytes, flag masks), and needs a value that is *never* out of range even mid-edit. This control is an adaptation of the stock one, rewritten around an integer.

Why it is worth a page: it encodes the answer to a question every numeric input faces — **what is the value while the author is halfway through typing it?**

## State

```text
RECORD IntegerUpDown
  value            : int                     # the committed value
  minimum, maximum : int                     # invariant: minimum <= maximum
  increment        : int                     # invariant: >= 0
  hexadecimal      : bool                    # display base
  user_edit        : bool                    # the text is ahead of the value
  value_changed    : bool                    # the value is ahead of the text
  initializing     : bool                    # range and value are being set, order unknown
  accelerations    : list<Acceleration>      # sorted by hold time, ascending
  accel_index      : optional<int>           # which entry is active; none = not accelerating
  hold_started_at  : optional<timestamp>
```

**Invariants** — outside an initialization bracket, `minimum <= value <= maximum` always holds; a write outside the range is rejected as a caller error rather than clamped, because the caller declared the range. Setting `minimum` above `maximum` drags `maximum` up with it (and the reverse), so the range is never inverted even when the two are set in the wrong order. `user_edit` and `value_changed` are never both meaningful: whichever is set names which side is stale.

## The text/value split

**Contract** — the displayed text and the stored value are two representations that are reconciled at four moments, and at no other time: when a spin step is taken, when focus is lost, when the value is read programmatically, and at the end of an initialization bracket.

```text
FUNCTION parse_text()                 # text -> value
  IF text is empty OR text is just a minus sign THEN keep value
  ELSE value = clamp(parse(text, base = hex ? 16 : 10))
  # a parse failure leaves the value alone: the author is mid-typing, not wrong
  user_edit = false

FUNCTION render_text()                # value -> text
  IF initializing THEN RETURN
  IF user_edit THEN parse_text()      # never render over an unparsed edit
  text = format(value, base = hex ? 16 : 10)
```

**Notes** — the lone-minus-sign case is the whole reason the split is explicit: a negative number is typed one character at a time, and the first character alone is not a number. Rejecting it would make negative values untypeable. Every numeric input in every language meets this; the decision recorded here is **a failed parse is not an error, it is an incomplete edit, and the previous value stands**.

Key filtering is the same idea from the other side: digits, the locale's decimal, group and negative signs, and backspace are admitted; hexadecimal letters only when the display base is sixteen; anything else is swallowed. The filter is permissive (it admits a decimal separator into an integer field) because the parse is the real gate.

## Acceleration

**Contract** — holding a spin button or an arrow key makes the step grow. The acceleration table is a list of (hold duration, step size) entries kept sorted by duration. While a hold is in progress, the control advances to the next entry once the elapsed hold time passes that entry's duration, and the active entry's step replaces the configured increment.

```text
FUNCTION advance_acceleration()
  IF not holding OR accel_index is the last entry THEN RETURN
  IF now - hold_started_at > duration_of(accel_index + 1)
    hold_started_at = now             # the clock restarts at each promotion
    accel_index = accel_index + 1
```

**Invariants** — a hold ends (and the index resets to none) on button or key release, and *also* when a step hits either end of the range. That second rule is the interesting one: without it, an author who holds the button until the value saturates and then reverses gets an enormous first step in the other direction.

**Notes** — the table is per-control and usually empty; an empty table means no acceleration and the configured increment always applies. A rebuild can replace the table with a curve, but must keep the saturation reset.

## `UpButton` / `DownButton`

**Contract** — advance the acceleration state, commit any pending text edit, then add or subtract the current step, clamping at the range end. An arithmetic overflow is treated identically to hitting the range end.

## `BeginInit` / `EndInit`

**Contract** — brackets a period in which range and value may be set in any order without range checks. On close, the value is clamped into the (now complete) range and the text is rendered.

**Notes** — this exists because layout descriptions assign properties in an order nobody controls. Without the bracket, setting a value of 500 before raising the maximum from its default would be rejected. A rebuild with a constructor that takes the whole configuration at once does not need it.
