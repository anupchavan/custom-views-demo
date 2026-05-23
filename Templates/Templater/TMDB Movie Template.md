<%*
/**
 * TMDB Movie → Frontmatter (Obsidian Templater)
 */
const API_KEY = "<API_KEY>";
//---------------------------------------------------------------------
// 0) Current file
//---------------------------------------------------------------------
const fileTitle = tp.file.title;
//---------------------------------------------------------------------
// 1) Ask for movie name
//---------------------------------------------------------------------
let query = await tp.system.prompt("Enter movie name (leave empty for file name):");
if (!query || !query.trim()) query = fileTitle;
query = query.trim();
//---------------------------------------------------------------------
// 2) Search TMDB (Movie)
//---------------------------------------------------------------------
const searchUrl =
  "https://api.themoviedb.org/3/search/movie" +
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
  const title = r.title || r.original_title || "Unknown";
  const year = (r.release_date || "").slice(0, 4) || "????";
  const lang = (r.original_language || "").toUpperCase();
  return `${title} (${year}) [${lang}]`;
});
//---------------------------------------------------------------------
// 3) Pick movie
//---------------------------------------------------------------------
const chosenMovie = await tp.system.suggester(display, limited, true, "Pick a movie");
if (!chosenMovie) {
  tR += "Selection cancelled.";
  return;
}
const movieId = chosenMovie.id;
//---------------------------------------------------------------------
// 4) Fetch movie details + credits + images + external_ids SEPARATELY
//---------------------------------------------------------------------
const detailsUrl = `https://api.themoviedb.org/3/movie/${movieId}?api_key=${API_KEY}&language=en-US`;
const creditsUrl = `https://api.themoviedb.org/3/movie/${movieId}/credits?api_key=${API_KEY}&language=en-US`;
const imagesUrl  = `https://api.themoviedb.org/3/movie/${movieId}/images?api_key=${API_KEY}`;
const extIdsUrl  = `https://api.themoviedb.org/3/movie/${movieId}/external_ids?api_key=${API_KEY}`;
let details, credits, images, externalIds;
try {
  details     = await tp.web.request(detailsUrl);
  credits     = await tp.web.request(creditsUrl);
  images      = await tp.web.request(imagesUrl);
  externalIds = await tp.web.request(extIdsUrl);
} catch (e) {
  tR += `TMDB details error: ${e}`;
  return;
}
//---------------------------------------------------------------------
// 5) EXTRACT FIELDS
//---------------------------------------------------------------------
const movieTitle = (details.title || details.original_title || chosenMovie.title || query || fileTitle).trim();
const yearStr = (details.release_date || "").slice(0, 4);
const year = yearStr || "";
const genres = (details.genres || []).map(g => g.name).filter(Boolean);
let plot = details.overview || "";
plot = plot.replace(/"/g, '\\"');
const runtime = details.runtime || "";
const published = details.release_date || "";
const imdbId = details.imdb_id || (externalIds && externalIds.imdb_id) || "";
const tmdbId = movieId;
const langCode = (details.original_language || "").toLowerCase();
//---------------------------------------------------------------------
// Language tag
//---------------------------------------------------------------------
const languageMap = {
  hi: "hindi", bn: "bengali", ta: "tamil", te: "telugu", kn: "kannada",
  ml: "malayalam", mr: "marathi", gu: "gujarati", pa: "punjabi", or: "odia",
  bho: "bhojpuri", sa: "sanskrit", ks: "kashmiri", as: "assamese", ur: "urdu",
  en: "english", ja: "japanese", ko: "korean", es: "spanish", fr: "french",
  de: "german", it: "italian", ru: "russian", zh: "chinese",
  tr: "turkish", th: "thai", id: "indonesian", fa: "persian", ar: "arabic",
  pt: "portuguese", vi: "vietnamese", he: "hebrew",
};
const languageTag = languageMap[langCode] || (langCode || "");
//---------------------------------------------------------------------
// Helper: check if a note exists
//---------------------------------------------------------------------
const PERSON_FOLDERS = ["References", ""];
async function noteExists(name) {
  const fileName = `${name}.md`;
  for (const folder of PERSON_FOLDERS) {
    const path = folder ? `${folder}/${fileName}` : fileName;
    try {
      const exists = await tp.file.exists(path);
      if (exists) return true;
    } catch (e) {}
  }
  return false;
}
//---------------------------------------------------------------------
// 5A) DIRECTORS
//---------------------------------------------------------------------
let directorNames = [];
const crew = credits?.crew || [];
for (const member of crew) {
  if (!member || !member.name) continue;
  if ((member.job || "").toLowerCase() === "director") directorNames.push(member.name);
}
{ const seen = new Set(); directorNames = directorNames.filter(n => !seen.has(n) && seen.add(n)); }
let directorsProcessed = [];
for (let i = 0; i < directorNames.length; i++) {
  const name = directorNames[i];
  if (i < 3) { directorsProcessed.push(`[[${name}]]`); }
  else { const exists = await noteExists(name); directorsProcessed.push(exists ? `[[${name}]]` : name); }
}
//---------------------------------------------------------------------
// 5B) WRITERS
//---------------------------------------------------------------------
let writerNames = [];
for (const member of crew) {
  if (!member || !member.name) continue;
  const dept = (member.department || "").toLowerCase();
  const job = (member.job || "").toLowerCase();
  if (dept === "writing" || job.includes("writer") || job.includes("screenplay") || job.includes("story") || job.includes("teleplay")) {
    writerNames.push(member.name);
  }
}
{ const seenW = new Set(); writerNames = writerNames.filter(n => !seenW.has(n) && seenW.add(n)); }
let writersProcessed = [];
for (let i = 0; i < writerNames.length; i++) {
  const name = writerNames[i];
  if (i < 3) { writersProcessed.push(`[[${name}]]`); }
  else { const exists = await noteExists(name); writersProcessed.push(exists ? `[[${name}]]` : name); }
}
//---------------------------------------------------------------------
// 5C) CAST
//---------------------------------------------------------------------
const rawCast = (credits?.cast || []).slice(0, 8).map(c => c.name).filter(Boolean);
let castProcessed = [];
for (let i = 0; i < rawCast.length; i++) {
  const actor = rawCast[i];
  if (i < 3) { castProcessed.push(`[[${actor}]]`); }
  else { const exists = await noteExists(actor); castProcessed.push(exists ? `[[${actor}]]` : actor); }
}
//---------------------------------------------------------------------
// 5D) COVER IMAGE (most popular poster)
//---------------------------------------------------------------------
let coverURL = "";
const posters = images?.posters || [];
if (posters.length) {
  posters.sort((a, b) => (b.vote_count || 0) - (a.vote_count || 0));
  const bestPoster = posters[0];
  if (bestPoster?.file_path) coverURL = `https://image.tmdb.org/t/p/original${bestPoster.file_path}`;
}
if (!coverURL && details.poster_path) {
  coverURL = `https://image.tmdb.org/t/p/original${details.poster_path}`;
}
//---------------------------------------------------------------------
// 5E) BACKDROP IMAGE (most popular backdrop)
//---------------------------------------------------------------------
let backdropURL = "";
const backdrops = images?.backdrops || [];
if (backdrops.length) {
  backdrops.sort((a, b) => (b.vote_count || 0) - (a.vote_count || 0));
  const bestBackdrop = backdrops[0];
  if (bestBackdrop?.file_path) backdropURL = `https://image.tmdb.org/t/p/original${bestBackdrop.file_path}`;
}
if (!backdropURL && details.backdrop_path) {
  backdropURL = `https://image.tmdb.org/t/p/original${details.backdrop_path}`;
}
//---------------------------------------------------------------------
// 6) RATING (Suggester 1–7)
//---------------------------------------------------------------------
const ratingChoices = ["1", "2", "3", "4", "5", "6", "7"];
let rating = await tp.system.suggester(ratingChoices, ratingChoices, false, "Pick your rating (1–7)");
if (!rating) rating = "";
//---------------------------------------------------------------------
// 6.5) LAST DATE (Natural Language Dates)
//---------------------------------------------------------------------
const nldatesPlugin = tp.app.plugins.getPlugin("nldates-obsidian");
const presets = ["Today", "Yesterday", "Tomorrow"];
let lastDateMoment;
const picked = await tp.system.suggester(presets, presets, false, "Last watched date (Esc = type)");
if (picked) {
  lastDateMoment = nldatesPlugin.parseDate(picked)?.moment;
} else {
  const typed = await tp.system.prompt("Enter last watched date (natural language or YYYY-MM-DD)", "today");
  if (typed) lastDateMoment = nldatesPlugin.parseDate(typed)?.moment;
}
if (!lastDateMoment) lastDateMoment = nldatesPlugin.parseDate("today").moment;
const lastDate = lastDateMoment.format("YYYY-MM-DD");
//---------------------------------------------------------------------
// 7) FILE RENAME + ALIASES
//---------------------------------------------------------------------
const colonFixed = movieTitle.replace(/:/g, " - ");
const illegalRe = /[\/#%&{}<>*?$!'"@+`|=]/g;
let sanitizedTitle = colonFixed.replace(illegalRe, "").trim();
if (!sanitizedTitle) sanitizedTitle = fileTitle;
const requiresAlias = sanitizedTitle !== movieTitle;
if (sanitizedTitle !== fileTitle) await tp.file.rename(sanitizedTitle);
//---------------------------------------------------------------------
// 8) BUILD YAML
//---------------------------------------------------------------------
const today = tp.date.now("YYYY-MM-DD");
let lines = [];
lines.push("---");
lines.push("categories:");
lines.push('  - "[[Movies]]"');
// Genres
if (genres.length) {
  lines.push("genre:");
  for (const g of genres) lines.push(`  - "[[${g}]]"`);
} else {
  lines.push("genres: []");
}
// Directors
if (directorsProcessed.length) {
  lines.push("directors:");
  for (const d of directorsProcessed) lines.push(`  - "${d}"`);
} else {
  lines.push("directors: []");
}
// Writers
if (writersProcessed.length) {
  lines.push("writers:");
  for (const w of writersProcessed) lines.push(`  - "${w}"`);
} else {
  lines.push("writers: []");
}
// Cast
if (castProcessed.length) {
  lines.push("cast:");
  for (const c of castProcessed) lines.push(`  - "${c}"`);
} else {
  lines.push("cast: []");
}
// Cover
lines.push(coverURL ? `cover: "${coverURL}"` : "cover: []");
// Backdrop
if (backdropURL) {
  lines.push("backdrop:");
  lines.push(`  - "${backdropURL}"`);
} else {
  lines.push("backdrop: []");
}
// Description
lines.push(`description: "${plot}"`);
lines.push(`year: ${year}`);
lines.push(`rating: ${rating}`);
lines.push(`runtime: ${runtime}`);
lines.push(`published: ${published}`);
lines.push(`created: ${today}`);
lines.push(`last: ${lastDate}`);
lines.push(`imdbId: ${imdbId}`);
lines.push(`tmdbId: ${tmdbId}`);
lines.push("via:");
// Aliases
if (requiresAlias) {
  const aliasEscaped = movieTitle.replace(/"/g, '\\"');
  lines.push("aliases:");
  lines.push(`  - "${aliasEscaped}"`);
}
lines.push("---");
tR += lines.join("\n");
%>