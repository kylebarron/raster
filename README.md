# Raster Extension Specification

- **Title:** Raster
- **Identifier:** `https://stac-extensions.github.io/raster/v1.0.0/schema.json`
- **Field Name Prefix:** raster
- **Scope:** Item, Collection
- **Extension [Maturity Classification](https://github.com/radiantearth/stac-spec/tree/master/extensions/README.md#extension-maturity):** Proposal
- **Owner**: @emmanuelmathot

This document explains the Raster Extension to the [SpatioTemporal Asset Catalog](https://github.com/radiantearth/stac-spec) (STAC) specification.

An item can describe assets that are rasters of one or multiple bands with some information common to them all (raster size, projection)
and also specific to each of them (data type, unit, number of bits used, nodata).
A raster is ofthen strongly linked with the the georeferencing transform and coordinate system definition
of all bands (using the [projection extension](https://github.com/radiantearth/stac-spec/tree/master/extensions/projection)).
In many applications, it is interesting to have some metadata about the raster in the asset (values statistics, value interpretation, transforms).
Finally, it is helping the user to have some rendering hints of the item using one or
more raster assets (RGB combination, simple band value processing) and to create on the fly visualisation with dynamic tilers.

- Examples:
  - [Planet Item example](examples/item-planet.json): Shows the basic usage of the extension in a STAC Item
  - [Sentinel-2 Item example](examples/item-sentinel2.json): Shows the statistics about individual bands and some RGB composites example
  - [Landsat-8 Item example](examples/item-landsat8.json): Shows the advanced composite example with band math processing
- [JSON Schema](json-schema/schema.json)
- [Changelog](./CHANGELOG.md)

## Item Asset fields

| Field Name   | Type                                         | Description                                                                                                                     |
| ------------ | -------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------- |
| raster:bands | \[[Raster band Object](#raster-band-object)] | An array of available bands where each object is a \[[Band Object](#raster-band-object)]. If given, requires at least one band. |

## Item Properties

| Field Name        | Type                                                   | Description                                                                                                                                                |
| ----------------- | ------------------------------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------- |
| raster:composites | \[[Raster Composite Object](#raster-composite-object)] | An array of possible band composition where each object is a \[[Composite Object](#raster-composite-object)]. If given, requires at least one composition. |

## Raster Band Object

When specifying a raster band object at asset level, it is recommended to use
the [projection](https://github.com/radiantearth/stac-spec/tree/master/extensions/projection) extension 
to specify information about the raster projection, especially `proj:shape` to specify the height and width of the raster.

| Field Name           | Type                             | Description                                                                                                                                                                                                        |
| -------------------- | -------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| nodata               | number                           | Pixel values used to identify pixels that are nodata in the assets .                                                                                                                                               |
| sampling             | string                           | One of `area` or `point`. Indicates whether a pixel value should be assumed to represent a sampling over the region of the pixel or a point sample at the center of the pixel.                                     |
| data_type            | string                           | The data type of the band. One of the [data types as decribed in file extension](https://github.com/stac-extensions/file#data-types).                                                                              |
| nbits                | number                           | The actual number of bits used for this band. Normally only present when the number of bits is non-standard for the `datatype`, such as when a 1 bit TIFF is represented as byte                                   |
| stats_mean           | number                           | mean value of all the pixels in the band                                                                                                                                                                           |
| stats_min            | number                           | minimum value of the pixels in the band                                                                                                                                                                            |
| stats_max            | number                           | maximum value of the pixels in the band                                                                                                                                                                            |
| stats_stdev          | number                           | standard deviation value of the pixels in the band                                                                                                                                                                 |
| stats_valid_percent  | number                           | percentage of valid (not `nodata`) pixel                                                                                                                                                                           |
| values               | \[[Value Object](#value-object)] | Array of value objects                                                                                                                                                                                             |
| overview_max_gsd     | number                           | The maximum Ground Sample Distance represented in an overview. This should be the GSD of the highest level overview, generally of a [Cloud Optimized GeoTIFF](http://cogeo.org/), but should work with any format. |
| color_interpretation | string                           | the color interpretation of the pixels in the bands. One of the [color interpreation](#color-interpretation)) below.                                                                                               |

**overview_max_gsd**: This field helps renderers of understand what zoom levels they can efficiently show.
It is generally used in conjunction with gsd
(from [common metadata](https://github.com/radiantearth/stac-spec/blob/master/item-spec/common-metadata.md#instrument)).
`overview_max_gsd` enables the calculation of the 'minimum' zoom level that a renderer would want to show,
and then the maximum zoom level is calculated from the gsd - the resolution of the image.
The former is based on the highest level of overview (also known as a pyramid) contained in the asset.

![overviews](images/overviews.png)

So in the above image it would be the ground sample distance of 'level 4', which will be a much higher gsd than the image,
as each pixel is greatly down-sampled.
Dynamic tile servers (like [titiler](https://github.com/developmentseed/titiler)) will
generally convert the gsd to [zoom levels](https://wiki.openstreetmap.org/wiki/Zoom_levels),
[Web Mercator](https://en.wikipedia.org/wiki/Web_Mercator_projection) or others,
which is easily done
(example python [to webmercator](https://github.com/cogeotiff/rio-cogeo/blob/b9b57301c2b7a4be560c887176c282e68ca63c27/rio_cogeo/utils.py#L62-L66)
or arbitrary
[TileMatrixSet](https://github.com/cogeotiff/rio-tiler-crs/blob/834bcf3d39cdc555b3ce930439ab186d00fd5fc5/rio_tiler_crs/cogeo.py#L98-L105))

## Value Object

Value object describes the raw pixel value in the band or any other derivable values from that band.

| Field Name | Type   | Description                                                                                                                                                               |
| ---------- | ------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| name       | string | name of the value                                                                                                                                                         |
| unit       | string | unit denomination of the value                                                                                                                                            |
| from       | string | key of another value in the [`values`](#raster-band-object) dictionary to be used as input to compute the value. If empty the value of the pixel in the band is the input |
| scale      | number | multiplicator factor of the pixel value to transform into the value (i.e. translate digital number to reflectance).                                                       |
| offset     | number | number to be added to the pixel value to transform into the value (i.e. translate digital number to reflectance).                                                         |

`scale` and `offset` defines parameters to compute another value. Next paragraphs describe some use cases.

`from` allows chaining values with each others. The following basic example show its usage

```json
"values": [
  {
    "name": "A",
    "unit": "m",
  },
  {
    "name": "B",
    "unit": "cm",
    "scale": 0.01,
    "offset": 0
  },
  {
    "name": "C",
    "unit": "mm",
    "scale": 0.1,
    "offset": 0,
    "from": "B"
  }
]
```

### Use Scale and offset as radiometric calibration parameters

In remote sensing, many imagery raster corresponds to raw data without any radiometric processing.
Each pixel is given in digital numbers (DN), i.e. native pixel values from the sensor acquisition.
Those digital numbers quantify the energy recorded by the detector (optical or radar).
The sensor radiometric calibration aims to turn back the DN value into a
physical unit value (radiance, light power, backscatter).
Hereafter, some examples of the usage of the `values` dictionary to perform radiometric correction.

#### Digital Numbers to Radiance (optical sensor)

<!-- https://labo.obs-mip.fr/multitemp/radiometric-quantities-irradiance-radiance-reflectance/ -->

A conventional way of deriving Top Of Atmosphere (TOA) Radiance
in ![formula](https://render.githubusercontent.com/render/math?math=W.sr^{-1}.m^{-3})
from DN values using `scale` and `offset` in the following formula:

![formula](https://render.githubusercontent.com/render/math?math=L_\lambda%20=%20scale%20\times%20DN%20%2B%20offset)

where ![formula](https://render.githubusercontent.com/render/math?math=L_\lambda) is TOA Radiance
in ![formula](https://render.githubusercontent.com/render/math?math=W.sr^{-1}.m^{-3}).

For example, the above value conversion is described in the values dictionary as

```json
"values": {
  "TOA radiance": {
      "unit": "W⋅sr−1⋅m−3",
      "scale": 0.0145,
      "offset": 3.48
  }
}
```

### Transform height measurement to water level

In remote sensing, radar altimeter instruments measures an absolute height from an absolute georeference (e.g. WGS 84 geoid).
In hydrology, you prefer having the water level relative to the "0 limnimetric scale".
Therefore, a usage of the value object here would be to indicate the offset between the reference height 0 of the sensor
and the 0 limnimetric scale to compute a water level.

In the following value definition example, 185 meters must be substracted from the pixel value to correspond to the water level.

```json
"values": {
  "Water Level": {
      "unit": "m",
      "offset": -185
  }
}
```

## Raster Composite Object

Raster composites are intended to be used to specify some possible sensor band combination with generic parameters.
It can be useful to propose visualization hints like spectral indices from electro-optical sensor (e.g. NDVI)
or specific overviews (e.g. Thermal signatures).

| Field Name        | Type                        | Description                                                                                                                                                                                                                |
| ----------------- | --------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| name              | string                      | **REQUIRED**. Denomination of the band composition (e.g. `ndvi`, `Color Infrared (vegetation)`  )                                                                                                                          |
| nodata            | number                      | Pixel values used to identify pixels that are nodata in the composition .                                                                                                                                                  |
| range             | \[number]                   | range of valid pixels values in the composition                                                                                                                                                                            |
| bands             | \[[string](#band-selector)] | An ordered array of string following [Band Selector](#band-selector) syntax. If given, requires at least one band. **REQUIRED** if no `band_expression` specified                                                          |
| band_expression   | string                      | Band math expression (e.g `(b4-b1)/(b4+b1)`). **REQUIRED** if no `bands` specified                                                                                                                                         |
| resampling_method | string                      | Resampling method, one of `nearest`, `average`, `bilinear` or `cubic`.                                                                                                                                                     |
| color_map         | string                      | Identifier of a color mapping. Currently based on [rio-tiler color maps](https://cogeotiff.github.io/rio-tiler/colormap/) that includes some from Matplotlib and some custom ones that are commonly used with raster data. |

## Band Selector

The band selector is a string indicating the raster asset by its key and optional band index if multi-band raster asset.

`<asset_key>[{<band_index>}]`

with

- `asset_key` (**REQUIRED**): Asset key to the raster asset in the item 
- `band_index` (OPTIONAL): Band position index in the raster asset

examples: `B4`, `data{2}`

### Color Interpretation

TBD

## Dynamic tile servers integration

Dynamic tile servers could exploit the information in the raster extension to automatically produce RGB
from raster bands or composition using their parameters.

### Titiler

[titiler](https://github.com/developmentseed/titiler) offers a native
[STAC reader](https://github.com/developmentseed/titiler/blob/master/docs/endpoints/stac.md).
Some query parameters could be set with the information from raster extension.

#### Shortwave Infra-red visual thermal signature example

From the [Sentinel-2 example](examples/item-sentinel2.json):

```json
"raster:composites":[ 
  {
    "name": "Shortwave Infra-red",
    "range": [0, 10000],
    "bands": [ "B12", "B8A", "B04"]
  }
]
```

| Query key | value                                                             | Example value                                                                                |
| --------- | ----------------------------------------------------------------- | -------------------------------------------------------------------------------------------- |
| url       | STAC Item URL                                                     | `https://raw.githubusercontent.com/stac-extensions/raster/main/examples/item-sentinel2.json` |
| assets    | Assets keys defined in the `bands` objects with field `asset_key` | `B12,B8A,B04`                                                                                |  |
| rescale   | Delimited Min,Max bounds defined in field `range`                 | `0,10000`                                                                                    |

URL: `https://api.cogeo.xyz/stac/crop/14.869,37.682,15.113,37.862/256x256.png?url=https://raw.githubusercontent.com/stac-extensions/raster/main/examples/item-sentinel2.json&assets=B12,B8A,B04&resampling_method=average&rescale=0,10000&return_mask=true`

**Result**: Lava thermal signature of Mount Etna eruption (February 2021)

![etna](images/etna.png)

#### Normalized Difference Vegetation Index (NDVI) example

From the [Landsat-8 example](examples/item-landsat8.json) \[[article](https://www.usgs.gov/core-science-systems/nli/landsat/landsat-normalized-difference-vegetation-index?qt-science_support_page_related_con=0#qt-science_support_page_related_con)]:

```json
"raster:composites":[ 
  {
    "name": "Normalized Difference Vegetation Index",
    "range": [-1, 1],
    "expression": "(B5–B4)/(B5+B4)",
    "color_map": "ylgn"
  }
]
```

| Query key  | value                                                     | Example value                                                                               |
| ---------- | --------------------------------------------------------- | ------------------------------------------------------------------------------------------- |
| url        | STAC Item URL                                             | `https://raw.githubusercontent.com/stac-extensions/raster/main/examples/item-landsat8.json` |  |
| rescale    | Delimited Min,Max bounds defined in field `range`         | `-1,1`                                                                                      |
| expression | Band math formula as defined in field `band_math_formula` | `(B5–B4)/(B5+B4)`                                                                           |
| color_map  | Color map defined in field `color_map`                    | `ylgn`                                                                                      |

URL:

`https://api.cogeo.xyz/stac/preview.png?url=https://raw.githubusercontent.com/stac-extensions/raster/main/examples/item-landsat8.json&expression=(B5–B4)/(B5+B4)&max_size=512&width=512&resampling_method=average&rescale=-1,1&color_map=ylgn&return_mask=true`

Result:  Landsat Surface Reflectance Normalized Difference Vegetation Index (NDVI) path 44 row 33.

![sacramento](https://api.cogeo.xyz/stac/preview.png?url=https://raw.githubusercontent.com/stac-extensions/raster/main/examples/item-landsat8.json&expression=(B5–B4)/(B5+B4)&max_size=512&width=512&resampling_method=average&rescale=-1,1&color_map=ylgn&return_mask=true)

## Contributing

All contributions are subject to the
[STAC Specification Code of Conduct](https://github.com/radiantearth/stac-spec/blob/master/CODE_OF_CONDUCT.md).
For contributions, please follow the
[STAC specification contributing guide](https://github.com/radiantearth/stac-spec/blob/master/CONTRIBUTING.md) Instructions
for running tests are copied here for convenience.

### Running tests

The same checks that run as checks on PR's are part of the repository and can be run locally to verify that changes are valid. 
To run tests locally, you'll need `npm`, which is a standard part of any [node.js installation](https://nodejs.org/en/download/).

First you'll need to install everything with npm once. Just navigate to the root of this repository and on 
your command line run:
```bash
npm install
```

Then to check markdown formatting and test the examples against the JSON schema, you can run:
```bash
npm test
```

This will spit out the same texts that you see online, and you can then go and fix your markdown or examples.

If the tests reveal formatting problems with the examples, you can fix them with:
```bash
npm run format-examples
```
