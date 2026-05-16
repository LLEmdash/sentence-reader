# Sentence Reader

A speed-reading web app that highlights one sentence at a time and holds each sentence for a duration proportional to its length. Most speed-readers flash one word at a time (RSVP) or sweep a highlight at a fixed pace; neither respects the rhythm of prose. This one does.

The math is simple: `duration = (word_count / WPM) × 60 seconds`, with a configurable minimum floor so one-word sentences don't blink past, an extra pause after each period, and a longer pause between paragraphs. Short sentences flash by. Long, clause-laden sentences get the time they deserve.

## Features

Paste any text and read it sentence by sentence at your chosen WPM. Adjust pacing on the fly with the slider. Skip forward and back, or tap any sentence to jump to it. Keyboard shortcuts: space to play/pause, arrow keys to step. Settings panel exposes the minimum hold time, the extra pause after periods, the pause between paragraphs, and an auto-scroll toggle. The sentence parser handles common abbreviations, decimals, and ellipses without false splits.

## Running locally

This is a single HTML file with no build step and no dependencies beyond Google Fonts. Open `index.html` in any modern browser and it works. For development, just edit and refresh.

Note: iOS Safari won't open local HTML files from the Files app — tapping the file shows source code in Quick Look rather than rendering it. To test on mobile, deploy to a URL (see below) or use a desktop browser.

## Deploying

The easiest free options, in order of friction:

**Netlify Drop.** Go to [app.netlify.com/drop](https://app.netlify.com/drop) and drag the project folder onto the page. You get a URL in seconds. Free, no account needed for a trial deploy. Sign up to keep it permanently.

**GitHub Pages.** Push the repo to GitHub, open Settings → Pages, select the branch, save. The site lives at `https://<username>.github.io/<repo>`. Free, permanent, version-controlled.

**Vercel or Cloudflare Pages.** Both are similar to Netlify, both free, both connect to a GitHub repo for automatic deploys on push.

Once deployed, open the URL in mobile Safari and use "Add to Home Screen" — the app then opens full-screen without browser chrome.

## Tech notes

Single HTML file (`index.html`) containing HTML, CSS, and JavaScript inline. No framework, no build, no package manager. Fonts loaded from Google Fonts. Designed to work in modern browsers including mobile Safari and Chrome.

The scroll behavior uses `element.scrollIntoView({ block: 'nearest' })` with CSS `scroll-margin` for offsets, which propagates reliably across iframe boundaries in mobile webviews — `window.scrollTo` is not reliable in that context.

## Roadmap

Planned in roughly this order: text cleaner for messy paste input (rejoin hyphenated line breaks, normalize quotes, strip page numbers and footnote markers), a library backed by IndexedDB so saved texts persist with progress, EPUB import via JSZip, and PWA polish (manifest, service worker, offline support).

## License

Personal project. No license set yet.
