# Capítulo 2 - Índices espectrales: NDVI, NDWI y NDBI

En este capítulo damos el salto de **ver bandas** a **extraer información física del territorio**.

Hasta ahora cada banda de Sentinel-2 era una imagen independiente: rojo, verde, infrarrojo cercano, SWIR... A partir de aquí vamos a combinarlas para construir índices que nos permitan responder preguntas como:

- ¿Dónde hay vegetación?
- ¿Qué zonas tienen vegetación más vigorosa?
- ¿Dónde aparece agua?
- ¿Qué superficies se comportan como suelo construido o impermeable?
- ¿Cómo convertir cada píxel en un vector de características preparado para Machine Learning?

El objetivo no es memorizar fórmulas. El objetivo es entender **por qué funcionan**.

---

## 2.1. Qué es un índice espectral

Un índice espectral es una operación matemática entre dos o más bandas de una imagen satelital.

La idea fundamental es sencilla: diferentes materiales reflejan la radiación de forma distinta según la longitud de onda.

Una hoja sana, el agua y una cubierta de hormigón no tienen la misma firma espectral.

Sentinel-2 mide esas diferencias mediante sus bandas.

Cuando combinamos bandas adecuadas podemos aumentar el contraste entre tipos de superficie.

Una forma muy habitual es el índice normalizado:

```text
(A - B) / (A + B)
```

Esta operación suele producir valores aproximadamente comprendidos entre -1 y 1.

La normalización reduce parte del efecto de la iluminación y facilita comparar píxeles y fechas.

> Importante: un índice no es una clasificación automática. Un NDVI alto suele asociarse a vegetación, pero los umbrales dependen del lugar, la fecha, el sensor, la atmósfera y el objetivo del estudio.

---

## 2.2. Las bandas que vamos a utilizar

Para este capítulo nos interesan principalmente estas bandas Sentinel-2 L2A:

| Banda | Nombre | Resolución | Uso principal |
|---|---|---:|---|
| B02 | Azul | 10 m | agua, atmósfera, color natural |
| B03 | Verde | 10 m | agua y vegetación |
| B04 | Rojo | 10 m | vegetación y RGB |
| B08 | NIR | 10 m | vegetación, biomasa, agua |
| B11 | SWIR 1 | 20 m | humedad, suelo, construido |
| B12 | SWIR 2 | 20 m | humedad, incendios, suelo |

Para los índices de este capítulo:

```text
NDVI = (B08 - B04) / (B08 + B04)
NDWI = (B03 - B08) / (B03 + B08)
NDBI = (B11 - B08) / (B11 + B08)
```

Hay un detalle muy importante: **B11 está a 20 m y B08 a 10 m**. No debemos operar ambas matrices hasta ponerlas en la misma cuadrícula.

---

# Parte A - Una única función para todos los índices

## 2.3. Evitar divisiones problemáticas

Podríamos escribir directamente:

```python
ndvi = (nir - red) / (nir + red)
```

pero algunos píxeles pueden tener un denominador cero o valores no válidos.

Una función más robusta es:

```python
import numpy as np


def normalized_difference(a, b):
    """Calcula (a-b)/(a+b) evitando divisiones inválidas."""
    a = a.astype("float32")
    b = b.astype("float32")

    denominator = a + b

    result = np.full(a.shape, np.nan, dtype="float32")

    valid = (
        np.isfinite(a)
        & np.isfinite(b)
        & (denominator != 0)
    )

    result[valid] = (a[valid] - b[valid]) / denominator[valid]

    return result
```

Y a partir de ahí:

```python
ndvi = normalized_difference(nir, red)
ndwi = normalized_difference(green, nir)
ndbi = normalized_difference(swir1, nir)
```

Una sola receta. Tres índices.

---

# Parte B - NDVI: vegetación

## 2.4. Qué mide realmente el NDVI

NDVI significa **Normalized Difference Vegetation Index**.

```text
NDVI = (NIR - RED) / (NIR + RED)
```

En Sentinel-2:

```text
NDVI = (B08 - B04) / (B08 + B04)
```

La vegetación sana absorbe gran parte de la luz roja para la fotosíntesis y refleja fuertemente el infrarrojo cercano.

Por eso una superficie vegetal suele cumplir:

```text
NIR alto
RED bajo
```

El cociente normalizado se vuelve positivo.

### Interpretación orientativa

