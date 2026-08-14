# Capítulo 3 - La dimensión temporal: composiciones, cubos de datos y mapas de cambio

Una imagen Sentinel-2 es una fotografía espectral de un instante. Una serie temporal permite estudiar **procesos**.

En este capítulo pasamos de preguntar «¿cómo está el territorio?» a preguntas mucho más potentes:

- ¿Cómo ha cambiado la vegetación entre dos fechas?
- ¿Qué zonas se han reverdecido y cuáles han perdido vigor?
- ¿Cómo reducimos el efecto de nubes y escenas anómalas?
- ¿Cómo construimos un cubo trimestral de dos años?
- ¿Cómo resumimos el cambio con estadísticas reproducibles?
- ¿Cómo convertimos todo el análisis en un **informe técnico automático**?

El flujo será:

```text
STAC → varias escenas → máscara de calidad → NDVI por fecha
     → composiciones medianas → cubo temporal
     → diferencias / tendencias → mapas y estadísticas
     → informe técnico
```

---

## 3.1. Por qué necesitamos tiempo

Dos píxeles pueden tener hoy el mismo NDVI y haber seguido historias completamente diferentes. Uno puede ser un parque estable y otro una parcela recién revegetada.

La dimensión temporal permite separar **estado** de **evolución**.

Sentinel-2A, 2B y 2C forman una constelación diseñada para observación repetitiva de la superficie terrestre. Para un análisis serio no debemos asumir que cada fecha tendrá una escena perfecta: habrá nubes, sombras, diferencias estacionales y observaciones incompletas.

---

## 3.2. Buscar varias escenas con STAC

Partimos del catálogo Earth Search:

```python
import pystac_client

catalog = pystac_client.Client.open(
    "https://earth-search.aws.element84.com/v1"
)

bbox = [-74.03, 40.69, -73.93, 40.83]

search = catalog.search(
    collections=["sentinel-2-l2a"],
    bbox=bbox,
    datetime="2024-01-01/2025-12-31",
    query={"eo:cloud_cover": {"lt": 30}},
)

items = list(search.items())
print("Escenas encontradas:", len(items))
```

El porcentaje global de nubes sirve para filtrar candidatos, pero **no sustituye** la máscara de calidad sobre nuestra zona de estudio.

---

## 3.3. Ordenar y auditar las escenas

```python
items = sorted(items, key=lambda item: item.datetime)

for item in items[:10]:
    print(
        item.datetime.date(),
        item.properties.get("eo:cloud_cover")
    )
```

Conviene guardar una tabla de procedencia:

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

inventory.to_csv("inventario_escenas.csv", index=False)
```

Esta tabla será muy útil en el informe final: un análisis reproducible debe poder explicar qué observaciones utilizó.

---

# Parte A - Una serie NDVI

## 3.4. Calcular NDVI para cada fecha

Reutilizamos la función del capítulo 2:

```python
import numpy as np


def normalized_difference(a, b):
    a = a.astype("float32")
    b = b.astype("float32")
    den = a + b
    out = np.full(a.shape, np.nan, dtype="float32")
    valid = np.isfinite(a) & np.isfinite(b) & (den != 0)
    out[valid] = (a[valid] - b[valid]) / den[valid]
    return out
```

Para cada escena:

```python
ndvi = normalized_difference(nir, red)
ndvi = np.where(valid_scl, ndvi, np.nan)
```

El requisito crítico es que todas las fechas terminen sobre **la misma cuadrícula espacial**: mismo CRS, resolución, transform, ancho y alto.

---

## 3.5. Apilar fechas

Si `ndvi_list` contiene matrices alineadas:

```python
ndvi_cube = np.stack(ndvi_list, axis=0)
print(ndvi_cube.shape)
```

Podríamos obtener:

```text
(24, 1200, 900)
```

que significa:

```text
24 fechas × 1200 filas × 900 columnas
```

Ya tenemos un pequeño **cubo de datos**.

---

# Parte B - Composiciones medianas

## 3.6. Por qué no elegir simplemente «la mejor imagen»

Una escena con poca nubosidad global puede contener una nube exactamente encima de nuestra zona. Además, una sola observación puede ser anómala.

Una estrategia muy útil consiste en combinar varias observaciones válidas mediante la mediana:

```python
median_ndvi = np.nanmedian(ndvi_cube, axis=0)
```

La mediana suele ser más robusta frente a valores extremos que la media.

---

## 3.7. Composición trimestral

Agrupamos las fechas en trimestres:

```python
inventory["quarter"] = inventory["date"].dt.to_period("Q")
```

Conceptualmente queremos construir:

```text
2024Q1 → mediana de observaciones válidas
2024Q2 → mediana
2024Q3 → mediana
2024Q4 → mediana
2025Q1 → mediana
...
```

Una función:

```python

