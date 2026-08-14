# Capitulo 0 - Preparar el terreno

## Entender Sentinel-2 y dejar Python listo

Este primer capitulo construye la base conceptual y tecnica del manual. El objetivo es terminar con una idea clara de como una observacion satelital se transforma en datos que Python puede buscar, descargar y analizar, y dejar preparado un entorno reproducible para los siguientes capitulos.

## Que vamos a construir

- Buscar imagenes Sentinel-2 mediante un catalogo STAC.
- Leer bandas raster con Python y trabajar con ventanas espaciales.
- Entender reflectancia, nubes, mascaras y resoluciones.
- Calcular posteriormente NDVI, NDWI y NDBI con NumPy.
- Construir composiciones temporales y detectar cambios.
- Cruzar pixeles con barrios, hexagonos H3 y poblacion.
- Crear etiquetas a partir de OpenStreetMap y entrenar clasificadores.

## 1. Un satelite no ve como una camara

Sentinel-2 mide radiacion electromagnetica reflejada por la superficie terrestre en distintas regiones del espectro. El instrumento MSI registra 13 bandas espectrales desde el visible hasta el infrarrojo de onda corta (SWIR).

Bandas principales que utilizaremos:

| Banda | Uso orientativo | Resolucion |
|---|---|---:|
| B2 | Azul | 10 m |
| B3 | Verde | 10 m |
| B4 | Rojo | 10 m |
| B8 | Infrarrojo cercano (NIR) | 10 m |
| B5, B6, B7, B8A | Red-edge / vegetacion | 20 m |
| B11, B12 | SWIR / humedad, suelo, incendios | 20 m |
| B1, B9, B10 | Atmosfera / aerosoles / vapor / cirros | 60 m |

A 10 m, un pixel representa aproximadamente un cuadrado de 10 x 10 m sobre el terreno. Esto no significa que podamos identificar cualquier objeto de 10 m: el pixel contiene la senal combinada de lo que haya dentro.

## 2. El raster: la estructura basica

Una banda satelital puede imaginarse como una gran matriz. Cada posicion tiene fila, columna y valor, ademas de informacion espacial para saber que lugar del planeta representa.

```python
import numpy as np

raster = np.array([
    [12, 14, 15, 18, 20],
    [11, 13, 16, 19, 22],
    [10, 12, 15, 21, 25],
    [ 9, 11, 14, 20, 28],
], dtype=np.float32)

print(raster.shape)
print(raster[2, 3])
```

Mas adelante sustituiremos esta matriz de juguete por bandas reales. La logica numerica sera la misma: seleccionar, comparar, enmascarar, combinar y resumir matrices.

## 3. De DN a reflectancia

Los valores almacenados en un producto satelital no siempre son directamente porcentajes de luz reflejada. En Sentinel-2 Level-2A se trabaja con reflectancia de superficie codificada mediante una escala. Antes de aplicar formulas debemos consultar los metadatos del activo y convertir los datos a unidades coherentes.

**Regla importante:** no aplicar una constante memorizada a ciegas. En el Capitulo 1 leeremos los metadatos del propio producto y haremos explicita la conversion.

## 4. L1C frente a L2A

- **Level-1C:** reflectancia en la parte superior de la atmosfera (TOA), ortorrectificada.
- **Level-2A:** reflectancia de superficie (BOA) despues de correccion atmosferica y con informacion de clasificacion de escena.

Para estudiar vegetacion, agua o suelo a traves del tiempo, Level-2A sera nuestro punto de partida habitual.

## 5. STAC

STAC significa *SpatioTemporal Asset Catalog*. Es un estandar para describir y buscar datos geoespaciales.

Conceptos basicos:

1. **Catalog / Collection:** familia de datos.
2. **Item:** observacion concreta en una fecha y zona.
3. **Asset:** fichero asociado al item, como una banda o miniatura.
4. **Geometry / bbox / datetime:** filtros espaciales y temporales.

En 2026, Copernicus Data Space Ecosystem dispone de un catalogo STAC oficial en `https://stac.dataspace.copernicus.eu/v1/`.

## 6. Preparar Python

Crearemos un entorno conda independiente:

```bash
conda create -n sentinel2-manual -c conda-forge python=3.12 numpy pandas matplotlib requests rasterio geopandas shapely pyproj pystac-client stackstac xarray rioxarray h3 scikit-learn jupyterlab -y
conda activate sentinel2-manual
```

## 7. Programa de diagnostico

Crea `check_setup.py`:

```python
import importlib
import json
import sys
from urllib.request import urlopen

PACKAGES = [
    'numpy', 'pandas', 'matplotlib', 'rasterio', 'geopandas',
    'shapely', 'pyproj', 'pystac_client', 'xarray', 'rioxarray',
    'sklearn'
]

print(f'Python: {sys.version.split()[0]}')

for name in PACKAGES:
    mod = importlib.import_module(name)
    version = getattr(mod, '__version__', 'sin __version__')
    print(f'[OK] {name:14s} {version}')

url = 'https://stac.dataspace.copernicus.eu/v1/'
with urlopen(url, timeout=20) as response:
    data = json.load(response)

print('[OK] Red y catalogo STAC accesibles')
print('Catalogo:', data.get('title', data.get('id', 'sin titulo')))
print('TODO LISTO')
```

Ejecutalo con:

```bash
python check_setup.py
```

Si todo termina en `TODO LISTO`, tenemos el entorno preparado para comenzar a trabajar con imagenes Sentinel-2 reales.

---

## Siguiente capitulo

**Capitulo 1 - Rasters y datos Sentinel-2 reales:** crear un raster desde cero, buscar una escena real mediante STAC, seleccionar bandas, leer solo la zona necesaria, convertir a reflectancia y aplicar mascaras espaciales y de clasificacion.

*Serie Sentinel-2 desde cero en Python - Edicion 2026.*
