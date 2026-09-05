# unity-tax-locator-data

US ZIP code centroids for the Unity Tax Offices locator. Served to the browser
via jsDelivr so the office locator can convert a ZIP code into coordinates
without calling a geocoding API.

---

## Why this exists

The locator needs to turn a 5-digit ZIP code into a latitude/longitude so it
can measure distance to each office. That could be done with a geocoding API,
but ZIP centroids are a finite public dataset that changes very slowly, so
shipping the data is cheaper, faster and has no uptime dependency.

---

## Data source and licence

Data comes from the **GeoNames postal code dump**, file `US.zip`:

<https://download.geonames.org/export/zip/>

GeoNames is licensed **CC BY 4.0**. Attribution is required: a link to
<https://www.geonames.org> must appear on any site using this data. On the
Unity Tax site this sits in the footer, next to the map attribution.

GeoNames is used rather than the US Census ZCTA file because it also covers
PO Box-only ZIPs, "unique" ZIPs assigned to single organisations, and APO/FPO
military ZIPs. The Census file omits those, which would make some valid ZIPs
return "not found".

---

## Files

| Path | Contents |
|---|---|
| `v1/us-zip-centroids.min.json` | 41,488 ZIP codes, 3 decimal places (~110 m precision) |

### Format

A flat JSON object. Key is the 5-digit ZIP as a **string** (so leading zeros
survive). Value is `[latitude, longitude]`.

```json
{"64801":[37.097,-94.505],"11697":[40.559,-73.907],"04102":[43.66,-70.29]}
```

Roughly 1 MB raw, about 187 KB over the wire after Brotli compression.

---

## How the site uses it

```
https://cdn.jsdelivr.net/gh/dilshadkhaliq/unity-tax-locator-data@v1.0.0/us-zip-centroids.json
```

The URL is set in `CONFIG.zipDataUrl` in the locator script, in Webflow under
Page Settings → Before `</body>` tag on the office locator page.

**Always reference a git tag, never `@latest` or `@main`.** Tagged URLs are
immutable on jsDelivr and cached permanently. Branch URLs get a long max-age
and need manual purging, which is rate limited.

---

## Updating

### How often

**Once a year is plenty.** A calendar reminder each January, ahead of tax
season, is the right cadence.

There are roughly 41,500 active US ZIP codes, and USPS adds only about
**10–20 new 5-digit ZIPs per year** — under 0.05% drift. The commonly quoted
"~2,085 ZIP changes per year" figure is almost entirely ZIP+4 churn, which
this dataset does not use. New ZIPs also take years to come into full use, so
a slightly stale file causes no practical problem.

### Versioning rules
- **Always cut a new git tag**: `v2.0.0`.
- Update `CONFIG.zipDataUrl` in Webflow to the new tag, then publish.
- The old URL keeps working throughout, so there is no broken window.

### Regeneration steps

1. Download `US.zip` from <https://download.geonames.org/export/zip/>
2. Unzip it to get `US.txt` (tab-delimited, no header)
3. Run the script below
4. Commit the output to a new `vN/` folder
5. Create a release tagged `vN.0.0`
6. Update `CONFIG.zipDataUrl` in the Webflow page settings and publish

### Regeneration script

```python
# build_zip_centroids.py
# Usage: python3 build_zip_centroids.py US.txt us-zip-centroids.json

import csv, json, os, sys

src, dest = sys.argv[1], sys.argv[2]

# GeoNames US.txt columns (tab-delimited, no header):
#  0 country  1 postal_code  2 place_name  3 admin1  4 admin1_code
#  5 admin2   6 admin2_code  7 admin3      8 admin3_code
#  9 latitude 10 longitude  11 accuracy

centroids = {}
with open(src, encoding='utf-8') as f:
    for row in csv.reader(f, delimiter='\t'):
        zip_code = row[1]
        if zip_code in centroids:
            continue                     # keep the first row for a duplicate ZIP
        if not row[9] or not row[10]:
            continue                     # skip rows with no coordinates
        centroids[zip_code] = [round(float(row[9]), 3), round(float(row[10]), 3)]

os.makedirs(os.path.dirname(dest), exist_ok=True)
with open(dest, 'w') as f:
    json.dump(centroids, f, separators=(',', ':'))

print('%d ZIP codes -> %s (%d KB)' % (
    len(centroids), dest, os.path.getsize(dest) // 1024))
```

**3 decimal places is deliberate.** That is about 110 m of precision, far more
than a 10-mile radius search needs, and it saves roughly 36 KB compressed over
4 decimals.

### After regenerating, check

- Entry count is in the expected range (~41,000–42,000)
- A ZIP with a leading zero still works: `data["04102"]` should return a point
- Every current office ZIP resolves — export the ZIP column from the Webflow
  Tax Offices collection and confirm each one is present

---

## Related

- Locator script lives in Webflow: office locator page → Page Settings →
  Before `</body>` tag
- Office data lives in the Webflow CMS collection **Tax Offices** (`offices`)
