# Semantic KG Explorer for LIS — full-stack edition

A web application for searching, mapping and visualising Library and Information
Science (LIS) bibliographic knowledge graphs. The original single-file app has
been split into a **Node.js backend**, a **browser front end** and a **data layer**.

## Structure

```
lis-kg-explorer/
├── server/index.js                   Node.js HTTP server (no npm dependencies)
│                                       GET /api/health   service status
│                                       GET /api/dataset  dataset, gzip on the wire
│                                       GET /*            front-end files
├── public/index.html                 Front end: search, filters, KG, tree, AI Search, …
├── data/dataset.json.gz              Bibliographic dataset (27,825 records)
├── scripts/build-from-standalone.js  Regenerates data/ and public/ from the standalone HTML
├── test/server.test.js               Server tests (node:test)
├── test/query.test.js                Query engine tests: Boolean, wildcards, expansion
├── Dockerfile                        Container image
└── package.json
```

The front end runs in the browser. It loads the dataset once from `/api/dataset`
and does search, ranking, graph building and AI Search locally, so the core app
works without internet access once the server is running.

## Requirements

- Node.js 18 or later. There are no npm dependencies, so `npm install` is not needed.

## Run locally

```bash
npm start              # http://localhost:3000
PORT=8080 npm start    # another port
npm test               # automated tests
```

## Run with Docker

```bash
docker build -t lis-kg-explorer .
docker run --rm -p 3000:3000 lis-kg-explorer
```

## Rebuild the data and front end

`data/` and `public/index.html` are generated from the standalone HTML file:

```bash
npm run build -- /path/to/Semantic-KG-Explorer-LISJKG.html
```

## Search syntax (main search box)

| You type | Meaning |
|---|---|
| `digital libraries` | every term is required (AND is implied; unchanged from before) |
| `metadata OR "linked data"` | either side. Operators must be written in capitals |
| `repository NOT survey`, `repository -survey` | exclude records that match the term |
| `(ontology OR taxonomy) AND retrieval` | brackets group terms |
| `libr*`, `*graphy`, `*log*` | truncation: starts with, ends with, contains |
| `organi?ation` | `?` stands for exactly one character |
| `"digital lib*"` | wildcards also work inside quoted phrases |

Lowercase `and`, `or`, `not` are still ignored as stop words, so existing queries behave as before.
Plain queries without operators, wildcards or an expansion hit use the original ranking code unchanged.
The advanced search also accepts `*` and `?` in values and has two new match options, "ends with" and "contains text anywhere".

### Automatic query expansion

On by default; switch it off with the **Auto-expand query** checkbox under the search box
(the choice is remembered in the browser), or with **Search exactly as typed** in the notice above the results.
Terms are expanded with:

- abbreviation and variant pairs found in the dataset keywords, e.g. `Knowledge Organization (KO)`;
- synonyms imported in the Semantic Layer;
- a small built-in LIS thesaurus (OPAC, MARC, RDA, NLP, …);
- British/American spelling variants (`organisation`/`organization`, `catalogue`/`catalog`).

Expanded terms rank below exact matches (45% weight), are matched only in title, keywords and abstract,
are highlighted in blue, and are listed above the results. Evaluation tools and the Semantic AI search keep
using the unexpanded query.

Tests for the query engine are in `test/query.test.js` (`npm test`).

## Online features

- **Linked Open Data and Diamond OA** call public services (OpenAlex, Crossref,
  Wikidata, DOAJ and others). These need internet access; the rest of the app does not.
- **AI Search** works offline with the built-in semantic engine. For LLM-written
  answers, open AI Search → Settings and enter a key. The key is stored in the
  browser only and is sent straight to the provider. This server neither stores nor proxies it.
- **Graph store (Neo4j / GraphDB)** connects to the services you configure in the app.

## API

| Method | Path         | Response                                   |
|--------|--------------|--------------------------------------------|
| GET    | /api/health  | `{"status":"ok","uptime_seconds":…}`       |
| GET    | /api/dataset | JSON dataset, `Content-Encoding: gzip`     |

The dataset JSON has the shape `{ m: {…metadata…}, r: [[cites, authors, title, year, sourceIdx, publisherIdx, placeIdx, countryIdx, url, doi, issn, keywords, abstract], …] }`.

## Deployment notes

- Put the service behind a reverse proxy (nginx, Caddy) to add TLS.
- Responses include `X-Content-Type-Options: nosniff` and `Referrer-Policy: no-referrer`.
- The container runs as the non-root `node` user and includes a health check.
- Set `HOST=127.0.0.1` to listen on localhost only.
