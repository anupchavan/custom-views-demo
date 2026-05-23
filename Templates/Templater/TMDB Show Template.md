<%*
/**
 * TMDB TV Show → Frontmatter (Obsidian Templater)
 * FULL VERSION (with directors)
 */

const API_KEY = "083259692f1561db7b350d9649edb722"; // your key
const fileTitle = tp.file.title;

let query = await tp.system.prompt("Enter TV show name (leave empty for file name):");
if (!query || !query.trim()) query = fileTitle;
query = query.trim();

//---------------------------------------------------------------------
// 1) Search TMDB
//---------------------------------------------------------------------
const searchUrl =
  "https://api.themoviedb.org/3/search/tv" +
  `?api_key=${API_KEY}` +
  `&query=${encodeURIComponent(query)}` +
  "&include_adult=false&language=en-US&page=1";

let searchResponse;
try {
  searchResponse = await tp.web.request(searchUrl);
} catch (e) {
  tR += `TMDB search error: ${e}`;
  return;
}

const results = searchResponse?.results || [];
if (!results.length) {
  tR += `No results for "${query}".`;
  return;
}

const limited = results.slice(0, 15);
const display = limited.map(r => {
  const name = r.name || r.original_name || "Unknown";
  const year = (r.first_air_date || "").slice(0, 4) || "????";
  const lang = (r.original_language || "").toUpperCase();
  return `${name} (${year}) [${lang}]`;
});

//---------------------------------------------------------------------
// 2) Pick show
//---------------------------------------------------------------------
const chosen = await tp.system.suggester(display, limited, true, "Pick a TV show");
if (!chosen) {
  tR += "Selection cancelled.";
  return;
}

const tvId = chosen.id;

//---------------------------------------------------------------------
// 3) Fetch details, credits, and images SEPARATELY (Prevents payload crash)
//---------------------------------------------------------------------
const detailsUrl = `https://api.themoviedb.org/3/tv/${tvId}?api_key=${API_KEY}&language=en-US`;
const creditsUrl = `https://api.themoviedb.org/3/tv/${tvId}/credits?api_key=${API_KEY}&language=en-US`;
const imagesUrl  = `https://api.themoviedb.org/3/tv/${tvId}/images?api_key=${API_KEY}`;

let details, credits, images;
try {
  details = await tp.web.request(detailsUrl);
  credits = await tp.web.request(creditsUrl);
  images  = await tp.web.request(imagesUrl);
} catch (e) {
  tR += `TMDB fetch error: ${e}`;
  return;
}

//---------------------------------------------------------------------
// 4) EXTRACT FIELDS
//---------------------------------------------------------------------
const showTitle = (details.name || details.original_name || chosen.name || query || fileTitle).trim();
const yearStr = (details.first_air_date || "").slice(0, 4);
const year = yearStr || "";
const genres = (details.genres || []).map(g => g.name).filter(Boolean);

