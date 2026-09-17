# Changelog

All notable changes to mail-nv are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

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
