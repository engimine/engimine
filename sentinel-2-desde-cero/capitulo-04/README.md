# Capítulo 4 - Espacio verde por habitante: Sentinel-2 + barrios + H3 + GHSL

Ahora dejamos de mirar solo píxeles y construimos indicadores territoriales interpretables por barrio, distrito o hexágono.

> ¿Cuánto espacio verde detectado por Sentinel-2 corresponde a cada habitante y cómo se distribuye?

```text
Sentinel-2 → NDVI → vegetación → estadísticas zonales
barrios / H3 + población GHSL → m² verdes/habitante
→ mapas + ranking + informe técnico
```

## 4.1 Definir «verde»

Una regla didáctica:

```python
green_mask = ndvi > 0.4
```

El umbral debe documentarse y validarse cuando el estudio lo requiera. Sentinel-2 no observa directamente «parques»: mide reflectancia. Conviene conservar también NDVI medio, mediano y porcentaje de superficie sobre el umbral.

Si la cuadrícula proyectada es de 10 × 10 m:

```python
pixel_area_m2 = 100
green_area_m2 = np.sum(green_mask) * pixel_area_m2
```

No calcules áreas suponiendo que grados de longitud/latitud son metros.

## 4.2 Barrios y polígonos

```python
import geopandas as gpd

neighborhoods = gpd.read_file("barrios.geojson")
neighborhoods = neighborhoods.to_crs(raster_crs)
neighborhoods["area_m2"] = neighborhoods.geometry.area
```

Con `rasterstats`:

```bash
pip install rasterstats
```

```python
from rasterstats import zonal_stats

ndvi_stats = zonal_stats(
    neighborhoods,
    "ndvi.tif",
    stats=["mean", "median", "min", "max", "count"],
)
```

Para una máscara raster `1=verde`, `0=no verde`, nodata=sin observación:

```python
green_stats = zonal_stats(
    neighborhoods,
    "green_mask.tif",
    stats=["sum", "count"],
)

neighborhoods["green_pixels"] = [x["sum"] for x in green_stats]
neighborhoods["valid_pixels"] = [x["count"] for x in green_stats]
neighborhoods["green_m2"] = neighborhoods["green_pixels"] * 100
neighborhoods["green_pct"] = (
    100 * neighborhoods["green_pixels"] / neighborhoods["valid_pixels"]
)
```

Hay que distinguir superficie total del polígono de superficie realmente observada sin nodata.

## 4.3 H3: unidades homogéneas

Los barrios tienen formas y tamaños distintos. H3 permite analizar el territorio con celdas hexagonales jerárquicas.

```bash
pip install h3
```

```python
import h3
resolution = 9
```

La API exacta depende de la versión instalada. El flujo es: llevar el AOI a WGS84, obtener las celdas que lo cubren, convertir cada celda a polígono y crear un GeoDataFrame.

No elijas la resolución solo por estética: debe ser coherente con Sentinel-2, la resolución de población y la pregunta del estudio.

## 4.4 Población GHSL

El Global Human Settlement Layer (GHSL), del Joint Research Centre de la Comisión Europea, proporciona productos globales de población en cuadrícula.

Documenta siempre producto, edición/versión, año, resolución, CRS y significado de las celdas. Si Sentinel-2 y población son de años distintos, indícalo.

```python
import rasterio

with rasterio.open("ghsl_population.tif") as src:
    population = src.read(1)
    pop_crs = src.crs
    pop_transform = src.transform
```

Si la celda contiene número de habitantes, agregamos con suma:

```python
pop_stats = zonal_stats(
    neighborhoods,
    "ghsl_population.tif",
    stats=["sum"],
)

neighborhoods["population"] = [x["sum"] for x in pop_stats]
```

Una celda puede atravesar fronteras de barrios. En estudios finos puede ser necesario repartir población proporcionalmente por área en lugar de asignar celdas completas.

## 4.5 Indicador: m² verdes por habitante

```python
neighborhoods["green_m2_per_capita"] = (
    neighborhoods["green_m2"] / neighborhoods["population"]
)

neighborhoods.loc[
    neighborhoods["population"] <= 0,
    "green_m2_per_capita"
] = np.nan
```

Este indicador facilita comparar territorios de tamaños y poblaciones diferentes.

**Cobertura verde no equivale a accesibilidad.** La vegetación puede ser privada, estar concentrada en un extremo o quedar separada por infraestructuras. Tampoco mide por sí sola calidad ecológica o social.

## 4.6 Ranking

