# QgisGrimpProjectSetup — Project Configuration

Collects the product-family definitions that describe how GrIMP products are
to be organised in a QGIS layer tree.  After configuration, call
`getProductFamilies()` to populate each family's product list from a set of
COG/shapefile URLs returned by `grimpfunc.cmrUrls`.

---

## Construction

```python
import grimpqgis as grimpq

myProjectSetup = grimpq.QgisGrimpProjectSetup()

# Optionally set a default local base path (for local file access)
myProjectSetup = grimpq.QgisGrimpProjectSetup(defaultBasePath='/data/grimp')
```

**Parameters:**
- `defaultBasePath` — root directory for local product files.  Leave as `None`
  (the default) when working with remote URLs.

---

## Product family template

Every product family is a dict with these keys:

| Key | Default | Description |
|-----|---------|-------------|
| `name` | (family key) | Display name in the QGIS layer tree |
| `topDir` | (family key) | Sub-directory under `basePath` for local search |
| `basePath` | `defaultBasePath` | Root directory for local files |
| `productFilePrefix` | `None` | Filename prefix used to identify files in URL/glob search.  Can be a list to match multiple prefixes. |
| `productPrefix` | `None` | Short label used in QGIS layer names (e.g. `'Vel'`, `'SAR'`) |
| `bands` | `[]` | List of band tokens to include (e.g. `['vv', 'vx', 'vy']`) |
| `fileType` | `'tif'` | File extension to match — **do not include a leading dot** |
| `byYear` | `True` | Group products under a year sub-group |
| `category` | `None` | Optional top-level category (e.g. `'Velocity'`, `'Imagery'`) |
| `displayOptions` | (see below) | Per-band colour table and opacity settings |
| `products` | `{}` | Populated by `getProductFamilies()` — do not set manually |

---

## Default display options

`defaultDisplayOptions()` returns a copy of these per-band defaults:

| Band | colorTable | minV | maxV | invert | opacity |
|------|-----------|------|------|--------|---------|
| `image` | None (default renderer) | — | — | False | 1.0 |
| `sigma0` | Greys | -20 | 5 | True | 1.0 |
| `gamma0` | Greys | -20 | 5 | True | 1.0 |
| `browse` | None (default renderer) | — | — | False | 0.7 |
| `vv` | Blues | 0 | 1500 | False | 1.0 |
| `vx` | RdBu | -1000 | 1000 | False | 1.0 |
| `vy` | RdBu | -1000 | 1000 | False | 1.0 |
| `ex` | YlGn | 0 | 20 | False | 1.0 |
| `ey` | YlGn | 0 | 20 | False | 1.0 |

`colorTable=None` means the QGIS default renderer is used (appropriate for
RGB browse images).  Any QGIS named colour ramp can be used for the others.

---

## Key methods

### `addProductFamilies`

```python
myProjectSetup.addProductFamilies(
    'Annual',                            # one or more family names
    productFilePrefix='GL_vel_mosaic_Annual',
    category='Velocity',
    productPrefix='Vel',
    bands=['browse', 'vv', 'vx', 'vy'],
    byYear=True,
    fileType='tif')
```

Add one or more product families.  Each name becomes a key in
`self.productFamilies`.  Keyword arguments must be valid template keys
(an error is printed for unknown keys).

Multiple prefixes can be passed as a list to a single family:
```python
myProjectSetup.addProductFamilies(
    'TSX_W69.10N',
    productFilePrefix=['TSX_W69.10N', 'TSX_W69.10N_v2'])
```

### `updateProductFamily`

```python
myProjectSetup.updateProductFamily('Monthly', bands=['browse', 'vv'])
```

Modify one or more template keys on an existing product family.

### `getProductFamilies`

```python
# Remote URL-based (most common):
myProjectSetup.getProductFamilies(urls=myUrls.getCogs())

# Local files (urls=None, uses defaultBasePath + topDir + glob):
myProjectSetup.getProductFamilies()
```

Populate the `products` dict for every registered family.  For URL-based
access each URL is tested against the family's `productFilePrefix`, `bands`,
and `fileType`, then wrapped in a GDAL `/vsicurl/` path.

**Note:** call this separately for COG and shapefile products:
```python
myProjectSetup.getProductFamilies(urls=myUrls.getCogs())    # tif
myProjectSetup.getProductFamilies(urls=myUrls.getShapes())  # shp
```

### `defaultDisplayOptions`

```python
displayOptions = myProjectSetup.defaultDisplayOptions()
displayOptions['vv']['colorTable'] = 'Inferno'
displayOptions['vv']['maxV'] = 4000
```

Return a deep copy of the per-band display defaults.  Modify the copy before
passing it to `addProductFamilies`.

### `productCategories`

```python
cats = myProjectSetup.productCategories()
# e.g. ['Imagery', 'TSX', 'Velocity']
```

Return the unique list of category names across all registered families.

---

## Customising individual product families after setup

Product families are plain dicts and can be edited directly:

```python
# Override bands on the 'Monthly' family after creation
myProjectSetup.productFamilies['Monthly']['bands'] = ['browse', 'vv']
```

---

## Full example — velocity mosaics

```python
velProperties = {
    'category': 'Velocity',
    'productPrefix': 'Vel',
    'bands': ['browse', 'vv', 'vx', 'vy'],
    'byYear': True,
    'fileType': 'tif',
}

if myUrls.checkIDs(['NSIDC-0725', 'NSIDC-0727', 'NSIDC-0731', 'NSIDC-0766']):
    for name in ['Annual', 'Quarterly', 'Monthly', 's1cycle']:
        myProjectSetup.addProductFamilies(
            name,
            productFilePrefix=f'GL_vel_mosaic_{name}',
            **velProperties)
    # Drop vx/vy from monthly to keep the tree manageable
    myProjectSetup.productFamilies['Monthly']['bands'] = ['browse', 'vv']

    # Populate products from URL search results
    myProjectSetup.getProductFamilies(urls=myUrls.getCogs())
```
