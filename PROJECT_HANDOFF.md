# Sentence Reader — Project Handoff

This document is a full context dump for a new Claude Code session picking up this project. It covers the concept, what's been built, the technical decisions, bugs encountered and lessons learned, and the roadmap options that have been discussed. The accompanying file `sentence-reader.html` is the current working implementation.

---

## The Core Concept

Most speed-reading apps work in one of two modes: RSVP (Rapid Serial Visual Presentation), where one word at a time flashes in the center of the screen, or a moving highlight that sweeps across text at a fixed WPM. Apps like Spreeder, Outread, Spritz, Reedy, and AccelaReader all fall into one of these camps.

The user's insight: neither approach respects the rhythm of prose. RSVP destroys sentence shape entirely. Moving highlights ignore punctuation. The user wanted something different — **highlight one whole sentence at a time, and hold each sentence for a duration proportional to its word count.** A short sentence ("He paused.") gets a short hold. A long, clause-laden sentence gets the time it deserves. The math is simple: `duration = (word_count / WPM) × 60 seconds`, with a minimum floor (so one-word sentences don't blink past) and a small extra pause after each period.

A web search didn't surface any existing app that does exactly this, so we built it.

---

## Current State

`sentence-reader.html` is a single-file HTML artifact (HTML + CSS + JS, no build step, no dependencies beyond Google Fonts). It runs in any modern browser. As of the last edit, it:

- Splits pasted text into sentences using a regex-based parser that handles common abbreviations (Mr., Dr., e.g., U.S., etc.), decimal numbers (3.14), and ellipses.
- Treats double-newlines as paragraph boundaries and remembers which sentence ends each paragraph.
- Highlights one sentence at a time with a warm yellow highlighter color, dims sentences that haven't been read yet, and tints already-read sentences as a slightly faded ink color.
- Holds each sentence for `max(minMs, words/WPM × 60000) + pauseMs`, plus an additional `paraPauseMs` after sentences that end a paragraph.
- Auto-scrolls the active sentence into view, using `scrollIntoView({ block: 'nearest' })` with CSS `scroll-margin-top: 80px` and `scroll-margin-bottom: 200px` baked into each sentence to keep it clear of the fixed control bar.
- Has a play/pause/prev/next transport, a WPM slider (100–800, default 300), tap-any-sentence-to-jump, keyboard shortcuts (space, arrow keys), and a settings panel for the minimum-ms-per-sentence, extra-pause-after-period, paragraph-pause, and auto-scroll toggle.
- Includes a paste modal and a built-in sample text.

The design is editorial: a warm cream paper background (`#f3ece0`), a serif body font (Newsreader) for the reading text, Instrument Serif italic for headers, JetBrains Mono for the UI labels and stats, terracotta red (`#b14a2c`) as an accent. Highlighter yellow (`#f5d76e`) for the active sentence. The whole thing is meant to feel like a thoughtful reading object, not a tech demo.

---

## Technical Architecture

The implementation is intentionally minimal — one HTML file, no framework, no build. The JavaScript is wrapped in an IIFE with a small state object and a handful of pure-ish functions. Key pieces:

**Sentence splitter.** Located at the top of the JS. The approach: protect known abbreviations and decimals by temporarily replacing their periods with a placeholder character (`§`), protect ellipses with another placeholder, then split paragraphs by double newlines and split each paragraph by the regex `/(?<=[.!?])["')\]]?\s+(?=["'(\[]?[A-Z0-9—–-])/`. Restore the protected periods and ellipses. Each sentence is returned as `{ text, paragraphBreakAfter }`. This is pragmatic, not perfect — it will mis-split on rare edge cases — but handles everyday prose well.

**Timing.** `durationFor(sentence)` computes the hold time. State stores `wpm`, `minMs` (floor), `pauseMs` (added to every sentence), `paraPauseMs` (added when `paragraphBreakAfter` is true).

**Rendering.** All sentences are rendered as inline `<span class="sent">` elements inside `<p>` tags. Paragraph breaks come from the parser's `paragraphBreakAfter` flag. Each span gets `active`/`read`/nothing classes based on its index relative to `state.idx`. Updates between sentences only toggle these classes (no re-render).

**Scroll.** `scrollActiveIntoView()` calls `active.scrollIntoView({ behavior: 'smooth', block: 'nearest' })`. The scroll-margin CSS on `.sent` does the heavy lifting — telling the browser to leave 80px of clearance above and 200px below the sentence when scrolling it into view. This keeps the highlighted sentence clear of the fixed bottom control bar.

