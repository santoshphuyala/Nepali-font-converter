# Nepali Font Converter v5: Proposal and Implementation Checklist

Prepared from the point of view of a Nepali font and text-conversion analyst, on the basis of v4 as built and tested in this project. Effort sizes (S, M, L) are rough estimates for planning, not commitments. Anything marked **Decision** needs your answer before work starts.

**Contents**

1. [Where v4 stands](#1-where-v4-stands)
2. [Priorities at a glance](#2-priorities-at-a-glance)
3. [Feature specifications](#3-feature-specifications)
4. [Implementation checklist](#4-implementation-checklist)
5. [Test plan and regression corpus](#5-test-plan-and-regression-corpus)
6. [Risks and decisions needed](#6-risks-and-decisions-needed)
7. [Definition of done](#7-definition-of-done)

---

## 1. Where v4 stands

**Strengths**

* Two-way conversion between Preeti, Kantipur, Fantasy Himali, Sagarmatha, Kanchan and Unicode.
* Word, Excel, PowerPoint and text files, with layout, breaks and formatting kept.
* Quality tools: font auto-detect, suspicious-word flags, spelling-variant warnings, built-in rule check (115 conversion tests and 115 round-trips) and an original-vs-converted report with CSV.
* Fix system: per-word fixes, find/replace rules, export and import.
* Everything runs in the browser; nothing is uploaded.

**Known gaps found while building and testing v4**

| # | Gap | Effect on users |
|---|---|---|
| G1 | Kantipur, Himali, Sagarmatha and Kanchan are converted with the Preeti key layout. Differences in those fonts have not been verified against real samples. | Some glyphs may convert wrongly for those fonts. |
| G2 | English detection uses a small built-in word list. A less common English word in a legacy font is converted as if it were Preeti ("Hello" in the tests). | Manual **EN** presses. |
| G3 | Spelling warnings come from a short list (about 60 entries), not a dictionary. Typing mistakes in the source file that make a non-word are not caught. | Errors in the source pass through. |
| G4 | A Preeti colon and a visarga are the same shape, so a trailing `M` cannot be told apart. Anusvara versus nasal conjunct (संकल्पना / सङ्कल्पना) and ZWNJ are not options. | Style choices cannot be set. |
| G5 | One file at a time. The page freezes briefly on very large files because conversion runs on the main thread. | Slow for bulk work. |
| G6 | Not converted: PDF files, old .doc/.xls/.ppt, formulas and sheet names in Excel, text in images, data in embedded chart workbooks, document properties. | Manual follow-up. |
| G7 | Word, Excel and PowerPoint need a library (JSZip) loaded from the internet. | No Office conversion offline. |
| G8 | Converted Word and PowerPoint runs are not tagged as Nepali, so spell-check may treat them as English. | Red squiggles after conversion. |
| G9 | The word list shows 200 words and there is no side-by-side view of original and converted text in context. | Hard to review long documents. |
| G10 | Interface is English only. Your own test cases cannot be added from the screen. | Learning from mistakes depends on the maintainer. |

---

## 2. Priorities at a glance

| ID | Feature | Fixes | Size | Release |
|---|---|---|---|---|
| A1 | Font-specific mapping profiles (Kantipur, Himali, Sagarmatha, Kanchan) | G1 | L | 5.0 |
| A2 | Nepali dictionary check and suggestions | G3 | L | 5.0 |
| A3 | Smarter English and mixed-language detection, with your own English list | G2 | M | 5.0 |
| A4 | Style options: colon/visarga, anusvara/nasal, ZWNJ, danda spacing | G4 | M | 5.0 |
| A5 | "Add to my tests": your own regression tests from the screen | G10 | S | 5.0 |
| A6 | Unicode clean-up tool | new | M | 5.0 |
| B1 | Batch conversion with zip download and background processing | G5 | L | 5.0 |
| B2 | Nepali language tagging (ne-NP) on converted text | G8 | S | 5.0 |
| C1 | Side-by-side review view with click-to-fix | G9 | M | 5.1 |
| C2 | Undo and redo for fixes; fix sets with merge import | new | M | 5.1 |
| B3 | PDF text extraction (legacy-font PDFs) | G6 | L | 5.1 |
| B4 | More file types (.odt, .ods, .odp, .rtf, .html, .srt, .json) | G6 | M | 5.1 |
| B5 | Deeper Word, Excel and PowerPoint coverage | G6 | M | 5.1 |
| D1 | Offline use: bundled library, installable app | G7 | M | 5.2 |
| D2 | Nepali interface, accessibility and mobile polish | G10 | M | 5.2 |
| D3 | Typing aids: Preeti keyboard emulator, Roman → Unicode | new | M | 5.2 |
| D4 | Audit report with version, rule set and file fingerprint | new | S | 5.2 |
| D5 | Office tools: Nepali amount in words, BS ⇄ AD dates | new | M | optional |

Recommended order: **5.0** is accuracy and scale (A, B1, B2), **5.1** is review and reach, **5.2** is offline and polish.

---

## 3. Feature specifications

### A1. Font-specific mapping profiles

**Why.** The five legacy fonts share the Preeti layout for most keys, but differ in some special glyphs and conjuncts. Without checking, those glyphs convert wrongly.

**How it works**

* The key tables become data (one profile per font) instead of code: base layout plus a list of overrides per font.
* A profile editor lists every key, the glyph it should give, and the character code for review.
* Profiles can be exported and imported as JSON.
* The font family found by v4's auto-detect picks the profile.
* Each profile has its own test list.

**Acceptance criteria**

* [ ] For each font there is a set of at least 100 real sample words, checked by a Nepali typist or editor, that convert correctly.
* [ ] A character that differs between fonts converts according to the selected font.
* [ ] Unicode → font output uses the same profile in reverse.
* [ ] Changing the profile never changes Preeti results (the existing 115 tests still pass).

**Decision.** Real sample text from Kantipur, Himali, Sagarmatha and Kanchan is needed. Without it this feature cannot be verified.

### A2. Nepali dictionary check and suggestions

**Why.** Flags today only catch impossible letter sequences. A word list catches typing mistakes in the source file and unlikely conversions.

**How it works**

* After conversion, each word is looked up in an offline word list.
* Unknown words get an amber "not in dictionary" chip and up to three nearest suggestions (edit distance, plus common confusions such as ि/ी, ु/ू, श/ष/स, ब/व).
* The word list also feeds the spelling suggestions, replacing the hand-made list.
* Users can add words to a personal dictionary (stored locally, exportable).
* Names, places and abbreviations are common false alarms, so the check can be switched off and personal words are never flagged again.

**Acceptance criteria**

* [ ] Dictionary loads without blocking the page and works offline.
* [ ] On the regression corpus, fewer than 5% of correct words are flagged as unknown (target to be confirmed on real documents).
* [ ] Suggestions can be applied as saved fixes with one click.
* [ ] Personal dictionary can be exported and imported.

**Decision.** Source and licence of the word list, and the size limit for a single HTML file (a useful list may add several megabytes).

### A3. Smarter English and mixed-language detection

**Why.** Today only a short built-in list protects English words (G2).

**How it works**

* Larger English word list, loaded on demand.
* A per-token test: Preeti text rarely contains the patterns English does (a vowel in nearly every syllable, standard endings such as -ing, -tion); combine with the list and the font of the run.
* In office files, a run in a non-Nepali font is always English (already true); extend this to cells, text boxes, headers and notes.
* A personal "always English" list, managed from the word list (the **EN** button adds to it), exportable.
* Show a "kept as English" count in the report with the first 50 examples to check.

**Acceptance criteria**

* [ ] "Hello", "Age", names of months and common office words are kept in a Preeti-typed paragraph.
* [ ] No Nepali word from the regression corpus is wrongly treated as English.
* [ ] Personal English list survives reloads and can be exported.

### A4. Style options

| Option | Choices | Default |
|---|---|---|
| Trailing Preeti `M` | colon (:) / visarga (ः) / ask (flag it) | flag it |
| Anusvara and nasal conjunct | keep as typed / write ङ्, ञ्, ण्, न्, म् before a consonant of its group / write ं | keep as typed |
| ZWNJ after final halant before a following word | off / on | off |
| Space before danda and punctuation | keep / remove | keep |
| Digits | keep / Devanagari / Latin (exists) | keep |
| Quotation marks | keep / straighten / curl | keep |
| Spelling standard for suggestions | Nepal Academy / as typed | Nepal Academy |

**Acceptance criteria**

* [ ] Each option has a test in the rule check.
* [ ] Options are saved with the user's settings and shown in the report header.
* [ ] "Keep as typed" never changes the text.

### A5. "Add to my tests"

**Why.** Every time a word converts wrongly, the correct pair (typed text, expected result) is a new test. Today it has to be sent to the maintainer.

**How it works**

* In the word list, a **Add as test** button stores the typed word and the corrected result.
* The rule check runs built-in tests and the user's tests; failures are marked separately.
* Tests can be exported as JSON and as a plain list for sending to the maintainer.

**Acceptance criteria**

* [ ] A user fix can be turned into a test in one click.
* [ ] The rule check shows "built-in 116/116, yours 7/7".
* [ ] Exported tests can be imported on another computer.

### A6. Unicode clean-up tool

**Why.** Text that is already in Unicode often has errors that look fine but break search, sorting and fonts.

**Checks and fixes (each can be turned on or off)**

* Normalise to Unicode NFC, then fix orders NFC leaves wrong (for example vowel sign typed before the consonant, nukta, deprecated forms).
* Remove stray zero-width characters, double spaces, spaces before । , ; : ! ?
* Replace look-alike characters (Latin `|` for ।, Latin `0`, `1` in Devanagari words, Devanagari ऽ typed for ्).
* Standardise ं / ँ and the ङ्, ञ् before consonants (optional, see A4).
* Report all changes in the report.

**Acceptance criteria**

* [ ] A test set of 50 known-bad strings is fixed correctly and nothing else is changed.
* [ ] Works on pasted text and on office files in the same way as conversion.

### B1. Batch conversion

**How it works**

* Drag and drop several files (or a whole folder where the browser allows it); each is listed with its detected font and status.
* Conversion runs in a background worker, with a progress bar and a Cancel button, so the page stays responsive.
* Output is one zip with the converted files and one combined report (CSV and per-file).
* Files that fail are listed with the reason; the rest are still produced.

**Acceptance criteria**

* [ ] 50 mixed files convert without freezing the page.
* [ ] A 30 MB workbook converts without running out of memory, or stops with a clear message.
* [ ] Output names never overwrite each other.

### B2. Nepali language tagging

* In Word, set `w:lang` (and `w:lang w:bidi`) to ne-NP on converted runs; in PowerPoint set `lang` to ne-NP; in text boxes and chart text likewise.
* Legacy-font output keeps the original language tag.
* **Acceptance:** converted documents show no spell-check errors for correct Nepali words in Word 365 with the Nepali proofing tools installed.

### C1. Side-by-side review

* Two columns: original and converted, paragraph by paragraph (or row by row in Excel, slide by slide in PowerPoint), with changed words highlighted and flags shown as chips.
* Click a word to fix it, press **EN**, accept a suggestion, or add as test.
* Jump to next flagged word with a key.
* Large files are paged (for example 50 paragraphs at a time).

**Acceptance:** a 100-page Word file can be reviewed from start to end without freezing; fixes made in the view update the download.

### C2. Undo and fix sets

* Undo and redo for fixes, rules and EN changes in the current session.
* Named fix sets (for example one per organisation) that can be turned on and off.
* Import offers **merge** (keep both, show conflicts) or **replace**.
* Fix history with date and the word, so a bad fix can be traced.

### B3. PDF text extraction

**Why.** Many old Nepali documents are PDFs in which the text is Preeti-encoded.

**How it works**

* Extract text with its font names using a PDF library; convert text in legacy fonts and keep English text.
* Output a Word file (paragraphs kept; layout is approximate) or a text file.
* Scanned PDFs (images) are not supported and the page says so clearly.

**Acceptance:** a set of 10 real PDFs with Preeti text gives correct text; the page warns that layout, columns and tables are not preserved.

**Decision.** Worth the added library size, and accept that layout is not kept?

### B4. More file types

| Type | Approach |
|---|---|
| .odt, .ods, .odp | Same ZIP-and-XML method as Office files; convert `text:p`, `text:span` and font declarations. |
| .rtf | Parse the font table and runs; convert runs in legacy fonts. |
| .html, .htm | Convert text nodes inside elements whose font-family is a legacy font; update the font-family. |
| .srt, .vtt | Convert text lines only; keep timecodes. |
| .json, .xml | Convert only string values chosen by the user. |
| Old .doc, .xls, .ppt | Stay unsupported; give a clear message and a one-click how-to for Save As. |

### B5. Deeper Office coverage

| App | Add |
|---|---|
| Word | Content controls, SmartArt text, text in fields, document properties (title, subject, keywords), tracked-change text. |
| Excel | Text in formulas (optional, off by default), sheet names (optional, with formula references updated), data-validation lists, conditional-format text, headers and footers, text of pivot caches, embedded chart workbooks. |
| PowerPoint | Embedded chart workbooks, alt text, section and slide names, comments, hyperlink screen tips. |

Every item is optional and listed in the report; anything not converted is named in the status line.

### D1. Offline use

* Bundle the zip library inside the single file (or a service worker cache) so Office files convert without internet.
* Make the page installable on desktop and phone and usable offline.
* Show the version and rule-set date in the page footer.

**Acceptance:** with the network off, an installed copy converts a .docx and a .xlsx.

### D2. Nepali interface, accessibility and mobile

* Language switch (English / नेपाली) for all labels and messages.
* Keyboard shortcuts (paste and convert, copy, next flag, run rule check).
* Screen-reader labels, visible focus, colour contrast checked in light and dark mode, text-size setting.
* Phone layout checked (the word list, report table and typing guide are the hardest parts).

### D3. Typing aids

* **Preeti keyboard emulator:** type Preeti keys on any keyboard and see Nepali directly.
* **Roman → Unicode:** simple transliteration for short entries.
* **On-screen Unicode keyboard** for matras and conjuncts.

### D4. Audit report

* Header with app version, rule-set version and date, From, To, options used, file name, size and a fingerprint (SHA-256) of the original and the converted file.
* Export as CSV, HTML or PDF. Useful when converted documents must be traceable (official records).

### D5. Office tools (optional)

* **Amount in words** (Nepali numerals and words, lakh/crore system) for vouchers and cheques.
* **BS ⇄ AD date** converter, and detection and conversion of dates inside text.

---

## 4. Implementation checklist

Tick each item when it is done and tested. Items marked ★ are the minimum for each release.

### Phase 0. Preparation (before any coding)

* [ ] ★ Decide the open questions in section 6.
* [ ] ★ Collect real sample files: at least 10 Word, 5 Excel, 5 PowerPoint, 5 text and (if B3) 10 PDF files per legacy font, with a corrected "answer" for each.
* [ ] ★ Freeze v4 as the baseline; keep its 115 tests as the first regression set.
* [ ] Write a short sign-off checklist for the Nepali editor who will check results.
* [ ] Set up a simple issue list with the gap codes G1 to G10.
* [ ] Choose how the single file is built (one HTML file kept by hand, or a build step that joins separate source files into one).

### Phase 1. Version 5.0: accuracy and scale

**Data structure**

* [ ] ★ Move key tables into per-font profile data (A1), with Preeti as the base.
* [ ] ★ Add a rule-set version number and date, shown in the footer and the report.

**Accuracy**

* [ ] ★ Build verified sample sets for Kantipur, Himali, Sagarmatha and Kanchan and fix the differences (A1).
* [ ] ★ Add the dictionary and the "not in dictionary" flag with suggestions (A2).
* [ ] ★ Larger English list, per-token test and personal English list (A3).
* [ ] ★ Style options with tests for each (A4).
* [ ] ★ "Add as test" and the combined rule check (A5).
* [ ] Unicode clean-up tool with its 50-string test set (A6).
* [ ] Language tagging ne-NP in Word, PowerPoint and text boxes (B2).

**Scale**

* [ ] ★ Move conversion into a background worker with progress and Cancel (B1).
* [ ] ★ Multi-file input, zip output and combined report (B1).
* [ ] Memory guard: warn above a set file size and stop cleanly.

**Release tasks**

* [ ] ★ All tests in section 5 pass.
* [ ] ★ Update the README (new options, rule book additions, version history).
* [ ] ★ Smoke test on the browsers and Office versions in section 5.

### Phase 2. Version 5.1: review and reach

* [ ] ★ Side-by-side review view with click-to-fix and paging (C1).
* [ ] ★ Undo and redo; named fix sets; merge import (C2).
* [ ] PDF text extraction with the "layout not kept" warning (B3).
* [ ] ODF, RTF, HTML, SRT and JSON support (B4).
* [ ] Deeper Word, Excel and PowerPoint coverage, each item switchable (B5).
* [ ] Update README and rule book.

### Phase 3. Version 5.2: offline and polish

* [ ] ★ Bundle the zip library and add offline use (D1).
* [ ] ★ Nepali interface and keyboard shortcuts (D2).
* [ ] Accessibility pass and phone layout pass (D2).
* [ ] Typing aids (D3).
* [ ] Audit report with fingerprints (D4).
* [ ] Optional office tools (D5).

### Release routine (every version)

* [ ] Run the built-in rule check and the full regression corpus; save the results.
* [ ] Convert the sample files for every format and open each result in Word, Excel and PowerPoint (and LibreOffice); confirm there is no repair prompt.
* [ ] Round-trip check: Preeti → Unicode → Preeti returns the original text on the corpus.
* [ ] Check light and dark mode, phone and desktop.
* [ ] Update version number, rule-set date, README and the change list.
* [ ] Keep the previous version file available.

---

## 5. Test plan and regression corpus

**Corpus**

* The 115 built-in conversion tests and 115 round-trips from v4.
* The full 100-word list supplied for v4, with its typing errors corrected.
* Every word reported as wrong during use, with its corrected result.
* Real documents per font (Phase 0), each with an editor-approved answer file.

**Targets (to be confirmed on real documents)**

| Measure | Target |
|---|---|
| Built-in tests | 100% pass |
| Round-trip on corpus | 100% match |
| Word-level accuracy on the real-document corpus | at least 99.5% after the dictionary check |
| False "English" or "not in dictionary" flags | under 5% of correct words |
| Office files that open without a repair prompt | 100% |

**Compatibility matrix**

| Area | Versions |
|---|---|
| Browsers | Current Chrome, Edge, Firefox, Safari; Chrome on Android; Safari on iPhone |
| Office | Word, Excel and PowerPoint 2016 and Microsoft 365; LibreOffice; WPS Office |
| Import checks | Open converted files in Google Docs, Sheets and Slides |
| Files | Large (30 MB or more), password-protected (must give a clear message), damaged (must not crash) |

**Rules for adding a test**

1. Write the typed text exactly as it appears in the source.
2. Write the expected Unicode result checked by a Nepali editor.
3. Add the reverse check (Unicode → font → Unicode).
4. If a rule changes, rerun the full corpus and record every result that changed.

---

## 6. Risks and decisions needed

**Decisions**

| # | Question | Needed for |
|---|---|---|
| 1 | Can you provide real sample text from Kantipur, Himali, Sagarmatha and Kanchan? | A1 |
| 2 | Which Nepali word list may be used, and how large may the single HTML file be? | A2 |
| 3 | Is PDF support wanted, accepting that layout is not preserved? | B3 |
| 4 | Is the app used only as one HTML file, or may it be hosted with several files? | B1, D1 |
| 5 | Which Nepali spelling standard should suggestions follow? | A2, A4 |
| 6 | Is a Nepali interface needed, and who will review the translation? | D2 |

**Risks**

| Risk | Reduce it by |
|---|---|
| A rule that fixes one word breaks another | Run the whole corpus after every rule change; show a before/after list of changed results |
| Dictionary gives many false alarms on names and abbreviations | Personal dictionary; allow turning the check off; tune on real documents |
| Large files crash the browser | Background worker, size warnings, file-by-file processing |
| Converted Office files fail to open | Convert only text and font names; test every release in Word, Excel, PowerPoint and LibreOffice |
| Font licences | Do not bundle Preeti and similar fonts; rely on fonts the user has installed, and use a free Unicode font for display |
| Spelling suggestions seen as official | Label them as suggestions, never change text without a click |
| Fixes lost when browser data is cleared | Prompt to export fixes, tests and the dictionary; add an automatic reminder |

---

## 7. Definition of done

A version is ready to release when all of these are true:

* [ ] Every ★ item for the version is complete.
* [ ] The built-in rule check and the regression corpus pass at the targets in section 5.
* [ ] Converted files from every supported format open cleanly in the Office versions in the compatibility matrix.
* [ ] A Nepali editor has reviewed a sample of converted real documents and signed off.
* [ ] README, rule book and version history are updated.
* [ ] The previous version is kept for rollback.