def median_composite(arrays):
    stack = np.stack(arrays, axis=0)
    return np.nanmedian(stack, axis=0).astype("float32")
```

---

## 3.8. No confundir ausencia de datos con valor cero

Un píxel sin observaciones válidas debe permanecer como `NaN` o nodata.

```python
valid_count = np.sum(np.isfinite(ndvi_cube), axis=0)
```

Podemos exigir, por ejemplo, al menos tres observaciones:

```python
median_ndvi[valid_count < 3] = np.nan
```

El mapa de `valid_count` es un producto de control de calidad y merece aparecer en un informe técnico.

---

# Parte C - Cubo trimestral de dos años

## 3.9. Estructura del cubo

Para ocho trimestres:

```text
(time, y, x) = (8, filas, columnas)
```

Podemos guardar:

```python
quarter_cube = np.stack(quarterly_composites, axis=0)
```

Y conservar las etiquetas:

```python
quarters = [
    "2024Q1", "2024Q2", "2024Q3", "2024Q4",
    "2025Q1", "2025Q2", "2025Q3", "2025Q4",
]
```

Para proyectos grandes, `xarray` facilita trabajar con dimensiones etiquetadas:

```python
import xarray as xr

cube = xr.DataArray(
    quarter_cube,
    dims=("time", "y", "x"),
    coords={"time": quarters},
    name="NDVI",
)
```

---

# Parte D - Cambio entre dos periodos

## 3.10. Diferencia de NDVI

La operación más sencilla es:

```text
ΔNDVI = NDVI_final - NDVI_inicial
```

```python
delta_ndvi = ndvi_final - ndvi_initial
```

Interpretación básica:

```text
ΔNDVI > 0 → aumento relativo de verdor
ΔNDVI < 0 → disminución relativa de verdor
ΔNDVI ≈ 0 → estabilidad
```

Pero no debemos llamar automáticamente «degradación» a todo valor negativo. Puede deberse a estacionalidad, siega, cosecha, sequía temporal, sombras o diferencias de observación.

---

## 3.11. Comparar periodos equivalentes

Para detectar cambio estructural es mejor comparar estaciones equivalentes:

```text
2024Q2 vs 2025Q2
```

que comparar:

```text
2024Q1 vs 2025Q3
```

porque en el segundo caso mezclamos cambio real con ciclo estacional.

---

# Parte E - Green-up map

## 3.12. Mapa de reverdecimiento

Definimos una tolerancia para evitar interpretar ruido mínimo:

```python
threshold = 0.10

green_up = delta_ndvi > threshold
green_down = delta_ndvi < -threshold
stable = np.abs(delta_ndvi) <= threshold
```

Podemos construir una clasificación:

```python
change_class = np.zeros(delta_ndvi.shape, dtype="int8")
change_class[green_down] = -1
change_class[stable] = 0
change_class[green_up] = 1
```

El valor `0.10` es didáctico. En un estudio real debe justificarse mediante distribución de diferencias, incertidumbre y/o validación.

---

## 3.13. Cuantificar superficie

Si trabajamos en una cuadrícula proyectada de 10 × 10 m:

```python
pixel_area_m2 = 10 * 10

green_up_area = np.sum(green_up) * pixel_area_m2
green_down_area = np.sum(green_down) * pixel_area_m2

print("Reverdecimiento m²:", green_up_area)
print("Pérdida m²:", green_down_area)
```

En hectáreas:

```python
green_up_ha = green_up_area / 10_000
green_down_ha = green_down_area / 10_000
```

No debemos calcular áreas directamente a partir de grados de longitud/latitud como si cada píxel tuviera una superficie constante.

---

# Parte F - Tendencia temporal por píxel

## 3.14. Más allá de dos fechas

Una diferencia inicio-fin ignora todo lo que sucede entre medias. Podemos estimar una pendiente temporal.

```python

def pixel_slope(series):
    valid = np.isfinite(series)
    if valid.sum() < 4:
        return np.nan

    t = np.arange(len(series))[valid]
    y = series[valid]

    return np.polyfit(t, y, 1)[0]
