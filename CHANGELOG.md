# Changelog

All notable changes to mail-nv are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

## Unreleased

### Changed

- Internal: ladders rewritten as `match` with character literals, the date token kind is an enum, and errors are propagated with `!`. No public signature changed.

### Fixed

- `mailxfer.slice` refuses a span that reaches past the source, for a start and a length of any size.

## 0.1.0 — 2026-10-13

**The implementation, over the 0.0.2 interface.** Every public function
declared in the interface release has a body; the signatures, the
effect rows and the doc comments are unchanged. The version moves off
`0.0.x`, which is where an interface release lives, and the stability
moves from `draft` to `experimental`.

### Added

- Every body in all eight modules. `novo test tests` runs eight
  suites, 143 cases, green: the five interface suites plus the three
  landing suites below. `novo pkg publish` reports 99% of 2036
  instrumented lines under `src/` covered, with the ten remaining
  regions excused by `// cov: skip` and the reason on each.
- `tests/mail_nv_edge_tests.nv` — the four rules the rest of the
  grammar leans on, pinned from raw octets rather than from values the
  package made itself: a folded subject and a folded address list; a
  comma inside a quoted display name and a parenthesised comment,
  neither of which is a separator or a name; a `Date:` with a named
  zone and one with a two-digit year; and a message with no body at
  all, beside one whose body's first line looks like a header field.
- `tests/mail_nv_gap_tests.nv` — the corner of every module that the
  interface suites reach past: the obsolete zone table and the military
  letters, the RFC 2231 sections a long parameter is cut into, the
  transfer encodings' refusals, the multipart tree's span arithmetic,
  and the folding every field goes through.
- `tests/mail_nv_cover_tests.nv` — the publish coverage residual: the
  refusal each parser has for a shape the interface suites do not
  produce (a bad angle address naming its offset, an unclosed group, a
  part whose headers overrun its content, a word whose marker is
  malformed), the paths a builder takes for an empty name or an empty
  value, and the round trips that keep them honest.

### Fixed

- `mailword.percent_encode` wrote `=XX` where RFC 2231 § 4 requires
  `%XX`. A `boundary*0*` parameter written by the builders came back as
  a boundary containing `=3D`, so a message this package built could
  not be parsed back by this package. Caught by the round-trip case in
  `tests/mailmsg_tests.nv`.
- `mailword.encode_q_bytes` and `mailxfer.encode_quoted_printable`
  wrote `%{...}` where the interpolation syntax of the language is
  `${...}`, so those code paths emitted the literal text `%{` and the
  hexadecimal digits were never substituted.
- `mailword.utf8_to_latin1` substituted one `?` for a four-octet
  character and advanced by one octet, so the continuation octets were
  read again as leads and the word came out three octets long with a
  byte packed from a negative integer (`\u{1f600}` encoded as
  `=?iso-8859-1?B?P9g/?=` where the rule is one `?`). Filed as
  `novo/bugs/diagnostics/closed/mail-nv-utf8-to-latin1-re-reads-a-four-octet-character-s-continuation-octets`.
- `mailword.word_end` answered at the `?` that closes an encoded
  word's charset instead of at the `?=` that closes the word, so every
  well-formed word was cut to `=?charset?` and decoded to its own
  literal text — and a word whose text was really malformed became
  indistinguishable from one this package had cut short itself. Filed
  as
  `novo/bugs/diagnostics/closed/mail-nv-word-end-answers-at-the-charset-s-question-mark`.
- `mailhdr.zone_offset` gave the military letters `K`, `L` and `M` an
  hour too many (`(letter - 96)` where RFC 5322 § 4.3's skipped `J`
  makes `K` the tenth hour), so such a date was placed an hour early.
  Filed as
  `novo/bugs/diagnostics/closed/mail-nv-zone-offset-gives-the-military-letters-k-l-and-m-an-hour-too-many`.
- `mailxfer.encode_quoted_printable` wrote the soft line break as a
  bare `=`, not RFC 2045 § 6.7 rule 5's `=` and CRLF, so a wrapped
  body decoded one octet short — through this package's own decoder as
  well as anyone else's. Filed as
  `novo/bugs/diagnostics/closed/mail-nv-encode-quoted-printable-s-soft-break-is-a-bare-equals-sign`.
- `mailaddr.parse_entries` accepted a group whose `;` never arrived
  (`"Friends: x@y.z, "` parsed as a complete field), where RFC 5322
  § 3.4 ends a group with the semicolon. Filed as
  `novo/bugs/diagnostics/closed/mail-nv-parse-entries-accepts-a-group-whose-semicolon-never-arrives`.

### Known

- **The obsolete zone names are read and never written.** RFC 5322
  § 4.3's alphabetic zones and military letters are accepted and
  converted to an offset, and `format_date` always writes the numeric
  form § 3.3 specifies. A round trip through this package therefore
  does not preserve `GMT` as written, which is the intent and is pinned
  by `tests/mail_nv_edge_tests.nv`.
- **A two-digit year is windowed at 50.** `< 50` reads as `20xx` and
  `>= 50` as `19xx`, which is every implementation's rule and is a
  guess about a year that was never written down.
- **Three character sets are converted**: `us-ascii`, `iso-8859-1` and
  `utf-8`. Everything else is `MailUnknownCharset`, and
  `mailerror.is_unrenderable` says it is a rendering limit rather than
  a malformed message.
- **`MailError` carries eighteen reasons**, which the 0.0.1 and 0.0.2
  entries described as twenty. The type has not changed; the count in
  that prose was wrong.
- **No dependency on smtp-nv or dkim-nv**, as the interface release
  said. Both depend on this package rather than the other way round.
- **The effect row never widened.** Every function fits `core`'s empty
  budget, and `novo`'s floor moves to `>= 0.20.2`, the toolchain the
  bodies were verified against.

