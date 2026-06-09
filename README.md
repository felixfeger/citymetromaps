# # LC County GPS

A full GIS/navigation system for Lego City County — map editor + customer trip planner, built on Cloudflare Workers + D1 + Pages.

-----

## Architecture

```
GitHub Repo (this)
  ├── index.html      — Customer trip planner (public)
  ├── editor.html     — Admin map editor (restricted access)
  ├── worker.js       — Cloudflare Worker API
  └── schema.sql      — D1 database schema
```

**Cloudflare Stack:**

- **D1 Database** → stores roads, areas, addresses, transit stops/routes
- **Worker** → REST API (`/api/*`)
- **Pages** → serves the HTML frontends from this repo

-----

## Setup Guide

### Step 1 — Create D1 Database

1. Go to **Cloudflare Dashboard → Workers & Pages → D1**
1. Click **Create database**, name it `lcgps`
1. Open the database → go to **Console**
1. Paste the entire contents of `schema.sql` and click **Execute**

-----

### Step 2 — Create the Worker

1. Go to **Workers & Pages → Create → Worker**
1. Name it `lcgps-api`
1. Click **Edit Code** and paste the entire contents of `worker.js`
1. Click **Deploy**

**Bind D1 to the Worker:**

1. Open the worker → **Settings → Bindings**
1. Add a **D1 Database** binding:
- Variable name: `DB`
- Select your `lcgps` database
1. Click **Save and deploy**

Note your Worker URL: `https://lcgps-api.YOUR_SUBDOMAIN.workers.dev`

-----

### Step 3 — Update API URL in HTML files

In **both** `index.html` and `editor.html`, find this line near the top of the `<script>` tag:

```js
: 'https://YOUR_WORKER.workers.dev'; // ← UPDATE THIS
```

Replace `YOUR_WORKER.workers.dev` with your actual Worker URL, e.g.:

```js
: 'https://lcgps-api.felixfeger46.workers.dev';
```

-----

### Step 4 — Deploy to Cloudflare Pages

1. Push this repo to GitHub
1. Go to **Workers & Pages → Create → Pages → Connect to Git**
1. Select your repo
1. Build settings:
- **Framework preset:** None
- **Build command:** (leave blank)
- **Build output directory:** `/` (root)
1. Click **Save and Deploy**

Your site will be live at `https://YOUR-PROJECT.pages.dev`

- `https://YOUR-PROJECT.pages.dev/` → Customer trip planner
- `https://YOUR-PROJECT.pages.dev/editor.html` → Map editor

-----

## Using the Map Editor

### Drawing Roads

1. Open `editor.html`
1. In the **Draw** tab, click **Road**
1. Set the road type (Motorway, Primary, Secondary, Residential, Footpath, Cycleway)
1. Give it a name (e.g. “Main Street”)
1. Click **▷ Start Drawing**
1. Click on the map to place points along the road
1. Double-click or click **✓ Finish** to save

### Drawing Areas (Water, Parks, Buildings)

- Same as roads but choose **Water**, **Park**, or **Building**
- Draw at least 3 points to form a polygon
- The area fills automatically

### Drawing Rail Lines

- Choose **Rail** → select type (Subway, Tram, Commuter, Monorail)
- Draw like a road — will render as a dashed line

### Placing Addresses

1. Go to the **Addresses** tab
1. Click **📍 Click Map to Place Pin**
1. Click on the map (ideally on or near a road)
1. Fill in: House number, Street name, optional Unit/District/Postcode
1. Select type (Residential, Commercial, Government, Landmark)
1. Click **Save Address**

### Transit Stops & Routes

1. Go to the **Transit** tab
1. Click **🚏 Click Map to Place Stop** → click on the map
1. Fill in name, stop code, type, and which lines serve it
1. To create a **Route**: enter name, type, color, and the comma-separated stop IDs in order

-----

## Routing Logic

The Worker builds a graph from all road elements and runs **Dijkstra’s algorithm** to find the shortest path. Each mode applies different speed weights:

|Mode   |Motorway|Primary|Secondary|Residential|
|-------|--------|-------|---------|-----------|
|Driving|fastest |fast   |normal   |slow       |
|Walking|blocked |normal |normal   |normal     |
|Cycling|blocked |normal |normal   |normal     |

**Transit mode** finds the nearest stop to origin/destination and looks for a matching route.

Routing works best when:

- Roads are drawn as connected polylines (endpoints touch)
- The whole county road network is drawn in the editor

-----

## Tips

- The editor uses a **dark CartoDB** basemap — good for seeing your drawn elements
- The customer view uses **CartoDB Voyager** — clean and readable
- All elements are stored in D1, so they persist and both pages read from the same data
- You can protect `editor.html` with **Cloudflare Access** (Zero Trust) so only you can access it