```

Aplicación didáctica:

```python
trend = np.apply_along_axis(pixel_slope, 0, quarter_cube)
```

Para áreas muy grandes conviene utilizar enfoques vectorizados o procesamiento por bloques.

Una pendiente positiva indica tendencia creciente de NDVI; una negativa, decreciente. Sigue sin demostrar por sí sola una causa.

---

# Parte G - Serie temporal de una zona

## 3.15. Pasar de píxeles a una curva

Para cada trimestre:

```python
mean_series = np.nanmean(quarter_cube, axis=(1, 2))
median_series = np.nanmedian(quarter_cube, axis=(1, 2))
```

```python
import matplotlib.pyplot as plt

plt.figure(figsize=(10, 5))
plt.plot(quarters, median_series, marker="o")
plt.ylabel("NDVI mediano")
plt.xlabel("Trimestre")
plt.title("Evolución temporal del NDVI")
plt.xticks(rotation=45)
plt.tight_layout()
plt.savefig("serie_ndvi.png", dpi=200)
plt.show()
```

Esta figura será una de las piezas principales del informe.

---

# Parte H - Estadísticas de cambio

## 3.16. Crear indicadores objetivos

```python
valid = np.isfinite(delta_ndvi)

stats = {
    "ndvi_inicial_medio": float(np.nanmean(ndvi_initial)),
    "ndvi_final_medio": float(np.nanmean(ndvi_final)),
    "delta_medio": float(np.nanmean(delta_ndvi)),
    "delta_mediano": float(np.nanmedian(delta_ndvi)),
    "green_up_ha": float(green_up_ha),
    "green_down_ha": float(green_down_ha),
    "pixels_validos": int(valid.sum()),
}
```

También podemos calcular porcentajes:

```python
n = valid.sum()

stats["green_up_pct"] = 100 * np.sum(green_up & valid) / n
stats["green_down_pct"] = 100 * np.sum(green_down & valid) / n
stats["stable_pct"] = 100 * np.sum(stable & valid) / n
```

---

# Parte I - Guardar productos reproducibles

## 3.17. Productos mínimos recomendados

Un análisis temporal profesional debería conservar, como mínimo:

```text
outputs/
├── inventario_escenas.csv
├── ndvi_periodo_inicial.tif
├── ndvi_periodo_final.tif
├── delta_ndvi.tif
├── green_up_class.tif
├── valid_observations.tif
├── serie_ndvi.csv
├── serie_ndvi.png
├── mapa_cambio.png
├── estadisticas.json
└── informe_temporal.pdf
```

Los GeoTIFF conservan la georreferenciación; los CSV/JSON permiten auditar cifras; PNG sirve para comunicación; PDF reúne la interpretación.

---

# Parte J - Cómo hacer un informe técnico automático

## 3.18. Qué debe responder el informe

Un buen informe no consiste en pegar mapas. Debe responder de forma explícita:

1. **Qué se ha analizado.**
2. **Dónde está la zona de estudio.**
3. **Qué datos Sentinel-2 se utilizaron.**
4. **Qué fechas y criterios de calidad se aplicaron.**
5. **Cómo se calculó el indicador.**
6. **Qué cambios se detectaron.**
7. **Cuánta superficie representa cada cambio.**
8. **Qué limitaciones tiene el análisis.**
9. **Qué conclusión puede sostenerse y cuál no.**

---

## 3.19. Estructura recomendada del informe

```text
PORTADA
Título del estudio
Zona
Periodo
Fecha de elaboración
Autor

1. RESUMEN EJECUTIVO
2. OBJETO Y ALCANCE
3. ZONA DE ESTUDIO
4. DATOS Y FUENTES
5. METODOLOGÍA
   5.1 Búsqueda STAC
   5.2 Sentinel-2 L2A
   5.3 Máscara SCL
   5.4 NDVI
   5.5 Composición temporal
   5.6 Detección de cambio
6. RESULTADOS
   6.1 Serie temporal
   6.2 Mapa inicial
   6.3 Mapa final
   6.4 ΔNDVI
   6.5 Superficies de cambio
