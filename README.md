# Span Offset Debugger

Paste text and a span, name the system that produced the offsets and the one that reads them, and see the same numbers select four different substrings.

**Live demo:** https://0xelitesystem.github.io/span-offset-debugger/

The Language Server Protocol had to negotiate this. LSP 3.17 added a `positionEncoding` capability so that a client and a server can agree on what a character offset counts, because they were not agreeing. From the spec:

> Prior to 3.17 the offsets were always based on a UTF-16 string representation.

> Since 3.17 clients and servers can agree on a different string encoding representation (e.g. UTF-8).

> To stay backwards compatible the only mandatory encoding is UTF-16 represented via the string `utf-16`.

> Implementation considerations: since the conversion from one encoding into another requires the content of the file / line the conversion is best done where the file is read which is usually on the server side.

Source: Language Server Protocol Specification 3.17, Text Documents / Position. https://microsoft.github.io/language-server-protocol/specifications/lsp/3.17/specification/#textDocuments

Every major editor implements LSP. The protocol still needed a negotiated capability, a backward-compatibility clause and a note about where the conversion belongs, all for one integer. Your model, your tokenizer, your redactor and your highlighter negotiate nothing.

## Use

Open the page. It loads with a worked example already in the boxes.

1. Paste the source text.
2. Paste the spans, one per line. It reads `43,49`, `43 49`, `[43, 49]`, `{"start":43,"end":49}`, or a whole JSON array of any of those. A third field becomes a label.
3. Pick the **producer**: the convention the numbers are already in.
4. Pick the **consumer**: the convention of the system about to read them.

You get four things:

- **Four lanes.** The same text four times, with your raw numbers interpreted as UTF-16 code units, Unicode code points, UTF-8 bytes and grapheme clusters. Four bars, four substrings.
- **Drift per span.** The correct numbers in the consumer convention, the signed drift on each end, and the substring the consumer grabs if nobody converts.
- **Converted spans.** Copyable JSON in the consumer convention.
- **Silent shifts.** A BOM, CRLF, NFC versus NFD, an undecoded HTML entity or an unpaired surrogate in this text, reported with the offset where the shift begins and the measured length change in all four conventions.

There is a CRLF checkbox next to the source box, and it is not decoration. A textarea cannot hand a page a carriage return. The HTML standard: "The algorithm for obtaining the element's API value is to return the element's raw value, with newlines normalized." (WHATWG HTML, the textarea element). Paste CRLF text and the page receives lone LFs, so the CRLF hazard would be invisible and every offset would read one short per line. Ticking the box restores the CR before anything is measured.

The columns are axed by consuming system convention, not by programming language:

| Column | Counts | Systems that use it |
| --- | --- | --- |
| UTF-16 code units | 16-bit units | JavaScript strings, DOM Range and CharacterData offsets, LSP default (`utf-16`), Java, .NET, Windows APIs |
| Unicode code points | scalar values | Python 3 `str`, Go runes, LSP `utf-32`, naive character counters |
| UTF-8 bytes | bytes | Rust `str`, byte-level tokenizers, protobuf `string`, regex over bytes, LSP `utf-8` |
| Grapheme clusters | what a reader sees | user-visible characters, and what a redaction rectangle has to cover |

### The example that ships with it

One support-ticket line: an acute accent in a word, a diaeresis on a name, a ZWJ emoji sequence, a surname and a card number. The page renders the real characters from escapes; the files in this repo are pure ASCII.

Measured by the page: **53 UTF-16 code units, 51 Unicode code points, 61 UTF-8 bytes, 49 grapheme clusters.** Four lengths, one string.

A byte-level model returns `start 43, end 49` for the customer surname. Hand those two integers to four readers of the same text:

| Convention | What `[43, 49)` selects |
| --- | --- |
| UTF-8 bytes | `Muller` |
| UTF-16 code units | `card 4` |
| Unicode code points | `rd 424` |
| Grapheme clusters | ` 4242.` |

The redaction case is the one worth staring at. A rectangle drawn over grapheme clusters 43 to 49 blacks out part of the card number and leaves the surname legible. Nothing raised an error anywhere. The correct span in each convention is UTF-16 `[35, 41)`, code points `[33, 39)`, graphemes `[31, 37)`, so handing byte offsets straight to the JavaScript UI drifts both ends by -8.

A second example reproduces the string from the LSP spec, `a` U+10400 `b`, where the offset of `b` is 2 counted in code points and 3 counted in UTF-16, and shows what the page does when a span starts on the low surrogate: it names the pair and refuses, rather than returning half a character. A third example carries a BOM, CRLF line endings, an NFD name and an undecoded `&amp;` at the same time, and loads with the CRLF box already ticked.

## Why this exists