let plot = details.overview || "";
plot = plot.replace(/"/g, '\\"'); // escape double quotes for YAML

async function noteExists(name) {
  const path = `${name}.md`;
  try {
    return await tp.file.exists(path);
  } catch (e) {
    return false;
  }
}

//--- DIRECTORS ---
let directorNames = [];
if (Array.isArray(details.created_by)) {
  for (const p of details.created_by) {
    if (p && p.name) directorNames.push(p.name);
  }
}

const crew = credits?.crew || [];
for (const member of crew) {
  if (!member || !member.name) continue;
  const job = (member.job || "").toLowerCase();
  if (job === "director" || job === "series director") {
    directorNames.push(member.name);
  }
}

{
  const seenDir = new Set();
  directorNames = directorNames.filter(name => {
    if (seenDir.has(name)) return false;
    seenDir.add(name);
    return true;
  });
}

let directorsProcessed = [];
for (let i = 0; i < directorNames.length; i++) {
  const name = directorNames[i];
  if (i < 3) {
    directorsProcessed.push(`[[${name}]]`);
  } else {
    const exists = await noteExists(name);
    directorsProcessed.push(exists ? `[[${name}]]` : name);
  }
}

//--- CAST ---
const rawCast = (credits?.cast || [])
  .slice(0, 7)
  .map(c => c.name)
  .filter(Boolean);

let castProcessed = [];
for (let i = 0; i < rawCast.length; i++) {
  const actor = rawCast[i];
  if (i < 3) {
    castProcessed.push(`[[${actor}]]`);
  } else {
    const exists = await noteExists(actor);
    castProcessed.push(exists ? `[[${actor}]]` : actor);
  }
}

//--- LANGUAGE ---
const langCode = (details.original_language || "").toLowerCase();
const languageMap = {
  hi: "hindi", bn: "bengali", ta: "tamil", te: "telugu", kn: "kannada",
  ml: "malayalam", mr: "marathi", gu: "gujarati", pa: "punjabi", or: "odia",
  bho: "bhojpuri", sa: "sanskrit", ks: "kashmiri", as: "assamese", ur: "urdu",
  en: "english", ja: "japanese", ko: "korean", es: "spanish", fr: "french",
  de: "german", it: "italian", ru: "russian", zh: "chinese", tr: "turkish",
  th: "thai", id: "indonesian", fa: "persian", ar: "arabic", pt: "portuguese",
  vi: "vietnamese", he: "hebrew",
};
const languageTag = languageMap[langCode] || (langCode || "");

//--- COVER IMAGE ---
let coverURL = "";
const posters = images?.posters || [];

if (posters.length) {
  posters.sort((a, b) => (b.vote_count || 0) - (a.vote_count || 0));
  const bestPoster = posters[0];
  if (bestPoster && bestPoster.file_path) {
    coverURL = `https://image.tmdb.org/t/p/original${bestPoster.file_path}`;
  }
}

if (!coverURL && details.poster_path) {
  coverURL = `https://image.tmdb.org/t/p/original${details.poster_path}`;
}

//---------------------------------------------------------------------
// 5) Rating & File Rename
//---------------------------------------------------------------------
const ratingChoices = ["1", "2", "3", "4", "5", "6", "7"];
let rating = await tp.system.suggester(
  ratingChoices,
  ratingChoices,
  false,
  "Pick your rating (1–7)"
);
if (!rating) rating = "";

const colonFixed = showTitle.replace(/:/g, " - ");
const illegalRe = /[\\/#%&{}<>*\?\$!'\"@+`|=]/g;
let sanitizedTitle = colonFixed.replace(illegalRe, "").trim();

if (!sanitizedTitle) sanitizedTitle = fileTitle;

const requiresAlias = sanitizedTitle !== showTitle;
if (sanitizedTitle !== fileTitle) {
  await tp.file.rename(sanitizedTitle);
}

//---------------------------------------------------------------------
// 6) BUILD YAML
//---------------------------------------------------------------------
const today = tp.date.now("YYYY-MM-DD");

let lines = [];
lines.push("---");
lines.push("categories:");
lines.push('  - "[[Shows]]"');

if (genres.length) {
  lines.push("genre:");
  for (const g of genres) {
    lines.push(`  - "[[${g}]]"`);
  }
} else {
  lines.push("genre: []");
}

lines.push(`year: ${year}`);
lines.push(`plot: "${plot}"`);

if (directorsProcessed.length) {
  lines.push("director:");
  for (const d of directorsProcessed) {
    lines.push(`  - "${d}"`);
  }
} else {
  lines.push("director: []");
}

if (castProcessed.length) {
  lines.push("cast:");
  for (const c of castProcessed) {
    lines.push(`  - "${c}"`);
  }
} else {
  lines.push("cast: []");
}

lines.push(`rating: ${rating}`);

if (coverURL) {
  lines.push(`cover: "${coverURL}"`);
} else {
  lines.push("cover: []");
}

if (languageTag) {
  lines.push(`language: "${languageTag}"`);
}

lines.push(`created: ${today}`);
lines.push(`last: ${today}`);

if (requiresAlias) {
  const aliasEscaped = showTitle.replace(/"/g, '\\"');
  lines.push("aliases:");
  lines.push(`  - "${aliasEscaped}"`);
}

lines.push("---");
lines.push("");
lines.push("![[Show episodes.base#Show]]");

tR += lines.join("\n");
%>