| NDVI | Interpretación posible |
|---:|---|
| < 0 | agua, sombra, algunas superficies artificiales |
| 0 - 0.2 | suelo desnudo o superficie urbana |
| 0.2 - 0.4 | vegetación escasa |
| 0.4 - 0.6 | vegetación moderada |
| > 0.6 | vegetación densa o vigorosa |

Estos rangos son orientativos, no reglas universales.

---

## 2.5. Calcular NDVI en Python

Suponiendo que `red` y `nir` ya contienen reflectancia superficial:

```python
ndvi = normalized_difference(nir, red)

print("NDVI mínimo:", np.nanmin(ndvi))
print("NDVI máximo:", np.nanmax(ndvi))
print("NDVI medio:", np.nanmean(ndvi))
```

Visualización:

```python
import matplotlib.pyplot as plt

plt.figure(figsize=(10, 8))
plt.imshow(ndvi, vmin=-1, vmax=1, cmap="RdYlGn")
plt.colorbar(label="NDVI")
plt.title("NDVI - Sentinel-2")
plt.axis("off")
plt.show()
```

---

## 2.6. Una máscara sencilla de vegetación

Por ejemplo:

```python
vegetation = ndvi > 0.4
```

Visualización:

```python
plt.figure(figsize=(10, 8))
plt.imshow(vegetation, cmap="Greens")
plt.title("Píxeles con NDVI > 0.4")
plt.axis("off")
plt.show()
```

No debemos confundir esto con una clasificación científica completa. Es una **regla exploratoria**.

---

# Parte C - NDWI: agua

## 2.7. Qué NDWI vamos a utilizar

Existen varias fórmulas en la literatura llamadas NDWI.

En este manual utilizaremos la formulación de McFeeters basada en verde y NIR:

```text
NDWI = (GREEN - NIR) / (GREEN + NIR)
```

En Sentinel-2:

```text
NDWI = (B03 - B08) / (B03 + B08)
```

El agua suele absorber fuertemente el NIR, por lo que muchas masas de agua adquieren valores positivos.

```python
ndwi = normalized_difference(green, nir)
```

Visualización:

```python
plt.figure(figsize=(10, 8))
plt.imshow(ndwi, vmin=-1, vmax=1, cmap="BrBG")
plt.colorbar(label="NDWI")
plt.title("NDWI - Sentinel-2")
plt.axis("off")
plt.show()
```

Una primera máscara exploratoria podría ser:

```python
water = ndwi > 0.1
```

Pero el umbral debe comprobarse visualmente y, si el análisis es serio, validarse con datos de referencia.

---

## 2.8. NDWI no es NDMI

Es una confusión muy habitual.

El NDMI suele utilizar:

```text
NDMI = (NIR - SWIR) / (NIR + SWIR)
```

Sirve principalmente para estudiar contenido de humedad de la vegetación y del suelo.

En este capítulo usamos **NDWI verde-NIR para agua superficial**.

---

# Parte D - NDBI: superficies construidas

## 2.9. Qué es el NDBI

NDBI significa **Normalized Difference Built-up Index**.

```text
NDBI = (SWIR - NIR) / (SWIR + NIR)
```

Para Sentinel-2 utilizaremos:

```text
NDBI = (B11 - B08) / (B11 + B08)
```

Muchas superficies construidas muestran una reflectancia relativamente alta en SWIR respecto al NIR.

```python
ndbi = normalized_difference(swir1, nir)
```

Pero aquí aparece un problema técnico importante.

B11 tiene una resolución espacial de 20 m y B08 de 10 m.

No podemos hacer:

```python
ndbi = normalized_difference(b11_20m, b08_10m)
```

si sus matrices tienen dimensiones distintas.

Primero hay que alinear B11 con la cuadrícula de B08.

---

# Parte E - Reproyectar B11 de 20 m a la cuadrícula de 10 m

## 2.10. Alinear bandas antes de operar

Con Rasterio:

```python
import numpy as np
from rasterio.warp import reproject, Resampling

swir1_10m = np.empty_like(nir, dtype="float32")

reproject(
    source=swir1_20m,
    destination=swir1_10m,
    src_transform=swir_transform,
    src_crs=swir_crs,
    dst_transform=nir_transform,
    dst_crs=nir_crs,
    resampling=Resampling.bilinear,
)
```

Ahora sí:

```python
ndbi = normalized_difference(swir1_10m, nir)
```

### ¿Bilinear o nearest?

Para reflectancia continua suele ser razonable utilizar interpolación bilineal.

Para máscaras categóricas como SCL debemos utilizar:

```python
Resampling.nearest
```

porque no queremos crear clases intermedias inexistentes.

---

# Parte F - Comparar tres paisajes de Manhattan

## 2.11. El experimento

Vamos a utilizar tres zonas con comportamiento espectral distinto:

1. **Central Park** - vegetación urbana.
2. **Hudson River / costa oeste** - agua.
3. **Midtown Manhattan** - tejido urbano denso.

La idea no es demostrar que un índice sea perfecto, sino comprobar cómo responde cada uno ante superficies conocidas.

Podemos definir tres bounding boxes aproximadas en WGS84:

```python
patches = {
    "central_park": [-73.9819, 40.7644, -73.9497, 40.8008],
    "hudson": [-74.0185, 40.7240, -74.0070, 40.7850],
    "midtown": [-74.0060, 40.7440, -73.9710, 40.7650],
}
```

Los límites son deliberadamente amplios para fines didácticos.

---

## 2.12. Resumir un índice dentro de cada zona

Una función sencilla:

```python

def summarize_index(arr, name):
    values = arr[np.isfinite(arr)]

    return {
        "zone": name,
        "mean": float(np.mean(values)),
        "median": float(np.median(values)),
        "p10": float(np.percentile(values, 10)),
        "p90": float(np.percentile(values, 90)),
        "n": int(values.size),
    }
```

Si disponemos de cada parche ya recortado:

```python
rows = []

for name, arrays in patch_data.items():
    ndvi_patch = normalized_difference(arrays["nir"], arrays["red"])
    ndwi_patch = normalized_difference(arrays["green"], arrays["nir"])

    rows.append({
        "zona": name,
        "ndvi_medio": np.nanmean(ndvi_patch),
        "ndwi_medio": np.nanmean(ndwi_patch),
    })
```

Y podemos pasarlo a un DataFrame:

```python
import pandas as pd

summary = pd.DataFrame(rows)
print(summary)
```

---

## 2.13. Qué esperamos observar

### Central Park

Esperamos NDVI claramente mayor que en Midtown.

### Hudson River

Esperamos NDWI relativamente alto y NDVI bajo o negativo.

### Midtown

Esperamos NDVI bajo y, en algunas zonas, NDBI relativamente alto.

Pero hay que ser prudentes: sombras urbanas, cubiertas oscuras, vegetación de calles, agua turbia, píxeles mixtos y geometría solar pueden alterar la señal.

Este es precisamente uno de los aprendizajes importantes de teledetección:

> Los índices simplifican la señal. No sustituyen el razonamiento sobre el territorio.

---

# Parte G - Visualización comparativa

## 2.14. Tres índices en una misma escena

```python
fig, axes = plt.subplots(1, 3, figsize=(18, 6))

im0 = axes[0].imshow(ndvi, vmin=-1, vmax=1, cmap="RdYlGn")
axes[0].set_title("NDVI")
axes[0].axis("off")

im1 = axes[1].imshow(ndwi, vmin=-1, vmax=1, cmap="BrBG")
axes[1].set_title("NDWI")
axes[1].axis("off")

im2 = axes[2].imshow(ndbi, vmin=-1, vmax=1, cmap="RdBu")
axes[2].set_title("NDBI")
axes[2].axis("off")

plt.tight_layout()
plt.show()
```

La comparación lado a lado ayuda a entender que un mismo píxel puede tener varios atributos simultáneos.

---

# Parte H - Enmascarar nubes antes de analizar índices

## 2.15. Un índice calculado sobre una nube sigue siendo un número

NumPy no sabe qué es una nube.

Si no la eliminamos, calculará igualmente NDVI, NDWI o NDBI.

Por eso debemos reutilizar la máscara SCL del capítulo anterior.

Un ejemplo de clases que podemos excluir:

```python
bad_scl = [0, 1, 3, 8, 9, 10, 11]
```

Después:

```python
valid = ~np.isin(scl_10m, bad_scl)

ndvi_clean = np.where(valid, ndvi, np.nan)
ndwi_clean = np.where(valid, ndwi, np.nan)
ndbi_clean = np.where(valid, ndbi, np.nan)
```

