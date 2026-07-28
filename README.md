# Frog Studio X3 + Remix Machine — Deep Merge

One real HTML file (`index.html`) containing both apps' complete code,
switchable via a top bar. Not an iframe trick this time — actual merged
markup, CSS, and scripts in a single document.

## How the merge avoided breaking either app

Both apps independently define things like `.btn`, `.row`, `--panel`, each
meaning something different. Rather than rename everything (high risk of
missing a reference somewhere in ~1600 + ~2000 lines of code I'd have to
re-verify by hand), the merge uses the CSS `@scope` at-rule:

```css
@scope (#frog-app)  { /* Frog Studio's entire original CSS, untouched */ }
@scope (#remix-app) { /* Remix Machine's entire original CSS, untouched */ }
```

`@scope` makes every selector inside only match elements that are
descendants of that container — so both apps can use identical class names
and CSS variables (`--panel`, `.btn`, `.row`, `.active`, `.on` all collided)
with zero interference. `@scope` needs Chrome/Android WebView 118+ (shipped
late 2023) — any phone with auto-updating Chrome from the last ~2 years
has it.

**The one thing `@scope` can't fix**: JavaScript's `getElementById()` isn't
scoped — it searches the whole document. Both apps had a `masterVol` and a
`playBtn` id. Remix Machine's copies were renamed to `rmxMasterVol` /
`rmxPlayBtn` (Frog Studio's are untouched) — this is the only actual code
change made to either app's logic anywhere in this merge.

## What that means for you

- **Frog Studio X3**: 100% identical behavior to your upload. Nothing
  changed inside it.
- **Remix Machine**: identical behavior, with two internal element IDs
  renamed (invisible to you — same buttons, same everything). Saved
  project JSON files from Remix Machine still load correctly; the
  save/load format itself wasn't touched, only the DOM ids were.
- Both apps' audio engines are fully independent (each has always created
  its own AudioContext) — same caveat as before: nothing stops one
  automatically if you start playing in the other while it's running in
  the background.
- Each app's own sticky header now sits just below the switcher bar
  (shifted down 48px) instead of underneath it.

## Installing

Same as before — for the full offline install (not just a shortcut),
Chrome needs HTTPS. Unzip and open `index.html` locally for a quick
"Add to Home Screen," or drop the folder on **Netlify Drop**
(netlify.com/drop) for a free HTTPS link and the real Install prompt.