```python
ranking = neighborhoods.sort_values(
    "green_m2_per_capita", ascending=False
)[[
    "name", "population", "green_m2",
    "green_pct", "green_m2_per_capita"
]]

ranking.to_csv("ranking_barrios.csv", index=False)
```

Añadimos densidad:

```python
neighborhoods["population_density_km2"] = (
    neighborhoods["population"] /
    (neighborhoods["area_m2"] / 1_000_000)
)
```

## 4.7 Repetir el análisis con H3

```python
hex_green = zonal_stats(hexagons, "green_mask.tif", stats=["sum", "count"])
hex_pop = zonal_stats(hexagons, "ghsl_population.tif", stats=["sum"])

hexagons["green_pixels"] = [x["sum"] for x in hex_green]
hexagons["green_m2"] = hexagons["green_pixels"] * 100
hexagons["population"] = [x["sum"] for x in hex_pop]
hexagons["green_pc"] = hexagons["green_m2"] / hexagons["population"]
```

Así podemos comparar la lectura administrativa con una malla espacial homogénea.

## 4.8 Mapas

```python
import matplotlib.pyplot as plt

fig, ax = plt.subplots(figsize=(10, 10))
neighborhoods.plot(
    column="green_m2_per_capita",
    legend=True,
    ax=ax,
    edgecolor="black",
    linewidth=0.4,
)
ax.set_title("Espacio verde detectado por habitante")
ax.axis("off")
plt.tight_layout()
plt.savefig("mapa_verde_per_capita.png", dpi=250)
plt.close()
```

Para H3:

```python
fig, ax = plt.subplots(figsize=(10, 10))
hexagons.plot(column="green_pc", legend=True, ax=ax, linewidth=0.1)
ax.set_title("Espacio verde por habitante - H3")
ax.axis("off")
plt.tight_layout()
plt.savefig("mapa_h3_verde.png", dpi=250)
plt.close()
```

## 4.9 Informe ambiental automático

Estructura recomendada:

```text
PORTADA
1. Resumen ejecutivo
2. Objeto y alcance
3. Ámbito de estudio
4. Fuentes: Sentinel-2, límites, GHSL, H3
5. Metodología: NDVI, umbral, estadística zonal, población
6. Resultados por barrio
7. Análisis H3
8. Desigualdades territoriales
9. Limitaciones
10. Conclusiones
11. Reproducibilidad
```

Resumen calculado:

```python
summary = {
    "population_total": float(neighborhoods["population"].sum()),
    "green_area_total_m2": float(neighborhoods["green_m2"].sum()),
}
summary["green_pc_city"] = (
    summary["green_area_total_m2"] / summary["population_total"]
)
summary["median_green_pc"] = float(
    neighborhoods["green_m2_per_capita"].median()
)
```

Texto automático basado en datos:

```python
executive_text = f"""
La zona analizada concentra una población estimada de
{summary['population_total']:,.0f} habitantes y una superficie clasificada
como vegetación de {summary['green_area_total_m2']/1_000_000:.2f} km².
El indicador agregado es de {summary['green_pc_city']:.2f} m² de superficie
verde detectada por habitante.

Las diferencias describen cobertura vegetal estimada y no constituyen por sí
solas una medida completa de accesibilidad, calidad o justicia ambiental.
"""
```

### PDF con ReportLab