**Controls.** Play/pause uses `setTimeout`, scheduled per sentence with the duration from `durationFor`. A `requestAnimationFrame` loop drives the progress bar. Changing WPM mid-play pauses and resumes to recompute the current sentence's remaining time. The settings panel is hidden by default and toggled by a button in the header.

---

## Bugs Hit and Lessons Learned

These are worth carrying forward because they cost real time to debug:

**The iOS Safari "can't open local HTML file" problem.** When you save an `.html` file to iCloud Files and tap it, iOS shows you Quick Look (raw source code), not a rendered page. There is no "Open in Safari" option for local files. This is a long-standing Apple limitation. The fix is to host the file at a URL — Netlify Drop, GitHub Pages, anything. The user has not yet deployed.

**Mobile webview scroll issues — three layers.** When the artifact runs inside the Claude mobile app, it's loaded in an iframe. We hit three separate issues, in this order:

1. *Inner-div scrolling was set up as `flex: 1; overflow-y: auto` on `.reader-wrap`, with `min-height: 100dvh` on the parent.* This works fine in desktop Safari but breaks in mobile webviews where the iframe is sized to fit content — the inner div doesn't actually scroll because there's no constrained viewport for it to scroll within. Fix: removed the inner scroll, made the page itself scroll.

2. *Even after switching to body-scroll, `window.scrollTo` did nothing.* The cause: `html, body { height: 100%; }` was still in the CSS, which locks the body to viewport height. Content overflowed visually but the body wasn't a scrollable element. Removing `height: 100%` (keeping only `min-height: 100vh` on body for the empty state) fixed this.

3. *`window.scrollTo` still wasn't reliable across iframe boundaries in mobile webviews.* `element.scrollIntoView`, by contrast, propagates scroll requests up through nested scroll containers — including across iframes — because that's how the spec works. Switching to `scrollIntoView({ block: 'nearest' })` with CSS `scroll-margin` for offsets is the robust pattern. Use this from now on.

**Takeaway:** for any web project that needs to work inside iframes (including the Claude mobile app, embedded previews, etc.), prefer `scrollIntoView` over `window.scrollTo`, avoid inner-div scroll containers, and don't lock body height with `height: 100%`.

---

## Roadmap — Three Paths Discussed

The user wants to evolve this into something more substantial. Three paths were discussed in increasing ambition:

**Path 1: Single HTML file with localStorage.** Add a small library of saved texts, remember progress per text, persist settings. Stays a single file. Lives on one device per browser. Quick to build (1–2 hours). Good if it's just for the user on one device.

**Path 2: Static web app, deployed online.** Same features as Path 1, but deployed to Vercel/Netlify/Cloudflare Pages (all free). Real URL. Works on any device, but data is still per-device unless accounts are added. Room to add a proper text cleaner, EPUB import, etc. Maybe 1–2 weekends of work. **This is the recommended sweet spot.**

**Path 3: Full app with backend.** Login, cloud library, multi-device sync. Supabase or Firebase make this less painful than it sounds, but it's still a real commitment with ongoing maintenance. Worth it only if multi-device sync matters or the user wants to share with others.

The user has not committed to a path yet but expressed interest in eventually making this a real web app and possibly adding books. Path 2 with IndexedDB for storage is the natural next step.

---

## Specific Features Discussed

**Text cleaning.** When you copy a chapter from a PDF, e-reader, or webpage, you get hyphenated line breaks, footnote markers, page numbers, repeating headers/footers, smart-quote inconsistencies, zero-width spaces, etc. A good paste-and-clean flow would rejoin hyphenated breaks, collapse single newlines to spaces (preserving doubles as paragraph breaks), strip page-number-only lines, normalize quotes and dashes, optionally strip footnote markers like ¹²³ or `[12]`, and show a preview before committing. This is doable entirely in the browser. The same parser would clean EPUB chapter HTML, so the work isn't wasted.

**EPUB import.** EPUB is a ZIP file containing HTML, CSS, and an OPF manifest. Pipeline: user picks an `.epub` → unzip in browser (JSZip, ~100KB) → read `META-INF/container.xml` to find the OPF → parse OPF to get the spine and TOC → show chapter list → on selection, extract chapter HTML and run it through the text cleaner. About 150–200 lines of code if going the manual route, or use `epub.js` (~70KB) which handles parsing but is designed for rendering — slightly working against the grain for plain-text extraction.

