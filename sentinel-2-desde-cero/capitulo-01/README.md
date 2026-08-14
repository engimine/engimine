# Capítulo 1 - Rasters y datos Sentinel-2 reales

En este capítulo dejamos la teoría inicial y empezamos a trabajar con datos reales. El objetivo es entender qué ocurre debajo de una plataforma GIS: crear un raster a mano, buscar una escena Sentinel-2 mediante STAC, leer únicamente la zona que necesitamos, trabajar con reflectancia y construir máscaras de calidad.

El flujo será:

```text
matriz NumPy → raster georreferenciado → STAC → Sentinel-2 L2A
→ COG remoto → ventana espacial → reflectancia → máscara AOI/SCL → GeoTIFF
```

## 1.1. Un raster no es simplemente una imagen

Un raster geoespacial combina una matriz de valores con información que indica dónde se encuentra cada celda en el mundo. Necesitamos, como mínimo, ancho y alto, tamaño de píxel, transformación espacial, CRS y valor nodata.

```python
import numpy as np

raster = np.array([
    [1, 1, 2, 2],
    [1, 3, 3, 2],
    [4, 4, 3, 2],
    [4, 4, 4, 2],
], dtype="uint8")

print(raster.shape)
```

Cada número puede representar una clase o una medición. Todavía no sabe dónde está en la Tierra.

## 1.2. Georreferenciarlo

```python
from rasterio.transform import from_origin

transform = from_origin(
    500000,   # coordenada X superior izquierda
    4500000,  # coordenada Y superior izquierda
    10,       # tamaño píxel X
    10,       # tamaño píxel Y
)
```

Si utilizamos un CRS proyectado en metros, cada píxel de 10 x 10 m representa 100 m².

## 1.3. Guardar nuestro primer GeoTIFF

```python
import rasterio

with rasterio.open(
    "raster_manual.tif",
    "w",
    driver="GTiff",
    height=raster.shape[0],
    width=raster.shape[1],
    count=1,
    dtype=raster.dtype,
    crs="EPSG:32631",
    transform=transform,
) as dst:
    dst.write(raster, 1)
```

Ya tenemos un archivo GIS real creado desde Python.

---

# Parte A - Buscar Sentinel-2 sin Earth Engine

## 1.4. STAC

STAC permite describir y buscar activos geoespaciales mediante un estándar abierto. Utilizaremos Earth Search v1 como catálogo público.

```python
import pystac_client

catalog = pystac_client.Client.open(
    "https://earth-search.aws.element84.com/v1"
)
```

## 1.5. Definir la zona de estudio

Usaremos Manhattan como caso didáctico:

```python
bbox = [-74.03, 40.69, -73.93, 40.83]
```

El orden es:

```text
[min_lon, min_lat, max_lon, max_lat]
```

## 1.6. Buscar escenas

```python
search = catalog.search(
    collections=["sentinel-2-l2a"],
    bbox=bbox,
    datetime="2025-06-01/2025-08-31",
    query={"eo:cloud_cover": {"lt": 20}},
)

items = list(search.items())
print("Escenas:", len(items))
```

Ordenamos por nubosidad:

```python
items = sorted(
    items,
    key=lambda x: x.properties.get("eo:cloud_cover", 100),
)

item = items[0]
print(item.id)
print(item.datetime)
print(item.properties.get("eo:cloud_cover"))
```

Una baja nubosidad global no garantiza que nuestra AOI esté libre de nubes; más adelante utilizaremos SCL.

---

# Parte B - Inspeccionar los activos

## 1.7. Qué archivos contiene una escena

```python
for key in item.assets:
    print(key)
```

Los nombres exactos de assets dependen de la colección. Debemos inspeccionarlos, no asumirlos a ciegas.

Para nuestro flujo necesitaremos equivalentes de:

```text
B02 / blue
B03 / green
B04 / red
B08 / nir
SCL
```

Y más adelante B11 para NDBI/NDMI.

## 1.8. Consultar metadatos del asset

```python
asset = item.assets["red"]
print(asset.href)
print(asset.extra_fields)
```

STAC puede proporcionar información raster como escala y offset. Conviene leerla en vez de codificar constantes sin comprobar el producto.