```python
from reportlab.lib.pagesizes import A4
from reportlab.lib.styles import getSampleStyleSheet
from reportlab.lib.units import cm
from reportlab.platypus import SimpleDocTemplate, Paragraph, Spacer, Image, Table, TableStyle, PageBreak
from reportlab.lib import colors


def build_green_report(summary, ranking, output="informe_verde.pdf"):
    doc = SimpleDocTemplate(
        output, pagesize=A4,
        leftMargin=2*cm, rightMargin=2*cm,
        topMargin=2*cm, bottomMargin=2*cm,
    )
    styles = getSampleStyleSheet()
    story = [
        Paragraph("Indicadores de espacio verde por habitante", styles["Title"]),
        Spacer(1, 0.5*cm),
        Paragraph("1. Resumen ejecutivo", styles["Heading1"]),
        Paragraph(
            f"Población estimada: {summary['population_total']:,.0f}. "
            f"Superficie verde: {summary['green_area_total_m2']/1_000_000:.2f} km². "
            f"Indicador agregado: {summary['green_pc_city']:.2f} m²/habitante.",
            styles["BodyText"],
        ),
        Spacer(1, 0.5*cm),
        Image("mapa_verde_per_capita.png", width=15*cm, height=15*cm),
        PageBreak(),
        Paragraph("2. Ranking territorial", styles["Heading1"]),
    ]

    rows = [["Unidad", "Población", "Verde m²", "% verde", "m²/hab"]]
    for _, r in ranking.head(15).iterrows():
        rows.append([
            str(r["name"]), f"{r['population']:.0f}",
            f"{r['green_m2']:.0f}", f"{r['green_pct']:.1f}",
            f"{r['green_m2_per_capita']:.2f}",
        ])

    table = Table(rows, repeatRows=1)
    table.setStyle(TableStyle([
        ("BACKGROUND", (0,0), (-1,0), colors.lightgrey),
        ("FONTNAME", (0,0), (-1,0), "Helvetica-Bold"),
        ("GRID", (0,0), (-1,-1), 0.4, colors.grey),
        ("FONTSIZE", (0,0), (-1,-1), 8),
    ]))
    story += [table, PageBreak(), Paragraph("3. Análisis H3", styles["Heading1"]),
              Image("mapa_h3_verde.png", width=15*cm, height=15*cm),
              Paragraph("4. Limitaciones", styles["Heading1"]),
              Paragraph("El indicador depende del umbral NDVI, fecha y calidad de imagen, resolución, producto de población y método de agregación. Cobertura vegetal no equivale necesariamente a espacio verde público accesible.", styles["BodyText"])]
    doc.build(story)
```

## 4.10 Productos de salida

```text
outputs/
├── green_mask.tif
├── ndvi.tif
├── barrios_indicadores.geojson
├── h3_indicadores.geojson
├── ranking_barrios.csv
├── estadisticas_resumen.json
├── mapa_verde_per_capita.png
├── mapa_h3_verde.png
└── informe_verde.pdf
```

```python
neighborhoods.to_file("barrios_indicadores.geojson", driver="GeoJSON")
hexagons.to_file("h3_indicadores.geojson", driver="GeoJSON")
```

## 4.11 Checklist de calidad

- [ ] Fecha/composición Sentinel-2 documentada.
- [ ] Nubes y sombras tratadas.
- [ ] Umbral NDVI documentado y validado cuando proceda.
- [ ] CRS adecuado para superficies.
- [ ] Fuente de límites documentada.
- [ ] Producto GHSL, año, resolución y versión documentados.
- [ ] Tratamiento de celdas de población fronterizas explicado.
- [ ] Nodata excluido correctamente.
- [ ] Cobertura y accesibilidad diferenciadas.
- [ ] Ranking acompañado de limitaciones.
- [ ] Datos intermedios y parámetros conservados.

## 4.12 Pipeline final

```python
ndvi = build_ndvi_composite(aoi, dates)
green_mask = ndvi > GREEN_THRESHOLD
save_green_mask(green_mask)

neighborhoods = load_neighborhoods().to_crs(raster_crs)
neighborhoods = add_green_statistics(neighborhoods, "green_mask.tif")
neighborhoods = add_population(neighborhoods, "ghsl_population.tif")
neighborhoods["green_m2_per_capita"] = neighborhoods["green_m2"] / neighborhoods["population"]

hexagons = build_h3_grid(aoi)
hexagons = add_green_statistics(hexagons, "green_mask.tif")
hexagons = add_population(hexagons, "ghsl_population.tif")

export_outputs(neighborhoods, hexagons)
plot_neighborhood_map(neighborhoods)
plot_h3_map(hexagons)
build_green_report(summary, ranking)
```

Hemos pasado de:

```text
PÍXEL → VEGETACIÓN → BARRIO/H3 → POBLACIÓN
→ m² VERDES/HABITANTE → MAPA → RANKING → INFORME
```

En el capítulo 5 cerraremos el recorrido con clasificación de cobertura del suelo mediante Machine Learning: etiquetas de OpenStreetMap, KNN, Random Forest, evaluación y una lectura honesta de los límites de trabajar a 10 m en una ciudad compleja.

## Referencias

- Copernicus Sentinel-2: https://dataspace.copernicus.eu/explore-data/data-collections/sentinel-data/sentinel-2
- GHSL: https://human-settlement.emergency.copernicus.eu/
- H3: https://h3geo.org/
- GeoPandas: https://geopandas.org/
- Rasterstats: https://pythonhosted.org/rasterstats/
- ReportLab: https://docs.reportlab.com/

[← Volver al índice](../README.md)
