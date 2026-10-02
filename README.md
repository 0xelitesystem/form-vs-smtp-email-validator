# Form vs SMTP Email Validator

Paste email addresses and see whether the browser's type=email rule, RFC 5322 and RFC 5321 each accept them, which rule decided each verdict, and the quoted form SMTP needs when they disagree.

**Live demo:** https://0xelitesystem.github.io/form-vs-smtp-email-validator/

## Use

1. Paste addresses into the box, one per line, or press **Load the disagreements**. Each line is checked exactly as typed. Nothing is trimmed, so a stray space counts. Blank lines are skipped, and lines over 2000 characters are refused with a message.
2. Press **Check**. Every line gets five columns:
   - **A. Browser form rule.** The HTML Standard's own regular expression for a valid email address, run verbatim, with a plain explanation of the part that failed: no `@` or more than one, a local character outside the allowed set, an empty label, a label that starts or ends with a hyphen, a label over 63 characters, or a disallowed domain character.
   - **B. This browser, live.** The line is assigned to a real `input type=email` and the page reads back `validity.valid`, `validity.typeMismatch` and `.value`. If the browser's value sanitization changed the value, the new value is shown. The column header carries this browser's user agent string.
   - **C. RFC 5322 addr-spec (message format).** VALID, VALID ONLY WITH COMMENTS OR FOLDING WHITE SPACE, VALID ONLY UNDER OBSOLETE SYNTAX, or INVALID, with the rule that decided it.
   - **D. RFC 5321 Mailbox (SMTP envelope).** VALID or INVALID, including the checks the ABNF leaves to its comments: Snum 0 to 255, the IPv6 group limits, and the registered-tag rule for address literals.
   - **E. Translation.** The minimally quoted RFC 5321 form, and when the browser rule rejects an address, the spelling it would accept, if one exists.
3. Read the notes row under any address that has one: size limits, the single-label domain rule, local-part case sensitivity, the empty quoted local part, and DNS label length.
4. Section 2 is a plain `type=email` field you type into. It shows `.value`, `validity.typeMismatch` and `validity.badInput` as this browser reports them.

### Real uses

- **A signup form refused a real address.** If column A rejects and column D accepts, the address works over SMTP and the form rule is what stopped it. Column E shows the spelling the form would accept, when there is one.
- **A form accepted an address your mail server will not send to.** `.john@example.com`, `a..b@example.com` and `john.@example.com` pass the browser rule and fail both RFCs. Column E gives `".john"@example.com`, a valid RFC 5321 Mailbox.
- **Picking the check for each layer.** A web form, a message header and an SMTP envelope use three different grammars. Paste your test addresses and see which layer each one breaks.
- **Writing validator tests.** Press **Load every self test fixture**, then copy any row that surprises you into your own suite with the deciding rule next to it.
- **Seeing what your own browser does.** Column B and section 2 are measured in your browser when you load the page, not quoted from a table.

It checks syntax only. There is no DNS lookup, no MX lookup and no network call, and it cannot tell whether a mailbox exists or accepts mail.

Out of scope, because each line is one address: comma separated lists (`type=email multiple` and RFC 5322 `address-list`), display names with angle brackets, and header folding across lines.

## Why this exists

The browser's email rule and the email RFCs disagree in both directions, and the disagreement is deliberate. The HTML Standard says of its own definition: "This requirement is a willful violation of RFC 5322, which defines a syntax for email addresses that is simultaneously too strict (before the "@" character), too vague (after the "@" character), and too lax (allowing comments, whitespace characters, and quoted strings in manners unfamiliar to most users) to be of practical use here."

So a form can accept `.john@example.com`, which no RFC grammar accepts, and reject `"john doe"@example.com` and `user@[192.0.2.1]`, which both RFC grammars accept. RFC 5322 section 3.4.1 itself says implementations must "conform to the syntax of addresses for the context in which they are used", and there are three contexts here, not one. RFC 5322 is the message format grammar. RFC 5321 is the SMTP envelope grammar. The "smtp" in this repo's name means RFC 5321.

