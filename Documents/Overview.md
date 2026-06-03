# grimpqgis — Package Overview

grimpqgis builds [QGIS](https://qgis.org/) project files that remotely browse
[GrIMP](https://nsidc.org/data/measures/grimp) products stored as Cloud
Optimised GeoTIFFs (COGs) at NSIDC.  The resulting `.qgs` project file lets
you view velocity fields, SAR images, and terminus positions in QGIS without
downloading the data — QGIS fetches only the tiles you zoom to.

It works in tandem with **grimpfunc**:
- `grimpfunc.NASALogin` handles EarthData authentication (writes `~/.netrc`
  and creates the GDAL cookie file).
- `grimpfunc.cmrUrls` searches the NSIDC CMR catalog and returns COG/shapefile
  URLs.
- `grimpqgis` takes those URLs and builds the QGIS layer tree.

---

## Import convention

```python
import grimpqgis as grimpq
```

---

## Classes

| Class | Description |
|-------|-------------|
| [`QgisGrimpProjectSetup`](QgisGrimpProjectSetup.md) | Define and populate product families from URL lists |
| [`QgisGrimpProject`](QgisGrimpProject.md) | Build the QGIS layer tree and save the project |

---

## QGIS layer hierarchy

Products are organised in a tree of three optional levels:

```
Category (e.g., "Velocity" or "Imagery")   ← optional
  └─ Product Family (e.g., "Annual", "Image Mosaics")
       └─ Year (e.g., "2020")               ← byYear=True
            └─ ProductPrefix-YYYY-mm-dd.YYYY-mm-dd
                 ├─ band1  (e.g., vv, vx)   ← multi-band
                 └─ band2
```

For single-band product families the date string is at the top level (no
per-band sub-group):

```
Category
  └─ Product Family
       └─ Year
            └─ ProductPrefix-band-YYYY-mm-dd.YYYY-mm-dd
```

---

## End-to-end workflow

### 1 — Authenticate

```python
import grimpfunc as grimp
import os

env = dict(GDAL_HTTP_COOKIEFILE=os.path.expanduser('~/.grimp_download_cookiejar.txt'),
           GDAL_HTTP_COOKIEJAR =os.path.expanduser('~/.grimp_download_cookiejar.txt'))
os.environ.update(env)

myLogin = grimp.NASALogin()
myLogin.view()
```

### 2 — Search for products

```python
myUrls = grimp.cmrUrls()
myUrls.view()               # interactive Panel widget
# or programmatically:
myUrls.initialSearch(firstDate='2019-01-01', lastDate='2023-12-31',
                     product='NSIDC-0725')
```

### 3 — Configure the project setup

```python
import grimpqgis as grimpq

myProjectSetup = grimpq.QgisGrimpProjectSetup()

# --- Terminus positions (shapefiles, NSIDC-0642) ---
if myUrls.checkIDs(['NSIDC-0642']):
    myProjectSetup.addProductFamilies(
        'Termini',
        productFilePrefix='termini',
        bands=['termini'],
        fileType='shp',
        byYear=False,
        productPrefix='Greenland')

# --- Individual glacier TSX velocities (NSIDC-0481) ---
displayOptions = myProjectSetup.defaultDisplayOptions()
displayOptions['vv']['colorTable'] = 'Inferno'
displayOptions['vv']['maxV'] = 4000

if myUrls.checkIDs(['NSIDC-0481']):
    for boxName in myUrls.findTSXBoxes():
        myProjectSetup.addProductFamilies(
            boxName,
            productFilePrefix=f'TSX_{boxName}',
            category='TSX', productPrefix='TSX',
            bands=['vv'], byYear=True, fileType='tif',
            displayOptions=displayOptions)

# --- Velocity mosaics (NSIDC-0725/0727/0731/0766) ---
velProperties = {'category': 'Velocity', 'productPrefix': 'Vel',
                 'bands': ['browse', 'vv', 'vx', 'vy'],
                 'byYear': True, 'fileType': 'tif'}

if myUrls.checkIDs(['NSIDC-0725', 'NSIDC-0727', 'NSIDC-0731', 'NSIDC-0766']):
    for name in ['Annual', 'Quarterly', 'Monthly', 's1cycle']:
        myProjectSetup.addProductFamilies(
            name,
            productFilePrefix=f'GL_vel_mosaic_{name}',
            **velProperties)
    myProjectSetup.productFamilies['Monthly']['bands'] = ['browse', 'vv']

# --- SAR image mosaics (NSIDC-0723) ---
if myUrls.checkIDs(['NSIDC-0723']):
    myProjectSetup.addProductFamilies(
        'Image Mosaics',
        category='Imagery', productPrefix='SAR',
        productFilePrefix='GL_S1bks',
        bands=['image', 'gamma0', 'sigma0'],
        byYear=True, fileType='tif')
```

### 4 — Populate product lists from URLs

```python
# COG products (tif)
if myUrls.checkIDs(['NSIDC-0481', 'NSIDC-0723',
                    'NSIDC-0725', 'NSIDC-0727', 'NSIDC-0731', 'NSIDC-0766']):
    myProjectSetup.getProductFamilies(urls=myUrls.getCogs())

# Shapefile products
if myUrls.checkIDs(['NSIDC-0642']):
    myProjectSetup.getProductFamilies(urls=myUrls.getShapes())
```

### 5 — Build and save the QGIS project

```python
import qgis.core as qc

qc.QgsApplication.setPrefixPath('.', True)
qgs = qc.QgsApplication([], False)
qgs.initQgis()

myProject = grimpq.QgisGrimpProject(myProjectSetup)

qgisPath = 'qgisProjects'
os.makedirs(qgisPath, exist_ok=True)
QgisProjectFileName = f'{qgisPath}/qgisProject'

# Save whole project
myProject.saveProject(QgisProjectFileName)

# Optionally save layer definition files (importable into existing projects)
myProject.saveLayerDefinitions(QgisProjectFileName, saveCategories=False)  # by product family
myProject.saveLayerDefinitions(QgisProjectFileName, saveCategories=True)   # by category

qgs.exitQgis()
qgs.exit()
```

---

## Installation notes

Install into a conda environment that also has QGIS:

```
pip install git+https://github.com/fastice/grimpQGIS.git@master
```

The QGIS Python API (`qgis.core`, `qgis.gui`) must be importable from the
same environment.  If not found automatically, the module attempts to locate it
under `$CONDA_PREFIX/share/qgis/python`.  On first failure it prints
instructions for finding the correct path manually.

**Restart the Jupyter kernel** between runs — the QGIS API does not always
exit cleanly and can crash the kernel if reused in the same session.

---

## Performance notes

- QGIS verifies every layer by reading its header when the project opens.
  300 layers can take 45 seconds to 5 minutes depending on network speed.
- Keep no more than a few layers visible at once — NSIDC limits concurrent
  connections to ~15.  If an image fails to render, uncheck excess layers then
  zoom to trigger a reload.
- Use **layer definition files** (`.qlr`) to split large projects: export a
  group, remove it from the main project to speed loading, and re-import later
  via **Layers → Add From Layer Definition File**.
