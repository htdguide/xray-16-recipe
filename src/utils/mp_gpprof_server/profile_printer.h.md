# src/utils/mp_gpprof_server/profile_printer.h

> Renders a profile as the plain-text key/value body the web response carries.

**Needs** — [`profile_data_types.h`](profile_data_types.h.md)

**Used by** — [`entry_point.cpp`](entry_point.cpp.md) · [`profile_request.cpp`](profile_request.cpp.md)

**Tier floor** — T3: text formatting.

## Purpose

Defines the **response body format** — the one part of this tool that a consumer outside
the repository actually sees, and therefore the part worth writing down precisely. It is
here rather than in a source file because the original wanted it inlinable; that is
incidental, and the format is not.

## The response body

A profile renders as a flat list of `name=value` lines, each terminated by a carriage
return and a line feed, in a fixed order: **every award's count and date, award by award,
in enumeration order, then every best score in enumeration order.** Sixty-seven lines,
always, whether or not the player ever earned anything.

```text
FUNCTION render(profile) -> text
  FOR EACH award IN award enumeration order
    name <- published_name(award)
    emit(name + "=" + profile.awards[award].count)
    emit(name + "_rdate=" + profile.awards[award].last_reward_date)
  FOR EACH score IN streak enumeration order
    emit(published_name(score) + "=" + profile.best_scores[score])
```

**Invariants**

- **An award contributes two lines and they are adjacent**: the count under the published
  name, and the date under that name with a fixed suffix. The suffix is the format; a
  consumer splits on it to pair the two.
- **Every line is always present**, zero-valued when nothing was earned. A consumer can
  therefore parse positionally and never has to handle an absent field.
- The line terminator is the two-character form, because the response travels over a
  protocol that specifies it.
- Values are rendered as decimal integers with no padding, no sign and no thousands
  separator. A date is published exactly as the service stored it — see
  [`profile_data_types.h`](profile_data_types.h.md) on the fact that its meaning is not
  recoverable.

**Notes**

- There is no header, no content type and no framing in the body itself; the response's
  own protocol supplies those. The body begins with a blank line separating it from the
  response headers, then the profile's name, then this list — see
  [`profile_request.cpp`](profile_request.cpp.md), which assembles that envelope.
- The renderer is defined over an abstract text sink so that it works against both the real
  output stream and a diagnostic one. That genericity is incidental; the format is the
  decision.
