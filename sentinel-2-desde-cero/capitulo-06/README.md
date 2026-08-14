# Capítulo 6 - Proyecto real en Tarragona: pipeline automático de Sentinel-2 a informe técnico

Este capítulo integra todo el manual en un caso práctico reproducible sobre **Tarragona ciudad**, utilizando una zona que incluye el entorno del **río Francolí, el Parc del Francolí, tejido urbano y el frente portuario**.

La elección es deliberada: en una sola AOI aparecen vegetación, agua, superficies construidas, áreas industriales, suelo desnudo y zonas de transición. Es un escenario excelente para comprobar NDVI, NDWI, NDBI, cambios temporales y clasificación.

## 6.1. Objetivo

El usuario solo tendrá que indicar:

- zona de estudio;
- fecha inicial y final;
- nubosidad máxima;
- carpeta de salida.

Python ejecutará:

```text
AOI Tarragona
   ↓
STAC Sentinel-2 L2A
   ↓
selección de escenas
   ↓
lectura remota de bandas COG
   ↓
reflectancia + máscara SCL
   ↓
NDVI / NDWI / NDBI
   ↓
comparación temporal
   ↓
estadísticas
   ↓
GeoTIFF + PNG + CSV + JSON
   ↓
informe PDF
```

---

# Parte A - AOI del ejemplo

## 6.2. Bounding box didáctico

Para cubrir Francolí, ciudad y puerto podemos utilizar, como ejemplo inicial:

```python
bbox_tarragona = [
    1.20,   # oeste
    41.075, # sur
    1.285,  # este
    41.145, # norte
]
```

Esto no sustituye un límite administrativo oficial. Para un informe profesional es preferible utilizar un GeoJSON o GeoPackage con la geometría exacta del ámbito.

## 6.3. Punto de control: Parc del Francolí

El Parc del Francolí se sitúa aproximadamente alrededor de:

```text
latitud  ≈ 41.1260
longitud ≈ 1.2337
```

Es útil como zona de control porque esperamos NDVI relativamente alto respecto a áreas urbanas o portuarias.

---

# Parte B - Configuración del proyecto

## 6.4. Archivo de parámetros

```python
CONFIG = {
    "name": "tarragona_francoli_port",
    "bbox": [1.20, 41.075, 1.285, 41.145],
    "start": "2024-04-01",
    "end": "2025-09-30",
    "cloud_max": 20,
    "green_threshold": 0.40,
    "change_threshold": 0.10,
    "output_dir": "outputs/tarragona",
}
```

Separar parámetros de código facilita repetir el análisis en otro lugar sin reescribir funciones.

---

# Parte C - Buscar escenas

## 6.5. Consulta STAC

```python
import pystac_client

catalog = pystac_client.Client.open(
    "https://earth-search.aws.element84.com/v1"
)

search = catalog.search(
    collections=["sentinel-2-l2a"],
    bbox=CONFIG["bbox"],
    datetime=f"{CONFIG['start']}/{CONFIG['end']}",
    query={
        "eo:cloud_cover": {"lt": CONFIG["cloud_max"]}
    },
)

items = sorted(
    list(search.items()),
    key=lambda x: x.datetime,
)

print("Escenas encontradas:", len(items))
```

Guardar inventario:

```python
import pandas as pd

inventory = pd.DataFrame([
    {
        "id": item.id,
        "date": item.datetime,
        "cloud_cover": item.properties.get("eo:cloud_cover"),
    }
    for item in items
])

inventory.to_csv(
    "outputs/tarragona/inventario_escenas.csv",
    index=False,
)
```

---

# Parte D - Bandas e índices

## 6.6. Bandas principales

Usaremos:

```text
B03 → verde
B04 → rojo
B08 → NIR
B11 → SWIR
SCL → calidad
```

Después de aplicar escala/offset y alinear cuadrículas:

```python
ndvi = normalized_difference(nir, red)
ndwi = normalized_difference(green, nir)
ndbi = normalized_difference(swir1_10m, nir)
```

## 6.7. Qué esperamos ver en Tarragona

### Parc del Francolí

Esperamos NDVI positivo y relativamente alto en superficies con vegetación activa.

### Río Francolí

NDWI puede responder a láminas de agua, aunque el comportamiento real dependerá de caudal, turbidez, anchura y mezcla de píxeles.

### Puerto de Tarragona

Esperamos una combinación de:

- NDVI bajo;
- NDBI relativamente elevado en superficies industriales;
- NDWI alto sobre dársenas y mar;
- fuertes contrastes espectrales entre agua, explanadas, cubiertas y zonas verdes.

### Tejido urbano

Los píxeles mixtos pueden combinar edificios, árboles, calles y sombras. Esto es ideal para comprobar los límites de una interpretación simple por umbrales.

---

# Parte E - Composiciones temporales

## 6.8. Evitar depender de una sola imagen

Agruparemos observaciones válidas por periodos equivalentes.

Ejemplo:

