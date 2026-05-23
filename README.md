# Custom Views Demo Vault

This is a demo Obsidian vault for the [Custom Views](https://github.com/anupchavan/obsidian-custom-views) plugin by me. It shows how to build rich, data-driven note layouts — illustrated here with a **movie tracking** use case that pulls live data from TMDB and renders an Apple TV-style card for each movie note.

---

## What's in this vault

```
├── References/          # Sample movie notes
├── Templates/
│   └── Templater/
│       ├── TMDB Movie Template.md
│       ├── TMDB Show Template.md
│       └── TMDB Show Episode Template.md
└── .obsidian/
    └── plugins/
        └── custom-views/   # Plugin data and config
```

---

## How Custom Views works

The plugin lets you define **views**: each view has a filter rule (which notes it applies to), an HTML template, optional CSS, and optional JavaScript. When you open a matching note, the plugin renders your HTML instead of the default preview, with note frontmatter fields available as template variables.

### Viewing the Movies view config

The pre-configured Movies view lives in `.obsidian/plugins/custom-views/data.json`. You can also inspect and edit views from the plugin settings in Obsidian (Settings → Custom Views).

The Movies view has three parts:

**Filter rule** — matches any note where the `categories` frontmatter field contains `[[Movies]]`:

```
categories contains [[Movies]]
```

**HTML template** — uses `{{field}}` syntax to inject frontmatter values. For example:

```html
<p class="movie-title">{{file.basename}}</p>
<p class="movie-genre">{{genres|split:","|join:"·"}}</p>
<img src="{{backdrop|first}}" />
```

**JavaScript** — runs after render. The Movies view JS does two things: it samples the dominant color from the backdrop image and rebuilds all Obsidian CSS variables to match (so the entire card palette adapts to each movie's artwork), then fetches cast and trailers live from TMDB using the `tmdbId` frontmatter field.

---

## Setting up the Templater templates

The templates in `Templates/Templater/` automate creating movie and show notes by searching TMDB and writing all frontmatter for you.

### 1. Get a TMDB API key

Sign up at [themoviedb.org](https://www.themoviedb.org/) → Settings → API → create a key (free).

### 2. Add your key to the Movie template

Open `Templates/Templater/TMDB Movie Template.md` and replace the placeholder on line 5:

```js
const API_KEY = "<API_KEY>";
```

The Show and Episode templates already have a key hardcoded in this demo — replace those too if you're using this vault for real.

### 3. Configure Templater

In Obsidian settings → Templater, set the template folder to `Templates/Templater`.

### 4. Using the Movie template

1. Create a new note in `References/` (the name doesn't matter — the template will rename it).
2. Open the Templater command palette and run **Templater: Open insert template modal**.
3. Pick **TMDB Movie Template**.
4. A prompt appears: type a movie name (or leave blank to use the note's filename).
5. Pick from the search results, choose your rating (1–7), enter a watched date, and done.

The template writes the full frontmatter block and renames the note to the movie's title. Once saved, open the note in Reading view — Custom Views renders the card.

### What the Movie template populates

| Field | Source |
|---|---|
| `categories` | Always `[[Movies]]` (triggers the view) |
| `genres` | TMDB genre list |
| `directors`, `writers`, `cast` | TMDB credits (wiki-linked if a note exists for that person) |
| `cover` | Best-rated poster from TMDB |
| `backdrop` | Best-rated backdrop image from TMDB |
| `description` | TMDB plot overview |
| `year`, `runtime`, `published` | TMDB movie details |
| `rating` | Your 1–7 rating |
| `last` | Date you watched |
| `imdbId`, `tmdbId` | TMDB external IDs |

### Using the TV Show template

Same flow as movies. It searches TMDB's TV database and fills `categories: [[Shows]]`, `genre`, `director` (from `created_by`), `cast`, `cover`, `year`, `plot`, and `rating`.

### Using the Show Episode template

Use this to log individual episodes. It asks for show name → season → episode, then writes a note with `categories: [[Show episodes]]`, `show`, `season`, `episode`, `rating`, and `published` (air date). The note is automatically renamed to `Show Name S1 3 Episode Title` format.

---

## The sample notes

The `References/` folder contains a handful of pre-filled movie notes so you can see the Custom Views rendering immediately without running the templates. Open any of them in Reading view (or Live Preview) to see the movie card.

---

## Editing the view

To modify the Movies view or create a new one, go to Settings → Custom Views. From there you can edit the filter rules, HTML template, CSS, and JavaScript directly in Obsidian.

The `{{field}}` template syntax supports filters. A few used in this view:

- `{{backdrop|first}}` — first item in an array field
- `{{genres|split:","|join:"·"}}` — split a comma string, rejoin with a separator
- `{{cast|slice:0,4}}` — take the first N items
- `{{file.basename}}` — the note's filename without extension
- `{{file.content}}` — the body of the note (below frontmatter)

Custom JavaScript has access to `this` (the container element) for DOM manipulation and can make fetch calls — which is how the TMDB cast and trailers sections load live data each time you open a note.