7. INTERPRETACIÓN
8. LIMITACIONES
9. CONCLUSIONES
10. REPRODUCIBILIDAD Y ARCHIVOS GENERADOS
```

---

## 3.20. Generar primero los gráficos

Ejemplo de mapa de cambio:

```python
plt.figure(figsize=(9, 8))
plt.imshow(delta_ndvi, vmin=-0.5, vmax=0.5, cmap="RdYlGn")
plt.colorbar(label="ΔNDVI")
plt.title("Cambio de NDVI")
plt.axis("off")
plt.tight_layout()
plt.savefig("mapa_cambio.png", dpi=250, bbox_inches="tight")
plt.close()
```

Guardar las estadísticas:

```python
import json

with open("estadisticas.json", "w", encoding="utf-8") as f:
    json.dump(stats, f, ensure_ascii=False, indent=2)
```

---

## 3.21. Generar un informe PDF con ReportLab

Para un informe sencillo y completamente automatizado podemos utilizar ReportLab:

```bash
pip install reportlab
```

Ejemplo mínimo:

```python
from reportlab.lib import colors
from reportlab.lib.pagesizes import A4
from reportlab.lib.styles import getSampleStyleSheet
from reportlab.lib.units import cm
from reportlab.platypus import (
    SimpleDocTemplate,
    Paragraph,
    Spacer,
    Image,
    Table,
    TableStyle,
    PageBreak,
)


def build_report(stats, output="informe_temporal.pdf"):
    doc = SimpleDocTemplate(
        output,
        pagesize=A4,
        rightMargin=2 * cm,
        leftMargin=2 * cm,
        topMargin=2 * cm,
        bottomMargin=2 * cm,
    )

    styles = getSampleStyleSheet()
    story = []

    story.append(Paragraph(
        "Análisis temporal de vegetación con Sentinel-2",
        styles["Title"],
    ))
    story.append(Spacer(1, 0.5 * cm))

    story.append(Paragraph("1. Resumen ejecutivo", styles["Heading1"]))

    summary = (
        f"El análisis muestra un cambio medio de NDVI de "
        f"{stats['delta_medio']:.3f}. "
        f"La superficie clasificada como reverdecimiento es "
        f"{stats['green_up_ha']:.2f} ha y la superficie con "
        f"disminución es {stats['green_down_ha']:.2f} ha."
    )

    story.append(Paragraph(summary, styles["BodyText"]))
    story.append(Spacer(1, 0.4 * cm))

    story.append(Paragraph("2. Resultados", styles["Heading1"]))

    table_data = [
        ["Indicador", "Resultado"],
        ["NDVI inicial medio", f"{stats['ndvi_inicial_medio']:.3f}"],
        ["NDVI final medio", f"{stats['ndvi_final_medio']:.3f}"],
        ["ΔNDVI medio", f"{stats['delta_medio']:.3f}"],
        ["Reverdecimiento", f"{stats['green_up_ha']:.2f} ha"],
        ["Disminución", f"{stats['green_down_ha']:.2f} ha"],
    ]

    table = Table(table_data, colWidths=[8 * cm, 6 * cm])
    table.setStyle(TableStyle([
        ("GRID", (0, 0), (-1, -1), 0.5, colors.grey),
        ("BACKGROUND", (0, 0), (-1, 0), colors.lightgrey),
        ("FONTNAME", (0, 0), (-1, 0), "Helvetica-Bold"),
        ("PADDING", (0, 0), (-1, -1), 6),
    ]))

    story.append(table)
    story.append(PageBreak())

    story.append(Paragraph("3. Serie temporal", styles["Heading1"]))
    story.append(Image("serie_ndvi.png", width=16 * cm, height=8 * cm))

    story.append(Spacer(1, 0.5 * cm))
    story.append(Paragraph("4. Mapa de cambio", styles["Heading1"]))
    story.append(Image("mapa_cambio.png", width=15 * cm, height=13 * cm))

    story.append(Paragraph("5. Limitaciones", styles["Heading1"]))
    story.append(Paragraph(
        "Los resultados deben interpretarse considerando estacionalidad, "
        "nubosidad residual, sombras, disponibilidad de observaciones, "
        "resolución espacial de Sentinel-2 y ausencia de validación de campo.",
        styles["BodyText"],
    ))

    doc.build(story)


