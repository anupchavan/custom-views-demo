<%*
/**
 * TMDB Show Episode → Frontmatter (Obsidian Templater)
 *
 * Flow:
 * - Ask for show name (fallback: current file name)
 * - TMDB TV search (top 15)
 * - Pick show
 * - If >1 season: pick season; else auto-pick the only season
 * - Fetch season episodes
 * - Pick episode
 * - Ask rating (1–7) via suggester
 * - Write YAML:
 *     categories, show, season, episode, rating, published, tags
 * - Rename file:
 *     "<show name> S<season> <episode> <episode name>"
 *   with ":" → " - " and other illegal filename chars removed
 */

const API_KEY = "083259692f1561db7b350d9649edb722"; // your TMDB key

//---------------------------------------------------------------------
// 0) Current file
//---------------------------------------------------------------------
const fileTitle = tp.file.title;

//---------------------------------------------------------------------
// 1) Ask for show name
//---------------------------------------------------------------------
let query = await tp.system.prompt("Enter TV show name (leave empty for file name):");
if (!query || !query.trim()) query = fileTitle;
query = query.trim();

//---------------------------------------------------------------------
// 2) Search TMDB (TV)
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
// 3) Pick show
//---------------------------------------------------------------------
const chosenShow = await tp.system.suggester(display, limited, true, "Pick a TV show");
if (!chosenShow) {
  tR += "Selection cancelled.";
  return;
}

const tvId = chosenShow.id;

//---------------------------------------------------------------------
// 4) Fetch show details (for seasons + language)
//---------------------------------------------------------------------
const detailsUrl =
  `https://api.themoviedb.org/3/tv/${tvId}` +
  `?api_key=${API_KEY}` +
  "&language=en-US";

let details;
try {
  details = await tp.web.request(detailsUrl);
} catch (e) {
  tR += `TMDB details error: ${e}`;
  return;
}

// Show title and language
const showTitle = (details.name || details.original_name || chosenShow.name || query || fileTitle).trim();
const langCode = (details.original_language || "").toLowerCase();

//---------------------------------------------------------------------
// Language tag (same map as main template)
//---------------------------------------------------------------------
const languageMap = {
  // --- Indian languages ---
  hi: "hindi",
  bn: "bengali",
  ta: "tamil",
  te: "telugu",
  kn: "kannada",
  ml: "malayalam",
  mr: "marathi",
  gu: "gujarati",
  pa: "punjabi",
  or: "odia",
  bho: "bhojpuri",
  sa: "sanskrit",
  ks: "kashmiri",
  as: "assamese",
  ur: "urdu",

  // --- Existing global languages ---
  en: "english",
  ja: "japanese",
  ko: "korean",
  es: "spanish",
  fr: "french",
  de: "german",
  it: "italian",
  ru: "russian",
  zh: "chinese",

  // --- Useful additional languages ---
  tr: "turkish",
  th: "thai",
  id: "indonesian",
  fa: "persian",
  ar: "arabic",
  pt: "portuguese",
  vi: "vietnamese",
  he: "hebrew",
};

const languageTag = languageMap[langCode] || (langCode || "");

//---------------------------------------------------------------------
// 5) Seasons
//---------------------------------------------------------------------
let seasons = Array.isArray(details.seasons) ? details.seasons : [];

// Filter out specials (season 0) and 0-episode seasons
seasons = seasons.filter(s => (s.season_number > 0) && (s.episode_count || 0) > 0);

if (!seasons.length) {
  tR += "No seasons with episodes found for this show.";
  return;
}

let chosenSeason;

// If only one season, auto-pick it; else ask user
if (seasons.length === 1) {
  chosenSeason = seasons[0];
} else {
  const seasonLabels = seasons.map(s => {
    const sn = s.season_number;
    const epCount = s.episode_count ?? "?";
    const air = s.air_date || "Unknown air date";
    return `Season ${sn} (${epCount} eps) – ${air}`;
  });

  chosenSeason = await tp.system.suggester(
    seasonLabels,
    seasons,
    true,
    "Pick a season"
  );

  if (!chosenSeason) {
    tR += "Season selection cancelled.";
    return;
  }
}

const seasonNumber = chosenSeason.season_number;

//---------------------------------------------------------------------
// 6) Fetch season episodes
//---------------------------------------------------------------------
const seasonUrl =
  `https://api.themoviedb.org/3/tv/${tvId}/season/${seasonNumber}` +
  `?api_key=${API_KEY}` +
  "&language=en-US";

let seasonDetails;
try {
  seasonDetails = await tp.web.request(seasonUrl);
} catch (e) {
  tR += `TMDB season error: ${e}`;
  return;
}

const episodes = seasonDetails?.episodes || [];
if (!episodes.length) {
  tR += "No episodes found for this season.";
  return;
}

//---------------------------------------------------------------------
// 7) Pick episode
//---------------------------------------------------------------------
const episodeLabels = episodes.map(ep => {
  const num = ep.episode_number;
  const name = ep.name || "Untitled";
  const date = ep.air_date || "Unknown air date";
  return `Ep ${num} – ${name} – ${date}`;
});

const chosenEpisode = await tp.system.suggester(
  episodeLabels,
  episodes,
  true,
  "Pick an episode"
);

if (!chosenEpisode) {
  tR += "Episode selection cancelled.";
  return;
}

const episodeNumber = chosenEpisode.episode_number;
const episodeName = (chosenEpisode.name || "").trim();
const episodeAirDate = chosenEpisode.air_date || "";

//---------------------------------------------------------------------
// 8) Ask for rating (1–7)
//---------------------------------------------------------------------
const ratingChoices = ["1", "2", "3", "4", "5", "6", "7"];
let rating = await tp.system.suggester(
  ratingChoices,
  ratingChoices,
  false,
  "Pick your rating (1–7)"
);
if (!rating) rating = "";

//---------------------------------------------------------------------
// 9) Rename file
//---------------------------------------------------------------------
// Base rename string: "<show name> S<season> <episode> <episode name>"
let targetTitle = `${showTitle} S${seasonNumber} ${episodeNumber} ${episodeName}`;

// Replace ":" with " - "
targetTitle = targetTitle.replace(/:/g, " - ");

// Strip other illegal filename characters
const illegalRe = /[\\/#%&{}<>*\?\$!'\"@+`|=]/g;
let sanitizedTitle = targetTitle.replace(illegalRe, "").trim();

// Fallback if somehow emptied
if (!sanitizedTitle) sanitizedTitle = fileTitle;

// Rename if changed
if (sanitizedTitle !== fileTitle) {
  await tp.file.rename(sanitizedTitle);
}

//---------------------------------------------------------------------
// 10) Build YAML
//---------------------------------------------------------------------
const today = tp.date.now("YYYY-MM-DD");

let lines = [];
lines.push("---");
lines.push("categories:");
lines.push('  - "[[Show episodes]]"');

lines.push(`show: "[[${showTitle}]]"`);
lines.push(`season: ${seasonNumber}`);
lines.push(`episode: ${episodeNumber}`);
lines.push(`rating: ${rating}`);
lines.push(`published: ${episodeAirDate}`);
lines.push(`created: ${today}`);

lines.push("---");

tR += lines.join("\n");
%>