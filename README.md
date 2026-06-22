# Le Souffleur

A web app for memorizing theater scripts. Paste a script, pick the character you play, and the app hides a tunable percentage of the words in **your** lines behind tappable blanks. Tap a blank to reveal the word.

The interface is available in **French and English** — switch anytime with the language button in the top bar.

## Features

- Adjustable hiding (percentage slider)
- Tap a blank to reveal the word
- "Shuffle" to redraw which words are hidden
- Light / dark theme
- French / English interface
- Local save + optional cloud sync

## Running it

No build step and nothing to install. Just open `index.html` in a browser, or serve the folder statically:

```sh
python -m http.server
```

then open the address it prints.

## Cloud sync (optional)

Add your Supabase credentials to `config.js`:

```js
window.SUPABASE_URL = "https://xxxx.supabase.co";
window.SUPABASE_ANON_KEY = "your-anon-key";
```

The `anon` key is public by design — security is enforced by Supabase Row Level Security (RLS). Never put the `service_role` key here.

The app also works entirely offline, without `config.js`: choose "Continue without an account" and everything stays local.

## Stack

Vanilla HTML / CSS / JavaScript (a single `index.html` file, one `"use strict"` IIFE). Supabase JS loaded from a CDN. No framework, no build.