---

# Parte C - COG: leer solo lo que necesitamos

## 1.9. Cloud Optimized GeoTIFF

Un COG está organizado para permitir lecturas parciales eficientes a través de HTTP. No necesitamos descargar necesariamente un tile Sentinel-2 completo para analizar un barrio.

Con Rasterio podemos abrir el recurso remoto:

```python
import rasterio

href = item.assets["red"].href

with rasterio.open(href) as src:
    print(src.crs)
    print(src.res)
    print(src.width, src.height)
```

## 1.10. Transformar el bbox al CRS del raster

Nuestro bbox está en EPSG:4326. La banda Sentinel-2 suele estar en una proyección UTM.

```python
from rasterio.warp import transform_bounds

with rasterio.open(href) as src:
    bounds_projected = transform_bounds(
        "EPSG:4326",
        src.crs,
        *bbox,
        densify_pts=21,
    )
```

## 1.11. Crear una ventana

```python
from rasterio.windows import from_bounds

with rasterio.open(href) as src:
    window = from_bounds(
        *bounds_projected,
        transform=src.transform,
    ).round_offsets().round_lengths()

    red_dn = src.read(1, window=window)
    window_transform = src.window_transform(window)
```

Esta idea es fundamental: **leer por ventana** reduce tráfico, memoria y tiempo.

---

# Parte D - Función reutilizable para leer una banda

```python
from rasterio.warp import transform_bounds
from rasterio.windows import from_bounds


def read_band_window(href, bbox_wgs84):
    with rasterio.open(href) as src:
        projected = transform_bounds(
            "EPSG:4326",
            src.crs,
            *bbox_wgs84,
            densify_pts=21,
        )

        window = from_bounds(
            *projected,
            transform=src.transform,
        ).round_offsets().round_lengths()

        data = src.read(1, window=window)

        profile = src.profile.copy()
        profile.update(
            height=data.shape[0],
            width=data.shape[1],
            transform=src.window_transform(window),
        )

        return data, profile
```

Uso:

```python
red_raw, red_profile = read_band_window(
    item.assets["red"].href,
    bbox,
)
```

---

# Parte E - DN, escala, offset y reflectancia

## 1.12. Qué estamos leyendo

Los valores almacenados en el raster no deben interpretarse automáticamente como reflectancia decimal. El producto utiliza cuantificación y metadatos de transformación.

La forma general es:

```text
valor físico = DN × scale + offset
```

Cuando el asset STAC expone `raster:bands`, podemos inspeccionarlo:

```python
meta = item.assets["red"].extra_fields.get("raster:bands", [{}])[0]

scale = meta.get("scale", 1.0)
offset = meta.get("offset", 0.0)

print(scale, offset)
```

Aplicación:

```python
red = red_raw.astype("float32") * scale + offset
```

Función:

```python
def to_physical_values(raw, asset):
    band_meta = asset.extra_fields.get("raster:bands", [{}])[0]
    scale = band_meta.get("scale", 1.0)
    offset = band_meta.get("offset", 0.0)

    out = raw.astype("float32") * scale + offset
    return out
```

No conviene aplicar `/ 10000` a ciegas sin saber qué colección y metadatos estamos leyendo.

---

# Parte F - Leer RGB y NIR

```python
band_keys = {
    "blue": "blue",
    "green": "green",
    "red": "red",
    "nir": "nir",
}

bands = {}
profiles = {}

for name, key in band_keys.items():
    raw, profile = read_band_window(item.assets[key].href, bbox)
    bands[name] = to_physical_values(raw, item.assets[key])
    profiles[name] = profile
```

Comprobamos:

```python
for name, arr in bands.items():
    print(name, arr.shape, np.nanmin(arr), np.nanmax(arr))
```

---

# Parte G - Crear una imagen RGB

## 1.13. El problema del contraste

La reflectancia física no siempre se ve bien si la mostramos directamente. Para visualización podemos aplicar un estiramiento percentil sin modificar los datos científicos originales.

```python
def stretch(arr, p_low=2, p_high=98):
    low, high = np.nanpercentile(arr, [p_low, p_high])
    out = (arr - low) / (high - low)
    return np.clip(out, 0, 1)
```

