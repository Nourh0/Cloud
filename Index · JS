const { addonBuilder, serveHTTP } = require('stremio-addon-sdk');

const PAGE_SIZE = 50;

const manifest = {
  id: 'community.nourcloud.archive',
  version: '1.0.0',
  name: 'Nour Cloud',
  description: 'أفلام قانونية ومجانية من Internet Archive (كلاسيكيات وملكية عامة)',
  resources: ['catalog', 'meta', 'stream'],
  types: ['movie'],
  idPrefixes: ['ia:'],
  catalogs: [
    {
      type: 'movie',
      id: 'ia-films',
      name: 'Nour Cloud - أفلام',
      extra: [{ name: 'search', isRequired: false }, { name: 'skip', isRequired: false }]
    }
  ]
};

const builder = new addonBuilder(manifest);

const first = (v) => (Array.isArray(v) ? v[0] : v);
const poster = (id) => `https://archive.org/services/img/${id}`;

async function getJSON(url) {
  const res = await fetch(url);
  if (!res.ok) throw new Error(`HTTP ${res.status} for ${url}`);
  return res.json();
}

// Catalog: browse or search films
builder.defineCatalogHandler(async ({ type, id, extra }) => {
  if (type !== 'movie' || id !== 'ia-films') return { metas: [] };

  let q = 'collection:feature_films AND mediatype:movies';
  if (extra && extra.search) {
    const term = extra.search.replace(/["():]/g, ' ').trim();
    if (term) q += ` AND title:(${term})`;
  }
  const skip = parseInt((extra && extra.skip) || '0', 10) || 0;
  const page = Math.floor(skip / PAGE_SIZE) + 1;

  const params = new URLSearchParams({ q, rows: String(PAGE_SIZE), page: String(page), output: 'json' });
  ['identifier', 'title', 'year', 'description'].forEach((f) => params.append('fl[]', f));
  params.append('sort[]', 'downloads desc');

  try {
    const data = await getJSON(`https://archive.org/advancedsearch.php?${params}`);
    const metas = (data.response.docs || []).map((d) => ({
      id: `ia:${d.identifier}`,
      type: 'movie',
      name: first(d.title) || d.identifier,
      poster: poster(d.identifier),
      releaseInfo: d.year ? String(first(d.year)) : undefined,
      description: first(d.description)
    }));
    return { metas };
  } catch (e) {
    console.error(e);
    return { metas: [] };
  }
});

// Meta: details for one film
builder.defineMetaHandler(async ({ type, id }) => {
  if (!id.startsWith('ia:')) return { meta: null };
  const ident = id.slice(3);
  try {
    const data = await getJSON(`https://archive.org/metadata/${encodeURIComponent(ident)}`);
    const m = data.metadata || {};
    return {
      meta: {
        id,
        type: 'movie',
        name: first(m.title) || ident,
        poster: poster(ident),
        background: poster(ident),
        description: first(m.description),
        releaseInfo: m.year ? String(first(m.year)) : undefined
      }
    };
  } catch (e) {
    console.error(e);
    return { meta: null };
  }
});

// Stream: direct MP4 links from the archive
builder.defineStreamHandler(async ({ type, id }) => {
  if (!id.startsWith('ia:')) return { streams: [] };
  const ident = id.slice(3);
  try {
    const data = await getJSON(`https://archive.org/metadata/${encodeURIComponent(ident)}`);
    const mp4s = (data.files || [])
      .filter((f) => /\.mp4$/i.test(f.name) && f.source !== 'metadata')
      .sort((a, b) => Number(b.size || 0) - Number(a.size || 0))
      .slice(0, 3);

    const streams = mp4s.map((f) => ({
      title: `Nour Cloud\n${f.name}`,
      url: `https://archive.org/download/${encodeURIComponent(ident)}/${encodeURIComponent(f.name)}`
    }));
    return { streams };
  } catch (e) {
    console.error(e);
    return { streams: [] };
  }
});

serveHTTP(builder.getInterface(), { port: process.env.PORT || 7000 });
console.log(`Nour Cloud addon running on port ${process.env.PORT || 7000}`);