```text
Primavera-verano 2024
vs
Primavera-verano 2025
```

Una composición mediana:

```python
def median_composite(arrays):
    stack = np.stack(arrays, axis=0)
    return np.nanmedian(stack, axis=0).astype("float32")
```

---

# Parte F - Cambio de vegetación

## 6.9. ΔNDVI

```python
delta_ndvi = ndvi_2025 - ndvi_2024
```

Clasificación didáctica:

```python
threshold = CONFIG["change_threshold"]

green_up = delta_ndvi > threshold
green_down = delta_ndvi < -threshold
stable = np.abs(delta_ndvi) <= threshold
```

No debemos interpretar automáticamente un `green_down` como degradación. Puede ser estacionalidad, poda, sequía, cambio temporal de cobertura, sombras o diferencias de observación.

---

# Parte G - Estadísticas por sectores

## 6.10. Crear subzonas de análisis

Podemos definir polígonos para:

```text
Parc del Francolí
tramo urbano del Francolí
centro urbano
puerto
litoral
```

Con GeoPandas:

```python
zones = gpd.read_file("input/zonas_tarragona.geojson")
zones = zones.to_crs(raster_crs)
```

Y calcular:

```text
NDVI medio
NDWI medio
NDBI medio
% verde
ΔNDVI medio
superficie green-up
superficie green-down
```

---

# Parte H - Indicadores

## 6.11. Tabla final

Una estructura útil:

```text
zona | ndvi_2024 | ndvi_2025 | delta_ndvi | green_pct | green_up_ha | green_down_ha
```

En Python:

```python
summary = pd.DataFrame(results)
summary.to_csv(
    "outputs/tarragona/resumen_zonas.csv",
    index=False,
)
```

---

# Parte I - Mapas

## 6.12. Mapa NDVI

```python
plt.figure(figsize=(10, 8))
plt.imshow(ndvi_2025, vmin=-1, vmax=1, cmap="RdYlGn")
plt.colorbar(label="NDVI")
plt.title("Tarragona - NDVI")
plt.axis("off")
plt.tight_layout()
plt.savefig("outputs/tarragona/ndvi_2025.png", dpi=250)
plt.close()
```

## 6.13. Mapa de cambio

```python
plt.figure(figsize=(10, 8))
plt.imshow(delta_ndvi, vmin=-0.5, vmax=0.5, cmap="RdYlGn")
plt.colorbar(label="ΔNDVI")
plt.title("Tarragona - Cambio de NDVI")
plt.axis("off")
plt.tight_layout()
plt.savefig("outputs/tarragona/delta_ndvi.png", dpi=250)
plt.close()
```

## 6.14. Mapa NDBI

Puede ser especialmente interesante en el entorno portuario e industrial:

```python
plt.figure(figsize=(10, 8))
plt.imshow(ndbi, vmin=-1, vmax=1, cmap="RdBu")
plt.colorbar(label="NDBI")
plt.title("Tarragona - Índice construido")
plt.axis("off")
plt.tight_layout()
plt.savefig("outputs/tarragona/ndbi.png", dpi=250)
plt.close()
```

---

# Parte J - Generación del informe

## 6.15. Estructura profesional

```text
PORTADA
Análisis multitemporal Sentinel-2
Tarragona - Francolí y entorno portuario

1. Resumen ejecutivo
2. Objeto
3. Ámbito de estudio
4. Datos utilizados
5. Metodología
   5.1 Catálogo STAC
   5.2 Sentinel-2 L2A
   5.3 Máscara SCL
   5.4 NDVI / NDWI / NDBI
   5.5 Composición temporal
   5.6 Detección de cambio
6. Resultados
   6.1 Parc del Francolí
   6.2 Río Francolí
   6.3 Ciudad
   6.4 Puerto
7. Comparación temporal
8. Limitaciones
9. Conclusiones
10. Archivos generados
```

---

## 6.16. Texto automático del resumen

```python
executive_summary = f"""
Se ha realizado un análisis multitemporal con imágenes Sentinel-2 L2A sobre
el ámbito Tarragona-Francolí-Puerto para el periodo {CONFIG['start']} a
{CONFIG['end']}.

El estudio combina índices NDVI, NDWI y NDBI, composiciones temporales y
estadísticas espaciales. Los cambios detectados representan variaciones
espectrales observadas por Sentinel-2 y no implican por sí mismos una causa
ambiental, urbanística o industrial concreta.
"""
```

---

# Parte K - Informe PDF automático

## 6.17. Función simplificada