Three things decide most of the confusing cases, and the page states each one where it applies:

- **Sizes are not syntax.** RFC 5321 section 4.5.3.1 gives 64 octets for the local part, 255 for the domain and 256 for the path, and says "Every implementation MUST be able to receive objects of at least these sizes. Objects larger than these sizes SHOULD be avoided when possible." An oversize address is reported as exceeding the size every server must accept, never as invalid. The figure 254, often quoted, comes from RFC 3696 erratum 1690 (status Verified), which derives it from the 256 octet path minus the two angle brackets.
- **Minimum quoting.** RFC 5321 section 4.1.2: "all quoted forms MUST be treated as equivalent, and the sending system SHOULD transmit the form that uses the minimum quoting possible". Column E applies that rule.
- **Semantics are not grammar.** `user@localhost` is valid in all three grammars. The only objection is RFC 5321 section 2.3.5, "Only resolvable, fully-qualified domain names (FQDNs) are permitted", so it gets a note, not a failure.

The RFC 5322 column is a recursive descent parser rather than one big regular expression. Comments nest without limit, and the page has a self test that feeds pathological 2000 character lines through every engine to show nothing hangs.

The page tests itself when it loads: fixtures derived by hand from the ABNF, the column A explainer fuzzed against the verbatim regex, the RFC 5321 parser fuzzed against a separately built regular expression, the RFC 5322 parser fuzzed against separately built regular expressions with comments nested at most three deep, and a check designed to fail so a self test that silently stops running shows up. Every count it prints is computed in your browser on that run.

It is one HTML file, MIT licensed, with no network call of any kind.

### Facts the page relies on, and where they were checked

All re-fetched on 2026-10-01.

- The regular expression and the "willful violation" sentence: HTML Standard, Email state (`type=email`), https://html.spec.whatwg.org/multipage/input.html#valid-e-mail-address
- The single-address value sanitization, "Strip newlines from the value, then strip leading and trailing ASCII whitespace from the value": same page.
- RFC 5322 (October 2008, updated by RFC 6854) sections 3.2.1 to 3.2.4, 3.4.1, 4, 4.1 and 4.4: https://www.rfc-editor.org/rfc/rfc5322.txt and https://www.rfc-editor.org/info/rfc5322
- RFC 5321 (October 2008, updated by RFC 7504) sections 2.3.5, 2.4, 4.1.2, 4.1.3 and 4.5.3.1.1 to 4.5.3.1.3: https://www.rfc-editor.org/rfc/rfc5321.txt and https://www.rfc-editor.org/info/rfc5321
- RFC 3696 erratum 1690, Verified, reported 2009-02-22, verified 2010-04-03: https://www.rfc-editor.org/errata/eid1690
- RFC 5321 erratum 5414, Held for Document Update, reported 2018-06-29, proposing `1*QcontentSMTP`: https://www.rfc-editor.org/errata/eid5414
- IANA Address Literal Tags, which lists only `IPv6`: https://www.iana.org/assignments/address-literal-tags/address-literal-tags.txt
- RFC 1035 section 2.3.4, "labels 63 octets or less": https://www.rfc-editor.org/rfc/rfc1035.txt
- RFC 5234 section 2.3, "ABNF strings are case insensitive": https://www.rfc-editor.org/rfc/rfc5234.txt
- RFC 6531 section 3.3 and RFC 6532 section 3.2, the UTF-8 extensions: https://www.rfc-editor.org/rfc/rfc6531.txt and https://www.rfc-editor.org/rfc/rfc6532.txt

This tool is independent and is not affiliated with or endorsed by the WHATWG, the IETF, the RFC Editor, IANA or any browser vendor.

## Privacy

Everything runs in your browser. The addresses you paste are never uploaded, logged or stored. There is no analytics, no telemetry, no font fetch, no CDN and no network call of any kind in this file, and its Content-Security-Policy sets `connect-src 'none'`. The only thing written to disk is your light or dark theme choice in `localStorage`. Open the file, disconnect from the network, and it still works.

## Run locally

```
git clone https://github.com/0xelitesystem/form-vs-smtp-email-validator.git
cd form-vs-smtp-email-validator
```