build_report(stats)
```

El resultado será:

```text
informe_temporal.pdf
```

---

## 3.22. Evitar informes que inventan conclusiones

La automatización debe insertar **cifras calculadas**, no fabricar explicaciones causales.

Correcto:

```text
Entre los dos periodos comparados, el 18,4 % de los píxeles válidos
presentó un incremento de NDVI superior al umbral definido.
```

Incorrecto sin evidencia adicional:

```text
La mejora se debe a las políticas municipales de plantación de árboles.
```

Sentinel-2 puede mostrar un patrón compatible con un cambio. Para atribuir la causa necesitamos otras fuentes.

---

## 3.23. Plantilla para redactar resultados

Podemos generar el texto con variables:

```python
result_text = f"""
Se analizaron composiciones equivalentes de los periodos seleccionados.
El NDVI medio pasó de {stats['ndvi_inicial_medio']:.3f}
a {stats['ndvi_final_medio']:.3f}, con una variación media de
{stats['delta_medio']:.3f}.

Aplicando el umbral de cambio definido, se identificaron
{stats['green_up_ha']:.2f} ha con incremento y
{stats['green_down_ha']:.2f} ha con disminución de NDVI.
Estos resultados describen cambios espectrales y no implican por sí solos
una causa ecológica o urbanística concreta.
"""
```

Esta forma de redacción es especialmente útil para generar informes de muchas parcelas, municipios o explotaciones sin perder trazabilidad.

---

# Parte K - Control de calidad del informe

## 3.24. Checklist antes de entregar

- [ ] CRS y resolución documentados.
- [ ] Periodo temporal indicado.
- [ ] IDs de escenas conservados.
- [ ] Criterio de nubosidad indicado.
- [ ] Clases SCL excluidas documentadas.
- [ ] Fórmula NDVI indicada.
- [ ] Método de composición explicado.
- [ ] Número de observaciones válidas comprobado.
- [ ] Comparación entre estaciones equivalentes cuando procede.
- [ ] Umbral de cambio justificado.
- [ ] Superficies calculadas en CRS proyectado.
- [ ] Mapas con leyenda y unidades.
- [ ] Limitaciones incluidas.
- [ ] Conclusiones separan observación de causalidad.
- [ ] Código, CSV y GeoTIFF conservados para reproducibilidad.

---

# Parte L - Script conceptual completo

## 3.25. Pipeline final

```python
# 1. Buscar escenas STAC
items = search_sentinel2(aoi, start_date, end_date)

# 2. Para cada escena
for item in items:
    red, nir, scl = read_and_align(item, aoi)
    valid = build_scl_mask(scl)
    ndvi = normalized_difference(nir, red)
    ndvi = np.where(valid, ndvi, np.nan)
    save_date_result(item.datetime, ndvi)

# 3. Agrupar por trimestre
quarterly = build_quarterly_medians(date_results)

# 4. Cubo temporal
cube = np.stack(quarterly, axis=0)

# 5. Comparar periodos equivalentes
delta = cube[-1] - cube[-5]

# 6. Clasificar cambio
green_up = delta > threshold
green_down = delta < -threshold

# 7. Calcular estadísticas
stats = calculate_change_statistics(delta, green_up, green_down)

# 8. Crear gráficos
plot_time_series(cube)
plot_change_map(delta)

# 9. Exportar GIS
save_geotiff("delta_ndvi.tif", delta)

# 10. Generar informe
build_report(stats, "informe_temporal.pdf")
```

Al terminar este capítulo hemos construido una cadena completa:

```text
SATÉLITE
   ↓
SERIE DE ESCENAS
   ↓
CONTROL DE CALIDAD
   ↓
NDVI POR FECHA
   ↓
COMPOSICIONES
   ↓
CUBO TEMPORAL
   ↓
CAMBIO Y TENDENCIA
   ↓
ESTADÍSTICAS + MAPAS
   ↓
INFORME PDF REPRODUCIBLE
```

Esto ya permite plantear aplicaciones reales en medio ambiente, agricultura, minería, restauración, seguimiento de obras, urbanismo y gestión territorial.

En el capítulo 4 cambiaremos de escala conceptual: dejaremos de preguntar únicamente qué ocurre en cada píxel y empezaremos a responder **qué significa para las personas y para las unidades territoriales**, combinando zonas verdes, H3, población GHSL y estadísticas zonales.

---

## Referencias técnicas

- Earth Search STAC v1: https://earth-search.aws.element84.com/v1
- Copernicus Sentinel-2: https://dataspace.copernicus.eu/explore-data/data-collections/sentinel-data/sentinel-2
- Rasterio: https://rasterio.readthedocs.io/
- Xarray: https://docs.xarray.dev/
- ReportLab: https://docs.reportlab.com/

---

[← Volver al índice del manual](../README.md)