```python
from reportlab.lib.pagesizes import A4
from reportlab.lib.styles import getSampleStyleSheet
from reportlab.lib.units import cm
from reportlab.platypus import (
    SimpleDocTemplate,
    Paragraph,
    Spacer,
    Image,
    PageBreak,
)


def build_tarragona_report(output="outputs/tarragona/informe.pdf"):
    doc = SimpleDocTemplate(
        output,
        pagesize=A4,
        leftMargin=2*cm,
        rightMargin=2*cm,
        topMargin=2*cm,
        bottomMargin=2*cm,
    )

    styles = getSampleStyleSheet()
    story = []

    story.append(Paragraph(
        "Análisis multitemporal Sentinel-2 - Tarragona",
        styles["Title"],
    ))

    story.append(Spacer(1, 0.5*cm))
    story.append(Paragraph(
        "Ámbito Francolí, ciudad y entorno portuario",
        styles["Heading2"],
    ))

    story.append(PageBreak())
    story.append(Paragraph("1. Resumen ejecutivo", styles["Heading1"]))
    story.append(Paragraph(executive_summary, styles["BodyText"]))

    story.append(PageBreak())
    story.append(Paragraph("2. NDVI", styles["Heading1"]))
    story.append(Image(
        "outputs/tarragona/ndvi_2025.png",
        width=15*cm,
        height=12*cm,
    ))

    story.append(PageBreak())
    story.append(Paragraph("3. Cambio temporal", styles["Heading1"]))
    story.append(Image(
        "outputs/tarragona/delta_ndvi.png",
        width=15*cm,
        height=12*cm,
    ))

    story.append(PageBreak())
    story.append(Paragraph("4. NDBI", styles["Heading1"]))
    story.append(Image(
        "outputs/tarragona/ndbi.png",
        width=15*cm,
        height=12*cm,
    ))

    story.append(Paragraph("5. Limitaciones", styles["Heading1"]))
    story.append(Paragraph(
        "Los resultados dependen de la nubosidad, las fechas seleccionadas, "
        "la resolución espacial de Sentinel-2, la presencia de píxeles "
        "mixtos, sombras y el método de composición temporal. Las variaciones "
        "espectrales no demuestran por sí solas causalidad.",
        styles["BodyText"],
    ))

    doc.build(story)
```

---

# Parte L - Productos de salida

## 6.18. Estructura final

```text
outputs/tarragona/
├── inventario_escenas.csv
├── ndvi_2024.tif
├── ndvi_2025.tif
├── ndwi.tif
├── ndbi.tif
├── delta_ndvi.tif
├── green_up.tif
├── valid_observations.tif
├── resumen_zonas.csv
├── estadisticas.json
├── ndvi_2025.png
├── delta_ndvi.png
├── ndbi.png
└── informe.pdf
```

---

# Parte M - Control de calidad

## 6.19. Checklist

- [ ] AOI documentada.
- [ ] CRS comprobado.
- [ ] Fechas indicadas.
- [ ] Inventario de escenas conservado.
- [ ] Nubosidad global y máscara SCL diferenciadas.
- [ ] Bandas alineadas antes de operar.
- [ ] Scale/offset revisados.
- [ ] NDVI/NDWI/NDBI calculados con píxeles válidos.
- [ ] Comparación entre periodos equivalentes.
- [ ] Áreas calculadas en CRS proyectado.
- [ ] Mapas exportados.
- [ ] Estadísticas guardadas en CSV/JSON.
- [ ] Limitaciones incluidas en el informe.
- [ ] Conclusiones no atribuyen causalidad sin evidencia adicional.

---

# Parte N - Función maestra

## 6.20. Ejecutar todo con una orden

El objetivo final es llegar a algo parecido a:

```python
def run_project(config):
    aoi = load_aoi(config)
    items = search_sentinel2(config, aoi)
    inventory = save_inventory(items, config)

    composites = build_composites(
        items,
        aoi,
        config,
    )

    indices = calculate_indices(composites)
    change = calculate_change(indices, config)
    statistics = calculate_statistics(indices, change, aoi)

    export_geotiffs(indices, change, config)
    export_tables(statistics, inventory, config)
    create_maps(indices, change, config)
    build_report(statistics, config)

    return statistics
```

Y ejecutarlo:

```python
results = run_project(CONFIG)
```

---

# Qué aporta este capítulo

Los capítulos anteriores enseñaban piezas. Aquí construimos un sistema.

```text
UNA ZONA DE TARRAGONA
        ↓
DATOS SATELITALES REALES
        ↓
PROCESAMIENTO REPRODUCIBLE
        ↓
INDICADORES
        ↓
CAMBIO TEMPORAL
        ↓
MAPAS
        ↓
INFORME TÉCNICO
```

El mismo proyecto puede reutilizarse cambiando únicamente el AOI y las fechas.

En siguientes capítulos podemos especializar este pipeline en casos sectoriales: restauración minera, seguimiento de canteras, incendios, agricultura, litoral y control ambiental de obras.

---

## Referencias

- Earth Search STAC v1: https://earth-search.aws.element84.com/v1
- Copernicus Sentinel-2: https://dataspace.copernicus.eu/explore-data/data-collections/sentinel-data/sentinel-2
- Rasterio: https://rasterio.readthedocs.io/
- GeoPandas: https://geopandas.org/
- ReportLab: https://docs.reportlab.com/

---

[← Volver al índice del manual](../README.md)
