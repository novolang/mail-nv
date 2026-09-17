# mail-nv

An electronic mail message is a header section and a body, specified in
[RFC 5322](https://www.rfc-editor.org/rfc/rfc5322). MIME —
[RFC 2045](https://www.rfc-editor.org/rfc/rfc2045),
[RFC 2046](https://www.rfc-editor.org/rfc/rfc2046),
[RFC 2047](https://www.rfc-editor.org/rfc/rfc2047) and
[RFC 2231](https://www.rfc-editor.org/rfc/rfc2231) — turns that body
into a tree of parts and lets a header carry text that is not US-ASCII.
This package reads a message into typed headers and a tree of parts, and
builds one back. It opens no socket.
[smtp-nv](https://novo-lang.org/packages/smtp-nv) sends what this package
builds, and [dkim-nv](https://novo-lang.org/packages/dkim-nv) signs it.

**Status: NOT IMPLEMENTED — interface only.** Every function is declared
with its full signature, but every body is a `todo()` that panics when
called. The package is published so its design can be reviewed and
depended on before it is implemented. Version 0.1.0 will be the first
working release.

## What a message is

A message is a **header section**, a blank line, and a **body**. The
header section is a sequence of **fields**, each a name, a colon and a
body (RFC 5322 section 2.2). Fields keep the order they arrived in, and
some of them appear more than once: a relay adds a `Received:` field at
the top for each hop it makes, so the newest is first.

A field body may be broken across several lines by inserting a line
break before a space or a tab. That is called **folding** (section
2.2.3). The line break is not part of the value, and removing it is
called unfolding. It must happen before anything reads the field.

Four field bodies have a grammar of their own.

| Field | Grammar | Section |
| --- | --- | --- |
| `From:`, `To:`, `Cc:`, `Bcc:`, `Reply-To:`, `Sender:` | addresses | RFC 5322 section 3.4 |
| `Date:` | a date, a time and an offset from UTC | RFC 5322 section 3.3 |
| `Message-ID:`, `In-Reply-To:`, `References:` | message identifiers | RFC 5322 section 3.6.4 |
| `Content-Type:`, `Content-Disposition:`, `Content-Transfer-Encoding:` | a media type and parameters | RFC 2045 section 5.1 |

An **address** is one of two shapes. A **mailbox** is an optional
display name and an **address specification** — the `local@domain` form
— written either as `ada@example.com` or as
`Ada Lovelace <ada@example.com>`. A **group** is a name, a colon, a list
of mailboxes and a semicolon. A group with no members is legal:
`undisclosed-recipients:;` is how a `To:` field says the recipients are
not shown.

MIME adds two things. First, a **tree**: a part whose `Content-Type` is
`multipart/something` has a **boundary** parameter, and its body is the
parts between the lines `--boundary` and `--boundary--` (RFC 2046
section 5.1.1). Each of those has its own header section and its own
body, and may itself be a multipart. Second, a **content transfer
encoding**: because a mail path may only be trusted with lines of at
most 998 octets of US-ASCII, a body is armoured as `quoted-printable`
(readable, for text that is mostly ASCII) or as `base64` (a third
larger, for binary).

A header field may only hold US-ASCII, so non-ASCII text in one is
written as an **encoded word**, `=?charset?encoding?text?=` (RFC 2047
section 2). A `Content-Type` parameter may not hold an encoded word, so
a long or non-ASCII parameter is split into numbered **sections**
instead (RFC 2231 section 3).

Every function in this package performs no input or output. A part's
body is a **span** — an offset and a length into the octets the message
was parsed from — so parsing a large message with attachments copies
none of them.

## Install

```
novo pkg add mail-nv
```

## Example

```novo
use std.bytes
use mailmsg
use mailmime

fn main() [io]
    // The octets of a message, as they came off a file or a socket.
    let octets = bytes.from_str("From: Ada <ada@example.com>\r\nSubject: Hello\r\n\r\nhi")

    // Read it. 32 bounds how deeply parts may nest.
    match mailmsg.parse(octets, 32)
        Err(e) => println("not a message: ${e.message()}")
        Ok(m)  =>
            // The subject with its encoded words already decoded.
            println(mailmsg.subject_of(m))

            // The authors. There may be more than one.
            match mailmsg.from_of(m)
                Err(e) => println("bad From: ${e.message()}")
                Ok(fs) => println("from ${fs[0].local_part}@${fs[0].domain}")

            // The text a reader would show.
            match mailmsg.text_of(m)
                Err(e) => println("cannot render: ${e.message()}")
                Ok(t)  => println(t)

            // Each attachment, by the name the sender gave it.
            for part in mailmime.attachments_of(m.root)
                println("attachment: ${mailmime.filename_of(part)}")
```

Build and test with `novo pkg build` and `novo test`. Today `novo test`
fails on purpose: every test reaches a
`not implemented: mail-nv.<module>.<fn>` panic. The tests are the
specification the implementation will have to satisfy.

## What the package contains

| Module | Contents |
| --- | --- |
| `mailerror` | Every reason a message can be refused, the octet it was refused at, and the two questions a caller asks about one. |
| `mailaddr` | Mailboxes, groups and address specifications: parsing, writing, and the comparison that decides whether two are the same mailbox. |
| `mailword` | Encoded words (RFC 2047) and parameter continuations (RFC 2231), both directions, and the three character sets this package converts. |
| `mailctype` | `Content-Type`, `Content-Disposition` and `Content-Transfer-Encoding` as values, with their parameters. |
| `mailhdr` | The header section: fields in order, folding and unfolding, dates, and message identifiers. |
| `mailxfer` | Spans into a message's octets, and the quoted-printable and base64 encodings both ways. |
| `mailmime` | The body as a tree of parts, with the preamble, the epilogue and the rule for choosing one arm of an alternative. |
| `mailmsg` | A whole message read, and a whole message built. |

## How to choose an entry point

**`mailmsg.parse` is the way in for a message that arrived.** It gives
a `MailMessage`: the octets, the header section, and the tree. Every
accessor on it — `subject_of`, `from_of`, `text_of` — answers a question
about what was sent.

**`mailmsg.draft` is the way in for a message being written.** Set the
fields with the `with_` functions and call `build`, which returns the
octets. A draft has no spans and a parsed message has no builder state,
which is why they are two types.

**`mailmime.parse_part` is for a part that is not a whole message.** A
`multipart/form-data` body from an HTTP request is the usual case: it
has MIME structure and no mail headers.

**`mailhdr` and `mailaddr` are for a caller that has one field.** A
milter reading `Received:` chains, or a mailing list checking a `From:`,
needs the grammar and not the message.

**`mailxfer` is for a caller that has encoded octets and nothing else.**
`decode_base64` and `decode_quoted_printable` take a span and answer
bytes.

## The rules a user needs

1. **A span is only valid against the octets it was cut from.** A part
   holds an offset and a length into `MailMessage.source`, not a copy.
   `mailmsg.part_bytes` pairs the two for you. Passing a part to
   `mailmime.decoded_body` with different octets answers
   `MailSpanOutOfRange` rather than wrong data.
2. **Unfold a field before reading it.** `mailhdr.parse_headers` does it
   for you and `MailField.value` is already unfolded. A value read
   straight out of raw octets stops at the first line break (RFC 5322
   section 2.2.3).
3. **A header field can appear more than once.** `value_of` answers the
   first. `field_count` is how a program refuses a message with two
   `From:` fields, which RFC 5322 section 3.6 forbids and which is how a
   filter is evaded when one parser reads the first and another reads
   the second.
4. **The domain of an address is compared without case, and the local
   part is compared with it.** RFC 5321 section 2.4 leaves the meaning
   of the local part to the receiving host. `mailaddr.same_mailbox`
   applies both rules.
5. **`-0000` in a `Date:` field does not mean UTC.** RFC 5322 section
   3.3 says it means the time was generated where the zone was unknown.
   `MailDateTime.offset_known` is false for it and true for `+0000`.
6. **There is no clock and no identifier generator here.** A draft's
   `Date:` and `Message-ID:` are values the caller supplies. A draft
   with neither is built without those fields.
7. **The whitespace between two adjacent encoded words disappears.**
   RFC 2047 section 6.2 requires it, and it is what lets long non-ASCII
   text be split across several words without gaining spaces.
8. **Text that is already ASCII is never wrapped in an encoded word.**
   `mailword.needs_encoding` is what a builder asks first.
9. **A `multipart/alternative` shows one arm: the last one understood.**
   RFC 2046 section 5.1.4 orders the arms plainest first.
   `mailmime.display_parts` applies the rule and `walk_order` ignores it.
10. **A part with an unknown `Content-Disposition` is an attachment.**
    RFC 2183 section 2.8 requires that reading.
11. **A `filename` parameter is the sender's text, unsanitised.** A
    sender may write `../../etc/passwd`. This package has no file system
    and cannot know what is safe where you are, so the caller sanitises
    it.
12. **Nesting is bounded by a parameter, not by a constant.**
    `mailmsg.parse` and `mailmime.parse_part` take `max_depth`; pass 32
    unless you have a reason not to. `MailTooDeep` is what exceeding it
    answers, which is how a message designed to exhaust memory is
    refused.
13. **`Bcc:` is written by `build_with_bcc` and not by `build`.** The
    copy for the wire leaves it out, because a `Bcc:` field that reaches
    a recipient is a disclosure.
14. **A message with several `From:` mailboxes needs a `Sender:`.** RFC
    5322 section 3.6.2 requires it, and `build` refuses a draft that
    breaks the rule.
15. **A body is canonical CRLF before it is encoded.** RFC 2045 section
    6.7 requires the conversion. `mailxfer.to_crlf` applies it and
    `to_lf` reverses it. Do not reverse it on a body whose signature is
    being checked: the signature covers the CRLF form.
16. **`mailmsg.raw_header_span` and `raw_body_span` give the octets as
    they arrived.** A rebuild from the parsed fields normalises folding
    and case and hashes to something else, so a signature check uses the
    spans.
17. **Three character sets are converted: `us-ascii`, `iso-8859-1` and
    `utf-8`.** Anything else is `MailUnknownCharset`, which
    `mailerror.is_unrenderable` reports as a message that arrived intact
    and cannot be shown. The octets are still reachable through
    `mailmime.decoded_body`.
18. **A boundary is chosen from the content and a seed.** RFC 2046
    section 5.1.1 requires a boundary that does not occur in the parts.
    This package has no source of randomness, so
    `mailmime.fresh_boundary` takes a seed and a caller that wants an
    unpredictable boundary supplies one from its own.

## What is not included

- **A socket.** Nothing here sends or receives.
  [smtp-nv](https://novo-lang.org/packages/smtp-nv) sends a message and
  is a separate package.
- **A signature.** Signing and verifying a message is
  [dkim-nv](https://novo-lang.org/packages/dkim-nv)'s subject. This
  package gives it the two spans it needs and depends on neither it nor
  smtp-nv.
- **A clock and a source of randomness.** See rules 6 and 18.
- **Character sets beyond the three named in rule 17.** A full
  conversion table is a different package.
- **A mailbox format.** mbox and Maildir are file formats, and reading
  a file costs an effect this package does not have. A caller reads a
  message's octets out of one and hands them here.
- **IMAP and POP3.** Those are protocols for fetching messages, not a
  grammar for one.
- **An HTML renderer or sanitiser.** `mailmime.html_part` gives you the
  part; what is safe to show is the caller's decision.
- **S/MIME and OpenPGP.** Encrypted and signed message formats layered
  on top of MIME, each a specification of its own.
- **`message/partial` reassembly.** RFC 2046 section 5.2.2's split
  messages need several messages and a store to hold them, which is a
  caller's job.

## Related packages

- [smtp-nv](https://novo-lang.org/packages/smtp-nv) is the protocol that
  carries a message to a server. Take it to send. Take this package to
  make the thing it sends.
- [dkim-nv](https://novo-lang.org/packages/dkim-nv) signs and verifies a
  message's headers and body. It reads what this package parses.
- [mime-nv](https://novo-lang.org/packages/mime-nv) maps a file
  extension to a media type. This package parses the `Content-Type`
  field itself.
- [calendar-nv](https://novo-lang.org/packages/calendar-nv) supplies the
  date and time types a `Date:` field is read into. This package depends
  on it.
- [base64-nv](https://novo-lang.org/packages/base64-nv) is base64 on its
  own. The variant here is RFC 2045's, which wraps at 76 characters and
  skips line breaks on the way in.

## Tests

```bash
novo test tests/mailaddr_tests.nv    # the address grammar
novo test tests/mailhdr_tests.nv     # folding, field order, dates, identifiers
novo test tests/mailmime_tests.nv    # media types, transfer encodings, the tree
novo test tests/mailmsg_tests.nv     # a message read, and a message built
novo test tests/mailerror_tests.nv   # the refusals
```

The normative sources are RFC 5322 for the message, RFC 2045 and RFC
2046 for MIME, RFC 2047 for encoded words, RFC 2231 for parameter
continuations, RFC 2183 for `Content-Disposition` and RFC 5321 section
2.4 for address comparison. The vectors are the examples those documents
print: RFC 5322 appendix A.1.1's message, appendix A.1.3's group, RFC
2046 section 5.1.1's multipart, and RFC 2047 section 8's encoded words —
including the three that establish that `=?ISO-8859-1?Q?a?=
=?ISO-8859-1?Q?b?=` decodes to `ab` with no space. The reference
implementations are the Rust crates `mail-parser` and `lettre`, and
Python's `email.message`.

The suite asserts that a folded field unfolds to one value, that two
`Received:` fields are both reachable, that `-0000` does not assert UTC,
that the space between adjacent encoded words is dropped, that a
`multipart` with no closing boundary is refused as truncated, that a
draft built here parses back to the same addresses and attachment
octets, and that `build` leaves `Bcc:` out.

The tests compile today and fail at run, each on the
`not implemented: mail-nv.<module>.<fn>` panic that is its body. That is
the expected state of an interface release. They turn green one at a
time as bodies land.

## Implementation status

| Item | Implemented |
| --- | --- |
| `MailError` and its twenty reasons, `mailerror.offset_of`, `.code`, `.is_unrenderable`, `.is_truncated` | the type is declared |
| `mailaddr.parse_address`, `.parse_entries`, `.mailboxes_of`, `.is_valid_addr_spec`, `.same_mailbox` | no |
| `mailaddr.addr_spec`, `.named_address`, `.group`, `.format_addr_spec`, `.format_address`, `.format_entries` | no |
| `mailword.parse_word`, `.decode_text`, `.decode_phrase`, `.needs_encoding`, `.encode_text`, `.encode_word` | no |
| `mailword.charset_named`, `.charset_name`, `.encoding_name`, `.assemble_parameter`, `.split_parameter` | no |
| `mailctype.parse_content_type`, `.format_content_type`, `.param_value`, `.with_param`, `.charset_of`, `.boundary_of` | no |
| `mailctype.is_multipart`, `.is_message`, `.default_content_type`, `.content_type`, `.param` | no |
| `mailctype.parse_disposition`, `.format_disposition`, `.disposition`, `.disposition_name`, `.disposition_is_attachment`, `.filename_of` | no |
| `mailctype.parse_transfer_encoding`, `.transfer_encoding_name` | no |
| `mailhdr.parse_headers`, `.value_of`, `.values_of`, `.field_count`, `.append`, `.prepend`, `.without`, `.headers`, `.field` | no |
| `mailhdr.unfold`, `.fold`, `.format_headers` | no |
| `mailhdr.parse_date`, `.format_date`, `.datetime` | no |
| `mailhdr.message_id`, `.parse_message_id`, `.parse_message_ids`, `.format_message_id`, `.addresses_of` | no |
| `mailxfer.span`, `.slice`, `.decode`, `.encode`, `.choose_encoding`, `.to_crlf`, `.to_lf` | no |
| `mailxfer.decode_base64`, `.encode_base64`, `.decode_quoted_printable`, `.encode_quoted_printable` | no |
| `mailmime.parse_part`, `.format_part`, `.leaf_part`, `.multipart_part`, `.fresh_boundary` | no |
| `mailmime.decoded_body`, `.decoded_text`, `.walk_order`, `.display_parts`, `.text_part`, `.html_part` | no |
| `mailmime.attachments_of`, `.filename_of`, `.part_by_content_id` | no |
| `mailmsg.parse`, `.subject_of`, `.from_of`, `.recipients_of`, `.date_of`, `.message_id_of`, `.thread_parents_of`, `.text_of`, `.part_bytes` | no |
| `mailmsg.raw_header_span`, `.raw_body_span` | no |
| `mailmsg.draft` and the fourteen `with_` builders, `.in_reply_to`, `.attachment`, `.inline_attachment` | no |
| `mailmsg.build`, `.build_with_bcc` | no |

## Licence

Apache-2.0. See `LICENSE`.

<!-- docs/writing-a-readme.md is the style guide for this page. -->