```python
rgb = np.dstack([
    stretch(bands["red"]),
    stretch(bands["green"]),
    stretch(bands["blue"]),
])
```

```python
import matplotlib.pyplot as plt

plt.figure(figsize=(9, 9))
plt.imshow(rgb)
plt.title("Sentinel-2 RGB")
plt.axis("off")
plt.show()
```

---

# Parte H - Máscara por polígono

## 1.14. Un bbox no siempre es nuestra AOI real

Podemos tener un municipio, una parcela o un polígono minero. GeoPandas permite cargarlo:

```python
import geopandas as gpd

aoi = gpd.read_file("aoi.geojson")
aoi = aoi.to_crs(red_profile["crs"])
```

Rasterizamos el polígono sobre la cuadrícula de la banda:

```python
from rasterio.features import geometry_mask

mask_inside = geometry_mask(
    aoi.geometry,
    out_shape=bands["red"].shape,
    transform=red_profile["transform"],
    invert=True,
)
```

Aplicación:

```python
red_aoi = np.where(mask_inside, bands["red"], np.nan)
```

La misma máscara puede aplicarse a las bandas que compartan exactamente la misma cuadrícula.

---

# Parte I - SCL: Scene Classification Layer

## 1.15. Por qué necesitamos una máscara de calidad

Sentinel-2 L2A incorpora SCL, una clasificación de escena que ayuda a identificar píxeles como vegetación, agua, nubes, sombras, nieve y otras categorías.

La SCL es categórica y suele estar a 20 m, mientras que B02/B03/B04/B08 están a 10 m.

No podemos combinarla directamente si las cuadrículas no coinciden.

## 1.16. Leer SCL

```python
scl_raw, scl_profile = read_band_window(
    item.assets["scl"].href,
    bbox,
)
```

No aplicamos interpolación continua a clases.

## 1.17. Reproyectar SCL a 10 m

```python
from rasterio.warp import reproject, Resampling

scl_10m = np.empty(
    bands["red"].shape,
    dtype=scl_raw.dtype,
)

reproject(
    source=scl_raw,
    destination=scl_10m,
    src_transform=scl_profile["transform"],
    src_crs=scl_profile["crs"],
    dst_transform=red_profile["transform"],
    dst_crs=red_profile["crs"],
    resampling=Resampling.nearest,
)
```

**Nearest neighbour** es obligatorio conceptualmente para categorías: una nube clase 9 y una clase 4 no deben generar una clase decimal inventada.

## 1.18. Crear una máscara válida

Una selección didáctica de clases problemáticas:

```python
bad_scl = [0, 1, 3, 8, 9, 10, 11]
valid_scl = ~np.isin(scl_10m, bad_scl)
```

Podemos combinarla con el AOI:

```python
valid = valid_scl & mask_inside

red_clean = np.where(valid, bands["red"], np.nan)
nir_clean = np.where(valid, bands["nir"], np.nan)
```

Las clases que se excluyen deben documentarse y adaptarse al objetivo del estudio.

---

# Parte J - Guardar el resultado

## 1.19. GeoTIFF de reflectancia recortada

```python
profile = red_profile.copy()
profile.update(
    dtype="float32",
    count=1,
    nodata=np.nan,
    compress="deflate",
)

with rasterio.open("red_clean.tif", "w", **profile) as dst:
    dst.write(red_clean.astype("float32"), 1)
```

Podemos guardar NIR del mismo modo y utilizar ambos en el capítulo 2.

---

# Parte K - Script conceptual completo

