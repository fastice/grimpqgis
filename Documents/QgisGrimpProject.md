# QgisGrimpProject — QGIS Project Builder

Takes a configured `QgisGrimpProjectSetup` object and uses the QGIS Python
API to build a complete layer tree, then saves the project as a `.qgs` file
and optionally as a set of layer definition (`.qlr`) files.

> **QGIS must be initialised before constructing this object** — see the
> startup sequence below.  Restart the Jupyter kernel between runs; the QGIS
> API does not always clean up properly and can crash the kernel if reused.

---

## QGIS startup sequence

```python
import qgis.core as qc

# Point QGIS at the local share/ directory for colour ramps and style files
qc.QgsApplication.setPrefixPath('.', True)
qgs = qc.QgsApplication([], False)
qgs.initQgis()
```

The `share/` directory distributed with the GrIMPNotebooks repository must be
present in the working directory (or the prefix path updated) so that QGIS can
find its style resources.

---

## Construction

```python
myProject = grimpq.QgisGrimpProject(myProjectSetup)
```

**Parameters:**
- `grimpSetup` — a populated `QgisGrimpProjectSetup` instance.
- `crs` — EPSG code for the project CRS.  Default: `3413` (Greenland polar
  stereographic).
- `relative` — if `True`, paths in the `.qgs` file are relative rather than
  absolute.  Default: `False` (absolute paths recommended for remote COG files).
- `**kwargs` — optional extent override keywords: `xmin`, `xmax`, `ymin`,
  `ymax` (metres in the project CRS).  The default extent covers all of
  Greenland (`xmin=-626000`, `ymin=-3356000`, `xmax=850000`, `ymax=-695000`).

The constructor immediately calls `_buildProductTree()`, which iterates through
all product families and bands in `grimpSetup`, creates QGIS raster or vector
layers, and adds them to the layer tree.  **This step can take 30 minutes or
more for several hundred products** because QGIS must read each layer header
to register it.

---

## Saving

### `saveProject`

```python
myProject.saveProject('qgisProjects/myProject')
# Writes: qgisProjects/myProject.qgs
```

Write the QGIS project file.  If the file already exists it is deleted first
(pass `append=True` to skip deletion).

### `saveLayerDefinitions`

```python
# One .qlr file per product family (e.g., Annual, Quarterly, Monthly)
myProject.saveLayerDefinitions('qgisProjects/myProject', saveCategories=False)

# One .qlr file per category (e.g., Velocity, Imagery)
myProject.saveLayerDefinitions('qgisProjects/myProject', saveCategories=True)
```

Save layer definition files that can be imported into existing QGIS projects
via **Layers → Add From Layer Definition File**.  Useful for:
- Splitting a large project into importable pieces to avoid slow startup.
- Re-using layer groups across projects.

Files are named `<prefix>.<familyOrCategoryName>`.

---

## QGIS shutdown

```python
qgs.exitQgis()
qgs.exit()
```

Always call these at the end of the session.  In Jupyter, restart the kernel
afterwards before running the notebook again.

---

## Layer tree structure built by the constructor

For each product family in `grimpSetup.productFamilies`:

1. An optional **Category** group is created or reused at the tree root
   (e.g. `'Velocity'`).
2. A **Product Family** group is created or reused under the Category
   (e.g. `'Annual'`).
3. If `byYear=True`, a **Year** sub-group is created for each product's mid-
   date year.
4. For multi-band families a **date-range** group is created per product
   (e.g. `Vel-2020-01-01.2020-01-06`), with each band as a child layer.
   For single-band families the band name is folded into the layer name and
   no extra group level is created.

All layers are initially **unchecked** and **collapsed** to avoid triggering
simultaneous network requests when the project opens.  Only the first group
at each level is set visible.

---

## Display options applied at construction

Raster layers receive a single-band pseudocolour renderer built from the
`displayOptions` dict stored in their product family:

| Key | Effect |
|-----|--------|
| `colorTable` | QGIS named colour ramp (e.g. `'Blues'`, `'RdBu'`, `'Greys'`).  `None` leaves the default renderer (used for RGB browse images). |
| `minV` / `maxV` | Classification range for the colour ramp |
| `invert` | Invert the colour ramp direction |
| `opacity` | Layer opacity (0–1) |

Vector layers (shapefiles) receive basic styling: line width 0.6 for line
geometries, `outline red` for polygon fills.

---

## Internal methods (reference)

| Method | Description |
|--------|-------------|
| `_buildProductTree()` | Main loop: iterates families and bands, calls `_addProductBand` |
| `_addProductBand(band, productFamily, products, group, bar)` | Resolves dates and group, then adds each product as a layer |
| `_getProductGroup(...)` | Returns the correct QGIS group for a product (handles byYear, single/multi-band) |
| `_getYearGroup(group, year)` | Find or create a year sub-group |
| `_getBandGroup(name, group, family)` | Find or create a band/date sub-group |
| `_getTopGroup(productFamily)` | Find or create the category group at the tree root |
| `_addproductFamilyToTree(productFamily)` | Find or create the product family group |
| `_addLayerToGroup(group, product, name, band, displayOptions)` | Dispatch to raster or vector layer creation and add to group |
| `_addRasterLayer(product, name, band, displayOptions)` | Create `QgsRasterLayer` and apply colour ramp |
| `_addVectorLayer(product, name)` | Create `QgsVectorLayer` with basic line/fill styling |
| `_setLayerColorTable(layer, colorTable, minV, maxV, invert, opacity)` | Apply single-band pseudocolour renderer to a raster layer |
| `_getDates(product)` | Parse start/end dates from a product filename or path |
| `_updateMaxExtent(layer)` | Track the largest layer extent seen so far |
| `_checkFirstLayerOfLastGroup(group)` | Recursively set only the first non-vector layer visible |
| `_productCount()` | Count total number of products across all families |
| `_getNumberOfBandsWithData(family)` | Count bands that have at least one product |

---

## Date parsing

`_getDates` extracts product dates from filenames using hard-coded position
tables:

| Filename prefix | Date field positions | Example |
|-----------------|---------------------|---------|
| `GL_vel` | pieces[4], pieces[5] | `GL_vel_mosaic_Annual_01Dec14_30Nov15_vv_v05.0.tif` |
| `GL_S1bks` | pieces[3], pieces[4] | `GL_S1bks_image_01Jan20_31Jan20.tif` |
| `TSX` | pieces[2], pieces[3] | `TSX_W69.10N_01Jan20_15Jan20_vv.tif` |
| terminus shapefiles | directory name (YYYY.MM.DD) | `.../2020.01.15/termini_...shp` |

Internal GIMP products with `/shp/` in the path use a separate parsing path
based on the directory name.
