# Email archive formats: field notes on PST, OST, OLM, MBOX and EML

Notes collected while implementing a parser for each of these five formats (for [Mailward](https://pst.aivismonitor.com), a browser-based archive viewer — disclosure: I build it). Most of what is written about these formats online is converter marketing; this file is the technical part, including the parts that are inconvenient.

Corrections welcome — open an issue.

## The one-line version of each

| Format | What it actually is | Written by | Readable without the original app? |
|---|---|---|---|
| PST | Single-file B-tree database ([MS-PST]) | Outlook for Windows | Yes, with a real parser |
| OST | Same container as PST, used as an offline cache | Outlook, automatically | Yes — the account lock lives in Outlook, not the file |
| OLM | ZIP of per-message XML | Outlook for Mac (File → Export) | Yes — unzip and parse the XML |
| MBOX | Plain text, one folder of mail, messages split on `From ` lines | Thunderbird, Apple Mail, Google Takeout | Yes, trivially |
| EML | One RFC 5322 message with MIME body | Nearly every client | Yes, it *is* the standard |

## PST: a database, not a mail folder

- The layout is publicly specified as **[MS-PST]**: a node database built from B-trees holding blocks, with a property layer on top. Each message is a set of typed properties, not a blob of RFC 5322 text.
- Consequence: there is no email text inside a PST waiting to be copied out. A reader walks the B-trees, resolves each message's property set, and *reconstructs* a standards-compliant message. Converting a PST is a rewrite, not a repackage — which is why "just rename it" never works.
- Two incompatible variants exist. **ANSI** (Outlook 97–2002) caps at 2 GB and was notorious for corrupting archives that hit the ceiling. **Unicode** (Outlook 2003+) is what any modern `.pst` uses. A reader that handles only one fails confusingly on the other; the variant is distinguishable from the header version bytes.
- No viewer can repair a damaged PST. If Microsoft's own `scanpst` cannot open it, reading tools cannot either — recovery is a different problem from parsing.

## OST: the same format, locked by the application

- Structurally an OST **is** the PST container; [MS-PST] describes both.
- The lock people hit is not in the file. Outlook binds an OST to the profile and account that created it and refuses to open one it cannot associate with a live account. Leave the company or lose the tenant, and a healthy file full of your own mail becomes unopenable *by Outlook*.
- Renaming `.ost` to `.pst` does not help — Outlook inspects the file, not the extension.
- A parser that reads the container directly never consults the account association. That is a technical capability, not an entitlement: only open mailbox data you have the right to access.

## OLM: a ZIP of XML that only one app writes

The least-documented format of the five, so in the most detail:

- An `.olm` is a **ZIP container**. Each message is a small XML document whose elements carry an `OPF` prefix: `OPFMessageCopySubject`, `OPFMessageCopySenderAddress`, `OPFMessageCopyHTMLBody`, and so on.
- Folder hierarchy is expressed by where the XML documents sit inside the archive. Attachments are separate files in the ZIP, referenced from the message XML by `OPFAttachmentURL`.
- **The trap:** OLM writes timestamps as UTC but omits the zone suffix — `2022-05-26T17:55:40`, no trailing `Z`. Parse that as local time and every date in the archive silently shifts by your UTC offset. This is the most common way an OLM conversion goes quietly wrong.
- Every message we have seen carries `OPFMessageCopyHTMLBody`, so HTML body extraction is dependable.

## MBOX: deceptively simple text

- One folder of mail concatenated into one text file; each message starts at a line beginning `From ` (sender + timestamp, the "From_ line").
- Because a body line could also start with `From `, writers escape it as `>From ` — and the variants (`mboxo`, `mboxrd`, `mboxcl`) disagree on exactly how, which is why round-tripping an MBOX through the wrong tool can corrupt quoting.
- **Google Takeout** delivers an entire Gmail account as one large MBOX. Gmail's labels do not become folders — they arrive as an `X-Gmail-Labels` header on each message. Any tool that presents a Takeout export as a folder tree is synthesizing it from those headers.

## EML: the format everything else converts *to*

- A single message: RFC 5322 headers plus a MIME body. It is the interchange format — every conversion out of the proprietary containers above ultimately produces this or MBOX.
- When investigating one: `Received` headers are prepended by each hop, so read them **bottom-up** to follow a message's actual path.

## Appendix: parsing multi-gigabyte archives in a browser

Notes from making the PST parser work on multi-GB files inside a Web Worker, on top of [pst-extractor](https://github.com/epfromer/pst-extractor):

- pst-extractor funnels every byte it reads through one method, `PSTFile.readSync`. Subclass it, override that method on the prototype, and you can feed the parser from any random-access source — no filesystem required.
- We back it with a paged reader: 64 KB pages, LRU-capped at 64 MB, loaded on demand via `File.slice()` + `FileReaderSync`. A 4 KB slice read measures ~200 µs in Chrome — fast enough that B-tree walks feel instant, and the file is never loaded whole.
- In Node the same problem has a one-line fix: construct `PSTFile` with the *filename*, not a Buffer from `fs.readFileSync` — the descriptor-backed path has no 2 GB Buffer ceiling.

## Further reading

Longer write-ups of everything above, kept current:

- [Email archive formats, explained](https://pst.aivismonitor.com/email-archive-formats) — the full reference this file condenses
- [Open a PST file without Outlook](https://pst.aivismonitor.com/) · [Open an OST file](https://pst.aivismonitor.com/open-ost-file-online) · [Open an OLM file](https://pst.aivismonitor.com/open-olm-file-online)
- [MS-PST specification](https://learn.microsoft.com/en-us/openspecs/office_file_formats/ms-pst/141923d5-15ab-4ef1-a524-6dce75aae546) (Microsoft)

---

Licensed [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) — reuse freely with attribution.