```python
# 1. Abrir catálogo STAC
catalog = pystac_client.Client.open(
    "https://earth-search.aws.element84.com/v1"
)

# 2. Buscar Sentinel-2 L2A
items = list(catalog.search(
    collections=["sentinel-2-l2a"],
    bbox=bbox,
    datetime="2025-06-01/2025-08-31",
    query={"eo:cloud_cover": {"lt": 20}},
).items())

# 3. Elegir candidato
item = min(
    items,
    key=lambda x: x.properties.get("eo:cloud_cover", 100),
)

# 4. Leer solo las ventanas necesarias
red_raw, red_profile = read_band_window(item.assets["red"].href, bbox)
nir_raw, nir_profile = read_band_window(item.assets["nir"].href, bbox)
scl_raw, scl_profile = read_band_window(item.assets["scl"].href, bbox)

# 5. Pasar a valores físicos según metadatos
red = to_physical_values(red_raw, item.assets["red"])
nir = to_physical_values(nir_raw, item.assets["nir"])

# 6. Reproyectar SCL con nearest
scl_10m = align_scl_to_reference(scl_raw, scl_profile, red_profile)

# 7. Máscara de calidad
bad_scl = [0, 1, 3, 8, 9, 10, 11]
valid = ~np.isin(scl_10m, bad_scl)

# 8. Aplicar AOI si existe
valid &= polygon_mask

# 9. Limpiar
red = np.where(valid, red, np.nan)
nir = np.where(valid, nir, np.nan)

# 10. Guardar
save_geotiff("red_clean.tif", red, red_profile)
save_geotiff("nir_clean.tif", nir, nir_profile)
```

Las funciones `align_scl_to_reference` y `save_geotiff` encapsulan exactamente las operaciones explicadas en las secciones anteriores.

---

# Parte L - Errores frecuentes

## Error 1: descargar escenas completas innecesariamente

Si el origen es COG, leer ventanas suele ser mucho más eficiente.

## Error 2: asumir que bbox y raster comparten CRS

El bbox WGS84 debe transformarse al CRS del raster antes de construir la ventana.

## Error 3: dividir siempre por 10.000

Debemos conocer la codificación del producto y, cuando estén disponibles, utilizar `scale` y `offset` de metadatos.

## Error 4: mezclar SCL de 20 m con bandas de 10 m

Hay que alinear cuadrículas.

## Error 5: usar bilinear para SCL

Las clases categóricas se remuestrean con vecino más próximo.

## Error 6: confundir baja nubosidad de escena con AOI limpia

El porcentaje `eo:cloud_cover` describe la escena; la máscara SCL controla nuestros píxeles.

## Error 7: estirar valores para visualizar y luego analizarlos

El `stretch` es solo para la imagen RGB. Los cálculos científicos deben utilizar los valores físicos originales.

---

# Parte M - Ejercicios

1. Crea un raster manual de 10 x 10 píxeles y guárdalo como GeoTIFF.
2. Cambia el bbox de Manhattan por tu ciudad y busca escenas Sentinel-2 L2A.
3. Imprime todos los assets de una escena y localiza rojo, verde, azul, NIR y SCL.
4. Lee una ventana pequeña de B04 y compara su tamaño con el tile completo.
5. Construye una composición RGB.
6. Carga un GeoJSON propio y enmascara las bandas por polígono.
7. Reproyecta SCL a la cuadrícula de 10 m con nearest.
8. Calcula qué porcentaje de píxeles de tu AOI queda válido después de la máscara.
9. Guarda B04 y B08 limpios como GeoTIFF.
10. Reto: crea una función `get_sentinel_patch(bbox, dates)` que automatice búsqueda, lectura y control de calidad.

---

# Qué hemos aprendido

Al terminar el capítulo entendemos el recorrido real de los datos:

```text
CATÁLOGO STAC
     ↓
ITEM SENTINEL-2
     ↓
ASSETS COG
     ↓
VENTANA ESPACIAL
     ↓
DN / METADATOS
     ↓
REFLECTANCIA
     ↓
AOI + SCL
     ↓
BANDAS LIMPIAS GEOREFERENCIADAS
```

Ya no dependemos de una interfaz que esconda estos pasos. Tenemos las piezas necesarias para el capítulo 2, donde combinaremos bandas y construiremos NDVI, NDWI y NDBI.

## Referencias técnicas

- Earth Search STAC v1: https://earth-search.aws.element84.com/v1
- Element 84 Earth Search: https://element84.com/earth-search/
- Copernicus Sentinel-2: https://dataspace.copernicus.eu/explore-data/data-collections/sentinel-data/sentinel-2
- Rasterio: https://rasterio.readthedocs.io/
- STAC: https://stacspec.org/

---

[← Volver al índice del manual](../README.md)