No todas las aplicaciones necesitan exactamente la misma máscara. Debemos decidir qué clases SCL excluir según el análisis.

---

# Parte I - De imagen a tabla: cada píxel como observación

## 2.16. La idea que conecta teledetección y Machine Learning

Hasta ahora pensamos así:

```text
una banda = una imagen
```

A partir de ahora también podemos pensar así:

```text
un píxel = una fila
cada banda o índice = una columna
```

Para un píxel cualquiera:

```text
[B02, B03, B04, B08, B11, NDVI, NDWI, NDBI]
```

Eso es un **vector de características**.

En Machine Learning suele llamarse *feature vector*.

---

## 2.17. Construir la matriz X

Suponiendo que todas las bandas ya están alineadas a la misma cuadrícula:

```python
features = np.stack(
    [
        blue,
        green,
        red,
        nir,
        swir1_10m,
        ndvi_clean,
        ndwi_clean,
        ndbi_clean,
    ],
    axis=-1,
)

print(features.shape)
```

Si la imagen tiene:

```text
1000 filas x 800 columnas
```

la matriz tendrá:

```text
(1000, 800, 8)
```

Ocho características por píxel.

---

## 2.18. Convertirla a formato de Machine Learning

Scikit-learn suele esperar:

```text
(n_muestras, n_features)
```

Por tanto:

```python
X = features.reshape(-1, features.shape[-1])

print(X.shape)
```

El resultado sería:

```text
(800000, 8)
```

Cada fila representa un píxel.

---

## 2.19. Eliminar píxeles no válidos

```python
valid_rows = np.all(np.isfinite(X), axis=1)

X_clean = X[valid_rows]

print("Píxeles totales:", len(X))
print("Píxeles válidos:", len(X_clean))
```

Esta matriz será la base del capítulo de clasificación.

---

# Parte J - Añadir coordenadas a cada píxel

## 2.20. Saber qué fila corresponde a qué lugar

Para análisis geoespacial conviene conservar las coordenadas.

Con Rasterio:

```python
from rasterio.transform import xy

rows, cols = np.indices(ndvi.shape)

xs, ys = xy(
    nir_transform,
    rows,
    cols,
    offset="center",
)

xs = np.asarray(xs)
ys = np.asarray(ys)
```

Ahora podemos generar una tabla:

```python
df = pd.DataFrame({
    "x": xs.ravel(),
    "y": ys.ravel(),
    "blue": blue.ravel(),
    "green": green.ravel(),
    "red": red.ravel(),
    "nir": nir.ravel(),
    "swir1": swir1_10m.ravel(),
    "ndvi": ndvi_clean.ravel(),
    "ndwi": ndwi_clean.ravel(),
    "ndbi": ndbi_clean.ravel(),
})

df = df.dropna()
```

Ahora tenemos datos satelitales en una estructura tabular familiar.

---

# Parte K - Guardar los índices como GeoTIFF

## 2.21. Función reutilizable

```python
import rasterio


def save_index(path, array, reference_profile):
    profile = reference_profile.copy()
    profile.update(
        dtype="float32",
        count=1,
        nodata=np.nan,
        compress="deflate",
    )

    with rasterio.open(path, "w", **profile) as dst:
        dst.write(array.astype("float32"), 1)
```

Uso:

```python
save_index("ndvi.tif", ndvi_clean, nir_profile)
save_index("ndwi.tif", ndwi_clean, nir_profile)
save_index("ndbi.tif", ndbi_clean, nir_profile)
```

Cada archivo conserva CRS, transformación espacial y resolución.

---

# Parte L - Script compacto del capítulo

## 2.22. Pipeline de índices

Este bloque resume la lógica principal una vez que las bandas ya han sido descargadas y convertidas a reflectancia:

```python
import numpy as np
from rasterio.warp import reproject, Resampling


def normalized_difference(a, b):
    a = a.astype("float32")
    b = b.astype("float32")

    den = a + b
    out = np.full(a.shape, np.nan, dtype="float32")

    valid = np.isfinite(a) & np.isfinite(b) & (den != 0)
    out[valid] = (a[valid] - b[valid]) / den[valid]

    return out


# 1. Reproyectar SWIR de 20 m a la cuadrícula NIR de 10 m
swir1_10m = np.empty_like(nir, dtype="float32")

reproject(
    source=swir1_20m,
    destination=swir1_10m,
    src_transform=swir_transform,
    src_crs=swir_crs,
    dst_transform=nir_transform,
    dst_crs=nir_crs,
    resampling=Resampling.bilinear,
)

# 2. Índices
ndvi = normalized_difference(nir, red)
ndwi = normalized_difference(green, nir)
ndbi = normalized_difference(swir1_10m, nir)

# 3. Máscara SCL
bad_scl = [0, 1, 3, 8, 9, 10, 11]
valid = ~np.isin(scl_10m, bad_scl)

ndvi = np.where(valid, ndvi, np.nan)
ndwi = np.where(valid, ndwi, np.nan)
ndbi = np.where(valid, ndbi, np.nan)

# 4. Cubo de características por píxel
features = np.stack(
    [blue, green, red, nir, swir1_10m, ndvi, ndwi, ndbi],
    axis=-1,
)

# 5. Matriz ML
X = features.reshape(-1, features.shape[-1])
X = X[np.all(np.isfinite(X), axis=1)]

print("Matriz final:", X.shape)
```

---

# Parte M - Errores habituales

## 2.23. Error 1: calcular índices con DN sin entender la escala

Lo correcto es trabajar con reflectancia superficial después de aplicar correctamente `scale` y `offset` de los metadatos cuando corresponda.

---

## 2.24. Error 2: mezclar matrices de 10 m y 20 m

Antes de combinar B11 con B08 hay que llevarlas a una cuadrícula común.

---

## 2.25. Error 3: usar interpolación bilineal para clases SCL

Las categorías deben remuestrearse con vecino más próximo.

---

## 2.26. Error 4: interpretar un umbral como verdad universal

`NDVI > 0.4` puede ser útil como regla exploratoria, pero no es una ley física.

---

## 2.27. Error 5: olvidar nubes y sombras

Los índices no detectan por sí mismos qué píxeles son válidos.

---

## 2.28. Error 6: asumir que NDBI equivale a edificio

NDBI puede responder también a suelo desnudo, superficies secas y otros materiales.

En zonas urbanas complejas es mucho más potente combinar múltiples bandas e índices y entrenar un clasificador.

---

# Parte N - Ejercicios

## Ejercicio 1

Calcula NDVI para una escena de tu ciudad y encuentra los 10 píxeles con valores más altos.

## Ejercicio 2

Prueba tres umbrales de vegetación:

```text
0.2
0.4
0.6
```

Compara visualmente cómo cambia la superficie detectada.

## Ejercicio 3

Calcula NDWI y comprueba si un umbral fijo separa correctamente el agua.

## Ejercicio 4

Reproyecta B11 a 10 m y calcula NDBI.

## Ejercicio 5

Compara los valores medios de NDVI, NDWI y NDBI en Central Park, Hudson River y Midtown.

## Ejercicio 6

Crea una matriz por píxel con estas características:

```text
B02, B03, B04, B08, B11, NDVI, NDWI, NDBI
```

## Ejercicio 7 - reto

Añade una novena característica:

```text
NDMI = (B08 - B11) / (B08 + B11)
```

Investiga qué información adicional aporta respecto a NDWI.

---

# Qué hemos aprendido

Al terminar este capítulo ya sabemos pasar de bandas independientes a variables con significado territorial.

Hemos construido:

```text
Bandas Sentinel-2
        ↓
Reflectancia
        ↓
Bandas alineadas
        ↓
NDVI / NDWI / NDBI
        ↓
Máscara de calidad
        ↓
Vector por píxel
        ↓
Matriz X para Machine Learning
```

Este cambio de mentalidad es fundamental.

Ya no tenemos solamente una imagen.

Tenemos un **dataset geoespacial multivariable**.

En el próximo capítulo añadiremos algo que todavía falta: **el tiempo**.

Pasaremos de una sola escena a composiciones temporales, cubos de datos y mapas de cambio.

---

## Referencias técnicas

- Earth Search STAC v1: https://earth-search.aws.element84.com/v1
- Element 84 - Earth Search v1 y colección `sentinel-2-l2a`: https://element84.com/geospatial/introducing-earth-search-v1-new-datasets-now-available/
- Copernicus Sentinel-2: https://dataspace.copernicus.eu/explore-data/data-collections/sentinel-data/sentinel-2

---

[← Volver al índice del manual](../README.md)