DRM caveat: Amazon, Apple Books, Kobo, and library lending EPUBs have DRM and won't open. Project Gutenberg, Standard Ebooks, Tor's free monthly books, Smashwords, and DRM-free indies work cleanly.

**Storage options.**
- *localStorage:* simple, ~5 MB total — runs out fast for books.
- *IndexedDB:* basically unlimited, browser-native, slightly more code. Right choice for a library of books.
- *File System Access API:* user keeps EPUB files in a folder on their device, app reads them. Chrome/Edge only at the moment.

**Library UI.** A book grid or list showing title, author, cover, and last-read position. Tap to open the table of contents. Tap a chapter to start reading. Resume-where-you-left-off should be automatic.

---

## The Deployment Question

The user discovered they can't open local HTML files in Safari on iOS — when they try, Files only offers Quick Look, which shows source code rather than rendering the page. The recommended fix and natural next step is to deploy.

Easiest paths:
- **Netlify Drop:** drag-and-drop the HTML file at `app.netlify.com/drop`, get a URL in seconds. Free, no account needed for the trial deploy.
- **GitHub Pages:** create a repo, push, enable Pages in settings. Slightly more setup, gives a permanent home and version control. Recommended once moving to Claude Code anyway.
- **Vercel/Cloudflare Pages:** also great, similar to Netlify.

Once deployed, the user can add the URL to their iOS home screen and it behaves like an installed app (full-screen, no browser chrome).

---

## Design Direction — Worth Preserving

The aesthetic is intentional and the user has not complained about it, so it's worth preserving as the project grows. Key tokens:

- **Background:** warm cream paper, `#f3ece0`, with very subtle radial gradient tints.
- **Ink:** `#1c160d` (deep brown-black) for active text, `#4a3f2c` for read text, `#b8a98a` for unread.
- **Highlight:** `#f5d76e` (highlighter yellow) for the active sentence.
- **Accent:** `#b14a2c` (terracotta) for the primary action color.
- **Fonts:** Newsreader for body (designed for reading), Instrument Serif italic for editorial headers, JetBrains Mono for UI labels and the stats counter.

The general feel is "Readwise meets a moleskine" — calm, editorial, paper-like. Stay away from gradient buttons, glassmorphism, or generic SaaS aesthetics.

---

## Open Questions for the User

Before committing to a direction, these are worth clarifying:

1. **Just for personal use, or eventually share with others?** This decides whether accounts/sync matter.
2. **One device or multiple?** If multiple, sync matters — pushes toward Path 3 eventually.
3. **Articles and short pieces, or full books?** If books, EPUB import and IndexedDB become priorities.
4. **Happy doing some text cleanup by hand, or want it to "just work"?** Decides how much effort goes into the cleaner.
5. **How much time to invest — weekend project or ongoing thing?** Sets the scope.

---

## Suggested Next Steps in Claude Code

If picking up from where this conversation ended, a sensible sequence:

1. **Set up a repo.** Initialize a git repo, drop in `sentence-reader.html` as `index.html`. Create a `README.md`.
2. **Deploy immediately** to GitHub Pages or Netlify so the user can test on their phone via URL. Validate that everything that worked in the artifact viewer still works in mobile Safari.
3. **Refactor toward a multi-file structure** as features grow. Split CSS, JS, and HTML once it makes sense (probably after adding the text cleaner). No build step needed yet — modern browsers support ES modules natively.
4. **Add a text cleaner.** A textarea-based paste-and-clean flow with toggles for the common normalizations (rejoin hyphens, normalize quotes, strip page numbers, etc.) and a preview.
5. **Add a library with IndexedDB.** Save cleaned texts as named documents. Show a library screen with the list. Remember the last sentence index per document.
6. **Add EPUB import** with JSZip. Parse OPF and TOC, show chapter picker, run chapter HTML through the cleaner.
7. **Polish and PWA-ify.** Add a manifest, a service worker for offline use, an icon, theme color. Now it installs to home screen as a real app.

---

## The Current File

`sentence-reader.html` is self-contained and around 770 lines. It's working and shippable as-is. The user has tested it inside the Claude mobile app and confirmed the reader, controls, settings, and (after the fixes described above) auto-scroll all work. It has not yet been tested in mobile Safari at a real URL — that's step 2 of the next-steps list and should be verified before building on top.
