# supporters-map
Map of general locations of Donors

# Supporters Map: setup

The page (`index.html`) reads a public file called `feed.json` and draws it. Zapier builds that file from REACH. No names or street addresses ever go into the file.

```
REACH ──(Zap A, per supporter)──> geocode + round + tag church/individual ──> Zapier Tables (private)
Zapier Tables ──(Zap B, scheduled)──> feed.json (public) ──> index.html on your website
```

## 0. Before you start: confirm what REACH sends

The REACH docs I could find list "New Supporter", "New Donation" and "Updated Donation" triggers, but they do not say which fields a supporter record carries. In Zap A, test the trigger and check that the sample record contains:

- a mailing address (street, city, state, ZIP), and
- group membership (the group named `church`).

If groups are missing from the trigger, ask REACH support (support@reachapp.co) how to get group membership into Zapier. A common workaround is a "Find Supporter" step, or a supporter tag that mirrors the group.

## 1. Zapier Tables: create a table called `map_points`

Fields: `supporter_id` (text), `type` (text), `lat` (number), `lng` (number), `city` (text), `state` (text).
Do not add name, email or street address fields.

## 2. Zap A: one supporter in, one anonymized point out

1. **Trigger:** REACH, New Supporter. Add an Updated Supporter trigger as a second Zap if REACH offers one, so address changes are picked up.
2. **Filter:** continue only if the address and ZIP are not empty.
3. **Webhooks by Zapier, GET.** Free US Census geocoder, no key needed (US addresses only):
   `https://geocoding.geo.census.gov/geocoder/locations/onelineaddress?benchmark=Public_AR_Current&format=json&address=` followed by street, city, state and ZIP from step 1, URL-encoded.
4. **Code by Zapier (JavaScript).** Inputs: `resp` (the webhook response), `groups` (REACH group field), `id`, `city`, `state`:

```js
const match = (inputData.resp.result?.addressMatches || [])[0];
if (!match) return {};                       // address not found: stop here
const r2 = n => Math.round(n * 100) / 100;   // ~1 km rounding, done BEFORE storing
const isChurch = String(inputData.groups || "").toLowerCase()
  .split(/[,;|]/).map(s => s.trim()).includes("church");
return {
  supporter_id: String(inputData.id),
  type: isChurch ? "church" : "individual",
  lat: r2(match.coordinates.y),
  lng: r2(match.coordinates.x),
  city: inputData.city || "",
  state: inputData.state || ""
};
```

5. **Zapier Tables: Create or Update Record** in `map_points`, matching on `supporter_id`.

Tip: if the Census geocoder misses many addresses, swap step 3 for Google Maps or Mapbox geocoding (needs a key).

## 3. Zap B: publish the public file

Zapier Tables is private, and a web page cannot read it without exposing your Zapier credentials, so the page reads a separate public file instead.

1. **Trigger:** Schedule by Zapier, every day (or hour).
2. **Zapier Tables: Find Records** in `map_points` (return all rows).
3. **Code by Zapier** turns the rows into JSON text:

```js
const rows = inputData.rows;  // map the Find Records output into an array of rows
const feed = rows.map(r => ({
  type: r.type, lat: r.lat, lng: r.lng, city: r.city, state: r.state
}));
return { json: JSON.stringify(feed) };
```

4. **Publish `feed.json`** somewhere public that allows other websites to read it (CORS). Easiest free option: a GitHub repository with GitHub Pages turned on, using Zapier's GitHub "Create or Update File" action. Any host Zapier can write to will do (S3, your website's file storage).

Then open `index.html`, set `CONFIG.FEED_URL` to the public address of `feed.json`, and upload `index.html` to your website (or the same GitHub Pages site). It can also be shown inside an existing page with an iframe.

## 4. Privacy checklist for a public map

- Coordinates are rounded to about 1 km in Zap A, and again in the page as a second safety net.
- Click popups show only "Church partner" or "Individual supporter" plus city and state.
- Decide whether donors should be told (privacy policy or a short notice) that approximate locations may appear on a public map. Consider an opt-out field in REACH that Zap A's filter respects.
- When a supporter is deleted or opts out in REACH, delete their row in `map_points`. Zap A only adds and updates.
- Churches are organizations, so you could show their exact address if you want. Skip the rounding for `type == "church"` in the Code step, but check that "church" group members are not people's home addresses.
- Rural areas: one dot at 1 km precision can still point to a single household. If that worries you, round to 1 decimal (about 10 km) in `r2`.

## 5. Hosting on GitHub Pages and showing it on Wix

**GitHub Pages (free)**

1. On github.com, create a repository, for example `supporters-map`. Set it to Public (free GitHub Pages needs that; the data is already anonymized).
2. Upload `index.html` and `sample-feed.json` renamed to `feed.json`.
3. In `index.html`, change `FEED_URL: ""` to `FEED_URL: "feed.json"`. Because the page and the data file live in the same place, no extra permissions are needed.
4. Repository Settings > Pages > Source: "Deploy from a branch", branch `main`, folder `/ (root)`. After a minute the page is live at `https://YOUR-NAME.github.io/supporters-map/`.
5. In Zap B, make the last step a GitHub action that creates or updates the file `feed.json` in that repository (Zapier connects with your GitHub login). Replace the example `feed.json` with the real feed on the first run.

**Wix**

1. In the Wix editor, go to Add Elements > Embed Code > Embed a Site (the element may also be called "HTML iframe").
2. Choose "Website address" and paste your GitHub Pages address.
3. Stretch the element to the size you want, and check the mobile view of your page and resize it there too.
4. Publish your Wix site. Updates to the map appear automatically, because Wix only displays the GitHub page.