Open `index.html` in any modern browser. It works straight from `file://`. Or serve it:

```
python -m http.server 8000
```

Then visit http://localhost:8000/

## Build

There is no build. One `index.html` with inline CSS and inline JavaScript, no dependencies, no package manager, no toolchain.

## License

MIT. Copyright (c) 2026 0xelitesystem. See [LICENSE](LICENSE).

## Third-party notices

**HTML Standard regular expression.** Column A uses the regular expression from the HTML Standard section "Email state (`type=email`)", copied verbatim, with a one-line source comment above it in `index.html`. Source: https://html.spec.whatwg.org/multipage/input.html#valid-e-mail-address. The HTML Standard's acknowledgements page (https://html.spec.whatwg.org/multipage/acknowledgements.html) reads: "Copyright © WHATWG (Apple, Google, Mozilla, Microsoft). This work is licensed under a Creative Commons Attribution 4.0 International License. To the extent portions of it are incorporated into source code, such portions in the source code are licensed under the BSD 3-Clause License instead." The regular expression is therefore used under the BSD 3-Clause License:

```
Copyright © WHATWG (Apple, Google, Mozilla, Microsoft).

Redistribution and use in source and binary forms, with or without
modification, are permitted provided that the following conditions are met:

1. Redistributions of source code must retain the above copyright notice,
   this list of conditions and the following disclaimer.

2. Redistributions in binary form must reproduce the above copyright notice,
   this list of conditions and the following disclaimer in the documentation
   and/or other materials provided with the distribution.

3. Neither the name of the copyright holder nor the names of its
   contributors may be used to endorse or promote products derived from
   this software without specific prior written permission.

THIS SOFTWARE IS PROVIDED BY THE COPYRIGHT HOLDERS AND CONTRIBUTORS "AS IS"
AND ANY EXPRESS OR IMPLIED WARRANTIES, INCLUDING, BUT NOT LIMITED TO, THE
IMPLIED WARRANTIES OF MERCHANTABILITY AND FITNESS FOR A PARTICULAR PURPOSE ARE
DISCLAIMED. IN NO EVENT SHALL THE COPYRIGHT HOLDER OR CONTRIBUTORS BE LIABLE
FOR ANY DIRECT, INDIRECT, INCIDENTAL, SPECIAL, EXEMPLARY, OR CONSEQUENTIAL
DAMAGES (INCLUDING, BUT NOT LIMITED TO, PROCUREMENT OF SUBSTITUTE GOODS OR
SERVICES; LOSS OF USE, DATA, OR PROFITS; OR BUSINESS INTERRUPTION) HOWEVER
CAUSED AND ON ANY THEORY OF LIABILITY, WHETHER IN CONTRACT, STRICT LIABILITY,
OR TORT (INCLUDING NEGLIGENCE OR OTHERWISE) ARISING IN ANY WAY OUT OF THE USE
OF THIS SOFTWARE, EVEN IF ADVISED OF THE POSSIBILITY OF SUCH DAMAGE.
```

**HTML Standard quotation.** The "willful violation" sentence, the value sanitization sentence and the punycode sentence quoted on the page and in this README are from the same section of the HTML Standard, Copyright © WHATWG (Apple, Google, Mozilla, Microsoft), licensed under the Creative Commons Attribution 4.0 International License (https://creativecommons.org/licenses/by/4.0/). They are quoted unchanged.

**RFC 5321 and RFC 5322.** The ABNF rule lines shown on the page are short citations, Copyright (C) The IETF Trust (2008). The RFC 5321 and RFC 5322 parsers are reimplemented from the published ABNF; no code is reproduced.

## Related

- [email-header-analyzer](https://github.com/0xelitesystem/email-header-analyzer) - read the headers of a message once it has been delivered.
- [regex-tester-with-explainer](https://github.com/0xelitesystem/regex-tester-with-explainer) - test and explain your own regular expressions.
- [unicode-character-inspector](https://github.com/0xelitesystem/unicode-character-inspector) - see exactly which code points a non-ASCII address contains.