Offsets are the part of an NLP or agent pipeline that nobody owns. The model emits them, the API forwards them, the UI trusts them, and no layer states its convention. When they are wrong the failure is silent: a highlight lands on the wrong words, a redaction covers the wrong characters, an annotation store fills with spans that no longer address anything. No exception, no log line.

Three things keep this concrete instead of a Unicode lecture.

**The protocol admits it.** LSP is implemented by every major editor and it still needed `positionEncoding`, negotiated at initialization, with UTF-16 kept as the only mandatory value for backward compatibility. That is a documented protocol failure, not a hypothetical.

**Some systems refuse and some do not.** Rust slicing panics rather than return a mangled substring: "Panics if begin or end does not point to the starting byte offset of a character (as defined by is_char_boundary), if begin > end, or if end > len." (std::str, https://doc.rust-lang.org/std/primitive.str.html). JavaScript `String.prototype.slice` does not refuse. It hands back a lone surrogate that renders as a replacement glyph or as nothing at all. This page names the case instead of showing you mangled output.

**The grapheme column hard-fails rather than guess.** If `Intl.Segmenter` is missing, the grapheme column computes nothing and says it is unavailable in this browser. There is no regex fallback, because a regex approximation of grapheme clusters is wrong for exactly the ZWJ sequences, flag pairs and skin-tone modifiers that column exists for. Shipping that fallback would reproduce the bug class the tool is built to find. `Intl.Segmenter` reached Baseline newly available on 2024-04-16 (web-features, feature key `intl-segmenter`), supported from Chrome 87, Edge 87, Firefox 125, Safari 14.1 and Safari on iOS 14.5, so refusing is cheap and guessing is the entire subject of the repo. A checkbox on the page simulates the missing API so you can see the refusal without hunting for an old browser.

Receipts for the four columns:

- UTF-16 code units: "Index values refer to char code units, so a supplementary character uses two positions in a String." (java.lang.String, Java SE 21 API documentation). The WHATWG DOM standard defines `CharacterData` offsets in code units, and `Range` start and end offsets into a text node are offsets into that data.
- Unicode code points: "Strings are immutable sequences of Unicode code points." (Python documentation, Text Sequence Type - str). The Go specification, For statements: "For a string value, the "range" clause iterates over the Unicode code points in the string starting at byte index 0." LSP records `utf-32` as the same axis: "Implementation note: these are the same as Unicode code points, so this `PositionEncodingKind` may also be used for an encoding-agnostic representation of character offsets."
- UTF-8 bytes: "Rust libraries may assume that string slices are always valid UTF-8." (std::str). "A string must always contain UTF-8 encoded or 7-bit ASCII text" (Protocol Buffers, Language Guide proto3, Scalar Value Types). LSP names this axis `utf-8`, documented as "Character offsets count UTF-8 code units (e.g bytes)."
- Grapheme clusters: Unicode Standard Annex #29, Unicode Text Segmentation: "This annex describes guidelines for determining default boundaries between certain significant text elements: user-perceived characters, words, and sentences." and "The default grapheme clusters are also known as extended grapheme clusters." ECMA-402 lists UAX #29 as a normative reference, and that is what `Intl.Segmenter` with `granularity: "grapheme"` segments on.

Every verdict on the page is a measurement of the text and the spans you pasted into it. None of it is a prediction about your production system.

## Privacy

Everything runs in the browser tab. The source text, the spans and the converted output never leave the page. No network requests, no external scripts, no fonts fetched from anywhere, no analytics, and nothing stored except the one theme preference in `localStorage`. Open DevTools, watch the network tab, paste a production string, and confirm nothing goes out.

## Run locally

```bash
git clone https://github.com/0xelitesystem/span-offset-debugger
cd span-offset-debugger
# open index.html in a browser, or:
python -m http.server 8000
```

## Build

No build. One HTML file with inline CSS and JavaScript. No dependencies, no bundler, no package manager, no network at runtime.

## Related

Those are about characters. This one is about spans, which is a different failure.

- [unicode-codepoint-breakdown](https://github.com/0xelitesystem/unicode-codepoint-breakdown), what a single character is: code point, name, encodings
- [unicode-and-normalization-reference](https://github.com/0xelitesystem/unicode-and-normalization-reference), NFC, NFD, and why two strings that look identical compare unequal
- [character-encoding-utf8-basics-reference](https://github.com/0xelitesystem/character-encoding-utf8-basics-reference), what UTF-8 is and how a code point is laid out in bytes

Use those to understand a character. Use this one when two systems disagree about where a character is.

## License

MIT.

## More

Part of a catalog of single-file browser tools and plain-language references, all MIT licensed and dependency-free: [0xelitesystem.github.io](https://0xelitesystem.github.io/).
Built by [elitesystem.ai](https://elitesystem.ai).
