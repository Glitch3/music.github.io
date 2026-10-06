<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width, initial-scale=1">
<title>Sound map</title>
<style>
  body { font-family: Arial, sans-serif; background: #fff; color: #000; margin: 10px; max-width: 760px; }
  a { color: #000; }
  .row { margin: 8px 0; }
  ol.results > li { margin: 0 0 16px; }
  .meta, .small { font-size: 13px; }
  .tracks { margin: 6px 0; font-size: 14px; }
  button.link { background: none; border: 0; padding: 0; font: inherit; text-decoration: underline; cursor: pointer; color: #000; }
</style>
</head>
<body>
<div class="row">
  <b>Sound map</b>
  <form id="search" style="display:inline">
    <input id="q" type="text" placeholder="Type an artist" autocomplete="off" aria-label="Artist name">
    <button type="submit">Search</button>
  </form>
</div>
<div class="row">
  <label>Popularity:
    <select id="pop">
      <option value="Infinity">Any</option>
      <option value="1000000">Under 1,000,000 listeners</option>
      <option value="250000">Under 250,000 listeners</option>
      <option value="50000">Under 50,000 listeners</option>
      <option value="10000">Under 10,000 listeners</option>
    </select>
  </label>
</div>

<p id="status" aria-live="polite"></p>
<div id="results">
  <p>Search for an artist to see similar artists. Use Popularity to find smaller artists. Click an artist's name to see their top tracks.</p>
</div>

<script>
// Last.fm API key. Only the API key goes here, never the "shared secret".
const LASTFM_API_KEY = '490d2260c0a13cc21edc04650c9e5235';

const statusEl = document.getElementById('status');
const resultsEl = document.getElementById('results');
const $ = id => document.getElementById(id);
const cache = new Map();
let view = null;      // { title, items: [{ name, score, info }] }
let ticket = 0;

/* ---------- Last.fm ---------- */

async function lfm(params) {
  const qs = new URLSearchParams({ ...params, api_key: LASTFM_API_KEY, format: 'json' });
  const url = 'https://ws.audioscrobbler.com/2.0/?' + qs;
  if (cache.has(url)) return cache.get(url);
  const p = fetch(url).then(r => r.json()).then(d => {
    if (d.error) throw new Error(d.message || 'Last.fm returned an error.');
    return d;
  });
  cache.set(url, p);
  p.catch(() => cache.delete(url));
  return p;
}

async function getSimilar(artist, limit = 60) {
  const d = await lfm({ method: 'artist.getsimilar', artist, autocorrect: 1, limit });
  return {
    artist: d.similarartists['@attr'].artist,
    list: d.similarartists.artist.map(a => ({ name: a.name, score: +a.match }))
  };
}

async function getInfo(name) {
  try {
    const d = await lfm({ method: 'artist.getinfo', artist: name, autocorrect: 1 });
    const a = d.artist;
    return {
      listeners: +a.stats.listeners,
      tags: ((a.tags && a.tags.tag) || []).slice(0, 4).map(t => t.name),
      url: a.url
    };
  } catch { return { listeners: NaN, tags: [], url: null }; }
}

async function getTopTracks(name) {
  const d = await lfm({ method: 'artist.gettoptracks', artist: name, autocorrect: 1, limit: 10 });
  return d.toptracks.track.map(t => ({ name: t.name, listeners: +t.listeners }));
}

// run fn over items, a few at a time, to stay inside Last.fm's rate limits
async function mapLimit(items, n, fn, onProgress) {
  const out = new Array(items.length); let i = 0, done = 0;
  await Promise.all(Array.from({ length: n }, async () => {
    while (i < items.length) {
      const k = i++;
      out[k] = await fn(items[k]);
      onProgress && onProgress(++done, items.length);
    }
  }));
  return out;
}

/* ---------- loading ---------- */

async function addInfo(items, t) {
  const infos = await mapLimit(items, 5, x => getInfo(x.name), (d, n) => {
    if (t === ticket) statusEl.textContent = `Loading artist details ${d} of ${n}…`;
  });
  items.forEach((x, k) => x.info = infos[k]);
  return items;
}

async function loadArtist(artist) {
  const t = ++ticket;
  statusEl.textContent = `Loading artists similar to ${artist}…`;
  const sim = await getSimilar(artist);
  if (!sim.list.length) throw new Error(`Last.fm has no similar artists for "${artist}".`);
  const items = await addInfo(sim.list, t);
  if (t !== ticket) return;
  view = { title: `Artists similar to ${sim.artist}`, items };
  document.title = `${sim.artist} on Sound map`;
  render();
}

/* ---------- rendering ---------- */

const fmt = n => isNaN(n) ? 'unknown' : n.toLocaleString();
const ytLink = (artist, track) =>
  'https://www.youtube.com/results?search_query=' + encodeURIComponent(artist + ' ' + track);
const artistHash = name => '#/' + encodeURIComponent(name).replace(/%20/g, '+');

function render() {
  if (!view) return;
  const cap = +$('pop').value;
  const shown = view.items.filter(x => !(x.info.listeners >= cap));

  resultsEl.innerHTML = '';
  const h = document.createElement('h2');
  h.textContent = view.title;
  resultsEl.append(h);

  const hidden = view.items.length - shown.length;
  statusEl.textContent = hidden ? `${shown.length} shown, ${hidden} hidden by your filters.` : '';

  if (!shown.length) {
    const p = document.createElement('p');
    p.textContent = 'No artists match these filters. Choose a higher popularity limit.';
    resultsEl.append(p);
    return;
  }

  const ol = document.createElement('ol');
  ol.className = 'results';
  shown.forEach(x => {
    const li = document.createElement('li');

    const name = document.createElement('button');
    name.className = 'link';
    name.innerHTML = '<b></b>';
    name.firstChild.textContent = x.name;
    name.setAttribute('aria-expanded', 'false');

    const meta = document.createElement('div');
    meta.className = 'meta';
    const bits = [];
    bits.push(Math.round(x.score * 100) + '% match');
    bits.push(fmt(x.info.listeners) + ' listeners');
    if (x.info.tags.length) bits.push(x.info.tags.join(', '));
    meta.textContent = bits.join('. ') + '.';

    const links = document.createElement('div');
    links.className = 'small';
    const sim = document.createElement('a');
    sim.href = artistHash(x.name); sim.textContent = 'Similar artists';
    links.append(sim);
    if (x.info.url) {
      const lf = document.createElement('a');
      lf.href = x.info.url; lf.target = '_blank'; lf.rel = 'noopener'; lf.textContent = 'Last.fm page';
      links.append(' | ', lf);
    }

    const tracks = document.createElement('div');
    tracks.className = 'tracks';
    tracks.hidden = true;

    name.addEventListener('click', () => toggleTracks(x.name, name, tracks));
    li.append(name, meta, tracks, links);
    ol.append(li);
  });
  resultsEl.append(ol);
}

async function toggleTracks(artist, btn, box) {
  const open = box.hidden;
  box.hidden = !open;
  btn.setAttribute('aria-expanded', open);
  if (!open || box.dataset.loaded) return;
  box.textContent = 'Loading top tracks…';
  try {
    const list = await getTopTracks(artist);
    box.innerHTML = '<b>Top tracks</b>';
    const ol = document.createElement('ol');
    list.forEach(t => {
      const li = document.createElement('li');
      const a = document.createElement('a');
      a.href = ytLink(artist, t.name); a.target = '_blank'; a.rel = 'noopener';
      a.textContent = t.name;
      li.append(a, ` (${fmt(t.listeners)} listeners)`);
      ol.append(li);
    });
    box.append(ol);
    box.dataset.loaded = '1';
  } catch (e) {
    box.textContent = "Couldn't load top tracks. Click the name again to retry.";
    box.hidden = true; btn.setAttribute('aria-expanded', 'false');
  }
}

/* ---------- navigation ---------- */

function showError(err) {
  statusEl.textContent = /fetch|network|load failed/i.test(err.message)
    ? "Couldn't reach Last.fm. Check your connection. Last.fm is blocked inside the Claude preview, so open the hosted page instead."
    : err.message;
}

function route() {
  let m;
  if ((m = location.hash.match(/^#\/(.+)/))) {
    const name = decodeURIComponent(m[1].replace(/\+/g, ' '));
    $('q').value = name;
    loadArtist(name).catch(showError);
  }
}
function go(hash) { if (location.hash === hash) route(); else location.hash = hash; }

window.addEventListener('hashchange', route);
$('search').addEventListener('submit', e => {
  e.preventDefault();
  const v = $('q').value.trim();
  if (v) go(artistHash(v));
});
$('pop').addEventListener('change', render);

route();
</script>
</body>
</html>