### Workarounds recorded here

- **An early return leaves a call-bound local live, and the local is
  what the tail moves, so the reference is stranded**
  (`memory-perceus/an-early-return-skips-the-release-of-a-call-bound-local-the-tail-moves`,
  closed, `fixed_in: 0.20.3`). `mailctype.parse_content_type` answered
  `Ok` with two `str.slice` results built in the literal the tail
  moves; under novo 0.20.2 the refusals above them stranded the
  intermediate and the suite exited 23 with every verdict green. The
  two names are now cut out of `value` only after the last refusal, so
  every path that returns early does so before the first call whose
  result the tail moves; the comment at the site in
  `src/mailctype.nv` names this filing. 0.20.3 is **not** the toolchain
  here — the manifest's floor is `>= 0.20.2`, which still strands it,
  so the shape stays. **Revert when the floor moves past 0.20.2 and the
  leak no longer reproduces.**

## 0.0.2 — 2026-09-28

The dependency ranges move to the dependencies' current releases.  A
pre-1.0 caret range admits only the release it names, so the old
ranges held this package on interface releases, and a program could
not take this package beside those packages' current releases.  No
signature in this package changed.

- calendar-nv: `^0.0.2` to `^0.2.0`.

## [0.0.1] — 2026-09-17

**The interface, published before anyone implements it.** Every public
type and function carries its full signature, its effect row and its
doc comment; every body is `todo()`; the release is recorded
`implemented = false`.

### Added

- `mailxfer` — the load-bearing interface. A part's body is a
  `MailSpan`, an offset and a length into the octets the message was
  parsed from, and never a copy. Parsing a message with four
  attachments allocates the tree and nothing else; a caller that wants
  one attachment calls `decoded_body` and pays for that one. The cost
  is stated rather than hidden: a span is meaningful only against the
  bytes it was cut from, every decoder takes those bytes as its first
  argument, and `MailSpanOutOfRange` is what a stale span answers
  instead of short or wrong data. The module also holds the
  quoted-printable and base64 codecs of RFC 2045 § 6.7 and § 6.8, and
  the CRLF canonicalisation § 6.7 requires before a text body is
  encoded.
- `mailmsg` — two types for the two directions. `MailMessage` is a view
  over octets that arrived; `MailDraft` is a value being built and has
  no spans at all. Nothing asks a draft what offset its `From:` was at
  and nothing asks a parsed message to add an attachment, so one type
  would carry a span table nobody writes beside a builder state nobody
  reads. `build` writes the copy for the wire and `build_with_bcc`
  writes the copy for the sender's own folder, named rather than
  flagged so a reviewer can see which a call site produces.
- `mailhdr` — the header section as a LIST of fields, not a map. The
  order is what a signature is computed over, `Received:` is one field
  per hop with the newest first, and a message with two `From:` fields
  is malformed in a way a caller should be able to SEE rather than have
  one of them silently dropped. `field_count` is the function that
  makes refusing it possible. `MailDateTime` is calendar-nv's
  `CivilDateTime` plus an offset in minutes, with `offset_known` false
  for `-0000`, which RFC 5322 § 3.3 says means the zone was unknown and
  which is a different fact from UTC.
- `mailmime` — the body as a tree, with the preamble and the epilogue
  kept as spans although RFC 2046 § 5.1.1 says a reader ignores them,
  because ignoring something is the caller's decision. `display_parts`
  applies § 5.1.4's rule for `multipart/alternative` — the last arm
  understood — and `walk_order` gives every part to a caller that is
  indexing rather than showing. Nesting is bounded by a `max_depth`
  parameter, so a `message/rfc822` bomb is `MailTooDeep` and not an
  exhausted machine.
- `mailaddr` — mailboxes and groups as two types, because RFC 5322's
  grammar is not recursive and a type that said it was would have an
  unreachable case. `same_mailbox` applies RFC 5321 § 2.4 exactly: the
  domain without case, the local part with it.
- `mailword` — encoded words (RFC 2047) and parameter continuations
  (RFC 2231), which are the two different mechanisms for the same
  problem and are needed in different places. `decode_text` and
  `decode_phrase` are separate because an encoded word inside a quoted
  string is literal text, and a decoder that did not know its context
  would rewrite a display name that legitimately contained `=?`.
- `mailctype` — `Content-Type`, `Content-Disposition` and
  `Content-Transfer-Encoding` as values, with the parameters kept in
  the order they were written for the same reason the fields are.
- `mailerror` — twenty reasons, each carrying the octet it stopped at
  where there is one. `is_unrenderable` divides a message that arrived
  intact and cannot be shown from one that is broken, which is the
  difference between keeping it and quarantining it; `is_truncated`
  divides "wait for more bytes" from "reject what you have".

### Known

- `novo test` is red, and that is the release's expected state: every
  assertion in the five API suites reaches `not implemented:
  mail-nv.<module>.<fn>`.
- **Three character sets are converted**: `us-ascii`, `iso-8859-1` and
  `utf-8`. Everything else is `MailUnknownCharset`, and
  `mailerror.is_unrenderable` says it is a rendering limit rather than a
  malformed message. A full conversion table is a different package's
  subject, and the README's "What is not included" names it.
- **No dependency on smtp-nv or dkim-nv**, although both are named in
  the README. A message is a message whether or not anybody sends or
  signs it, and both of those packages depend on this one rather than
  the other way round.
- **No effect row wanted to widen.** Every function fits `core`'s empty
  budget as written. The two places where a less careful design would
  have reached for one — a clock for `Date:` and randomness for a
  multipart boundary — take a value and a seed instead, and both are
  stated as rules in the README rather than left for the implementation
  to discover.
