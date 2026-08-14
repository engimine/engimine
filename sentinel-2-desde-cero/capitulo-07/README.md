# Capítulo 7 - Seguimiento ambiental de una cantera con Sentinel-2

Este capítulo aplica todo el manual a una **cantera real situada en el término municipal de Tarragona**, pero se presenta de forma deliberadamente anónima: no se publican nombres de empresas, titulares ni propietarios.

El objetivo es construir un sistema de seguimiento ambiental capaz de responder preguntas concretas:

- ¿Aumenta o disminuye la superficie sin vegetación?
- ¿Se detectan zonas con recuperación vegetal?
- ¿Cambian las balsas o superficies húmedas?
- ¿Qué sectores muestran mayor variación espectral?
- ¿Cuántas hectáreas cambian entre dos periodos?
- ¿Podemos generar automáticamente un informe técnico con mapas y resultados?

El flujo será:

```text
AOI cantera
   ↓
Sentinel-2 L2A
   ↓
control de nubes SCL
   ↓
RGB + NDVI + NDWI + NDBI
   ↓
composiciones temporales
   ↓
ΔNDVI + cambio de suelo desnudo
   ↓
hectáreas + porcentajes + series
   ↓
mapas + tablas
   ↓
informe de seguimiento ambiental
```

> **Criterio de rigor:** este capítulo no inventa cifras. El código genera automáticamente la tabla de resultados con los valores medidos sobre el AOI que utilice el lector. Las interpretaciones se formulan únicamente a partir de esos resultados.

---

## 7.1. Ámbito de estudio

El caso práctico corresponde a un área extractiva del sector nordeste de Tarragona, en un entorno próximo a la A-7 y a zonas de vegetación mediterránea y tejido urbano disperso.

Para reproducir el ejercicio de forma abierta puede utilizarse una ventana amplia de trabajo:

```python
BBOX = [1.255, 41.115, 1.305, 41.150]
```

Para un estudio técnico real debe sustituirse por el **polígono exacto de la explotación**:

```text
input/aoi_cantera.geojson
```

Una fuente pública describe actividad histórica de cantera en esta zona y señala la presencia de diferentes unidades extractivas, balsas y áreas posteriormente abandonadas o transformadas. Esto hace especialmente interesante el análisis multitemporal.

---

# Parte A - Preparar el proyecto

## 7.2. Estructura de carpetas

```text
cantera_tarragona/
├── input/
│   └── aoi_cantera.geojson
├── src/
│   └── seguimiento_cantera.py
└── outputs/
    ├── rasters/
    ├── maps/
    ├── tables/
    └── report/
```

## 7.3. Configuración

```python
CONFIG = {
    "bbox": [1.255, 41.115, 1.305, 41.150],
    "period_1": ["2023-05-01", "2023-09-30"],
    "period_2": ["2025-05-01", "2025-09-30"],
    "cloud_max": 25,
    "ndvi_green": 0.35,
    "ndvi_bare_max": 0.20,
    "change_threshold": 0.10,
    "output_dir": "outputs",
}
```

Comparar periodos estacionales equivalentes reduce parte del sesgo causado por la fenología.

---

# Parte B - Buscar Sentinel-2

## 7.4. Consulta STAC

```python
import pystac_client

catalog = pystac_client.Client.open(
    "https://earth-search.aws.element84.com/v1"
)


def search_period(start, end, bbox, cloud_max=25):
    search = catalog.search(
        collections=["sentinel-2-l2a"],
        bbox=bbox,
        datetime=f"{start}/{end}",
        query={"eo:cloud_cover": {"lt": cloud_max}},
    )
    return sorted(list(search.items()), key=lambda x: x.datetime)
```

```python
items_1 = search_period(*CONFIG["period_1"], CONFIG["bbox"], CONFIG["cloud_max"])
items_2 = search_period(*CONFIG["period_2"], CONFIG["bbox"], CONFIG["cloud_max"])
```

---

# Parte C - Productos espectrales

## 7.5. Índices

Usaremos:

```text
NDVI = (B08 - B04) / (B08 + B04)
NDWI = (B03 - B08) / (B03 + B08)
NDBI = (B11 - B08) / (B11 + B08)
```

En una cantera son especialmente útiles porque permiten separar tendencias asociadas a:

```text
vegetación
agua / humedad
superficies minerales o construidas
```

Ninguno de ellos debe interpretarse aisladamente como una clase jurídica o ambiental.

---

## 7.6. Máscara SCL

```python
BAD_SCL = [0, 1, 3, 8, 9, 10, 11]
valid = ~np.isin(scl_10m, BAD_SCL)
```

Todos los indicadores se calculan únicamente sobre píxeles válidos.

---

# Parte D - Composición por periodo

## 7.7. Mediana multiescena

```python
def median_composite(arrays):
    stack = np.stack(arrays, axis=0)
    return np.nanmedian(stack, axis=0).astype("float32")
```

Generamos:

```text
NDVI periodo 1
NDVI periodo 2
NDWI periodo 1
NDWI periodo 2
NDBI periodo 1
NDBI periodo 2
```

La composición mediana evita depender de una única pasada del satélite.

---

# Parte E - Superficie vegetal y suelo desnudo

## 7.8. Vegetación

```python
green_1 = ndvi_1 >= CONFIG["ndvi_green"]
green_2 = ndvi_2 >= CONFIG["ndvi_green"]
```

## 7.9. Superficie con baja cobertura vegetal

Una regla exploratoria:

```python
bare_1 = ndvi_1 <= CONFIG["ndvi_bare_max"]
bare_2 = ndvi_2 <= CONFIG["ndvi_bare_max"]
```

En una cantera `NDVI bajo` no significa automáticamente frente de explotación. Puede incluir:

- pistas;
- explanadas;
- roca;
- suelo seco;
- edificaciones;
- sombras;
- otras superficies sin vegetación.

Por eso combinaremos NDVI con NDBI, RGB y revisión visual.

---

# Parte F - Cambio temporal

## 7.10. ΔNDVI

```python
delta_ndvi = ndvi_2 - ndvi_1
```

```python
revegetation = delta_ndvi > CONFIG["change_threshold"]
vegetation_loss = delta_ndvi < -CONFIG["change_threshold"]
```

## 7.11. Expansión de superficie sin vegetación

```python
bare_gain = bare_2 & (~bare_1)
bare_recovery = bare_1 & (~bare_2)
```

Esto nos permite diferenciar:

```text
zona que pasa a baja cobertura vegetal
zona que deja de presentar baja cobertura vegetal
```

---

# Parte G - Convertir píxeles a hectáreas

## 7.12. Área

En una cuadrícula de 10 m:

```python
PIXEL_M2 = 100
PIXEL_HA = PIXEL_M2 / 10_000
```

```python
def area_ha(mask):
    return float(np.sum(mask) * PIXEL_HA)
```

Calculamos:

```python
results = {
    "green_ha_period_1": area_ha(green_1),
    "green_ha_period_2": area_ha(green_2),
    "bare_ha_period_1": area_ha(bare_1),
    "bare_ha_period_2": area_ha(bare_2),
    "revegetation_ha": area_ha(revegetation),
    "vegetation_loss_ha": area_ha(vegetation_loss),
    "bare_gain_ha": area_ha(bare_gain),
    "bare_recovery_ha": area_ha(bare_recovery),
    "mean_ndvi_period_1": float(np.nanmean(ndvi_1)),
    "mean_ndvi_period_2": float(np.nanmean(ndvi_2)),
    "mean_delta_ndvi": float(np.nanmean(delta_ndvi)),
}
```

---

# Parte H - Resultados: que acompañen siempre al código

## 7.13. Tabla automática de resultados

Este capítulo introduce una regla para el resto del manual:

> **Todo análisis debe terminar con un bloque de resultados legible por una persona que no programe.**

```python
import pandas as pd

result_table = pd.DataFrame([
    ["Vegetación periodo 1", results["green_ha_period_1"], "ha"],
    ["Vegetación periodo 2", results["green_ha_period_2"], "ha"],
    ["Baja cobertura vegetal periodo 1", results["bare_ha_period_1"], "ha"],
    ["Baja cobertura vegetal periodo 2", results["bare_ha_period_2"], "ha"],
    ["Revegetación espectral", results["revegetation_ha"], "ha"],
    ["Pérdida espectral de vegetación", results["vegetation_loss_ha"], "ha"],
    ["Nueva superficie de baja cobertura", results["bare_gain_ha"], "ha"],
    ["Recuperación de baja cobertura", results["bare_recovery_ha"], "ha"],
    ["NDVI medio periodo 1", results["mean_ndvi_period_1"], "NDVI"],
    ["NDVI medio periodo 2", results["mean_ndvi_period_2"], "NDVI"],
    ["Cambio medio NDVI", results["mean_delta_ndvi"], "NDVI"],
], columns=["Indicador", "Resultado", "Unidad"])

result_table.to_csv(
    "outputs/tables/resultados_cantera.csv",
    index=False,
)
```

### Salida esperada del programa

```text
Indicador                              Resultado      Unidad
Vegetación periodo 1                   [medido]        ha
Vegetación periodo 2                   [medido]        ha
Baja cobertura vegetal periodo 1       [medido]        ha
Baja cobertura vegetal periodo 2       [medido]        ha
Revegetación espectral                  [medido]        ha
Pérdida espectral de vegetación         [medido]        ha
Nueva superficie de baja cobertura      [medido]        ha
Recuperación de baja cobertura          [medido]        ha
NDVI medio periodo 1                    [medido]       NDVI
NDVI medio periodo 2                    [medido]       NDVI
Cambio medio NDVI                       [medido]       NDVI
```

Los campos `[medido]` se sustituyen automáticamente por los valores obtenidos al ejecutar el análisis sobre el AOI.

---

# Parte I - Interpretar los resultados

## 7.14. Generar frases desde datos

```python
if results["revegetation_ha"] > results["vegetation_loss_ha"]:
    vegetation_message = (
        "El balance espectral muestra una superficie con incremento de NDVI "
        "superior a la superficie con disminución."
    )
else:
    vegetation_message = (
        "La superficie con disminución de NDVI es superior a la superficie "
        "con incremento durante los periodos comparados."
    )
```

Para suelo desnudo:

```python
bare_delta = (
    results["bare_ha_period_2"]
    - results["bare_ha_period_1"]
)
```

```python
if bare_delta > 0:
    bare_message = (
        f"La superficie de baja cobertura vegetal aumenta "
        f"aproximadamente {bare_delta:.2f} ha entre ambos periodos."
    )
else:
    bare_message = (
        f"La superficie de baja cobertura vegetal disminuye "
        f"aproximadamente {abs(bare_delta):.2f} ha entre ambos periodos."
    )
```

Estas frases describen **lo observado**, no atribuyen la causa.

---

# Parte J - Mapas de resultados

## 7.15. Mapa RGB antes/después

Guardar:

```text
rgb_periodo_1.png
rgb_periodo_2.png
```

La inspección visual es imprescindible para interpretar cambios espectrales.

## 7.16. Mapa ΔNDVI

```python
plt.figure(figsize=(10, 8))
plt.imshow(delta_ndvi, vmin=-0.5, vmax=0.5, cmap="RdYlGn")
plt.colorbar(label="ΔNDVI")
plt.title("Cambio de vegetación - cantera objeto de estudio")
plt.axis("off")
plt.tight_layout()
plt.savefig("outputs/maps/delta_ndvi.png", dpi=250)
plt.close()
```

## 7.17. Mapa de zonas recuperadas y pérdidas

```python
change_classes = np.zeros(delta_ndvi.shape, dtype="int8")
change_classes[vegetation_loss] = -1
change_classes[revegetation] = 1
```

Este raster puede guardarse como GeoTIFF y abrirse en QGIS.

---

# Parte K - Seguimiento de balsas y humedad

## 7.18. NDWI

```python
water_like_1 = ndwi_1 > 0.10
water_like_2 = ndwi_2 > 0.10
```

En una cantera este resultado debe verificarse cuidadosamente porque sombras profundas y materiales oscuros pueden producir confusiones.

El informe debe utilizar expresiones como:

```text
"superficie con respuesta espectral compatible con agua"
```

antes que afirmar que existe una balsa sin comprobación visual o de campo.

---

# Parte L - Serie temporal

## 7.19. NDVI medio por fecha

```python
series = []

for date, ndvi in ndvi_by_date.items():
    series.append({
        "date": date,
        "mean_ndvi": float(np.nanmean(ndvi)),
        "median_ndvi": float(np.nanmedian(ndvi)),
    })

series_df = pd.DataFrame(series).sort_values("date")
```

Esto permite diferenciar una tendencia de un cambio puntual.

---

# Parte M - Informe ambiental automático

## 7.20. Estructura

```text
PORTADA
Seguimiento ambiental mediante Sentinel-2
Cantera objeto de estudio - Tarragona

1. RESUMEN EJECUTIVO
2. OBJETO
3. ÁMBITO DE ESTUDIO
4. DATOS UTILIZADOS
5. METODOLOGÍA
6. RESULTADOS
   6.1 Estado inicial
   6.2 Estado final
   6.3 Variación de NDVI
   6.4 Superficie con baja cobertura vegetal
   6.5 Zonas con recuperación espectral
   6.6 Superficies compatibles con agua
7. SERIE TEMPORAL
8. INTERPRETACIÓN TÉCNICA
9. LIMITACIONES
10. CONCLUSIONES
11. ARCHIVOS GENERADOS
```

---

## 7.21. Resumen ejecutivo automático

```python
executive_summary = f"""
El análisis compara dos periodos equivalentes mediante composiciones de
Sentinel-2 L2A y control de calidad SCL.

La superficie con incremento significativo de NDVI es de
{results['revegetation_ha']:.2f} ha, mientras que la superficie con
disminución significativa alcanza {results['vegetation_loss_ha']:.2f} ha.

La superficie clasificada mediante el criterio exploratorio de baja cobertura
vegetal pasa de {results['bare_ha_period_1']:.2f} ha a
{results['bare_ha_period_2']:.2f} ha.

Estos resultados representan cambios espectrales detectados por Sentinel-2.
No permiten atribuir por sí solos el origen del cambio ni sustituyen la
inspección de campo o la documentación de explotación y restauración.
"""
```

---

# Parte N - Qué puede detectar Sentinel-2 y qué no

## 7.22. Sí puede ayudar a observar

```text
cambios amplios de cobertura vegetal
expansión o reducción de superficies desnudas
cambios en grandes balsas
patrones temporales
sectores con variación espectral persistente
recuperación vegetal a escala de decenas de metros
```

## 7.23. No puede demostrar por sí solo

```text
cumplimiento legal de un proyecto de restauración
volumen extraído
altura exacta de bancos
estabilidad geotécnica
calidad química del agua
causa concreta de un cambio
límites pequeños inferiores a la resolución efectiva
```

Para esas cuestiones debemos integrar topografía, dron, LiDAR, ensayos, cartografía de proyecto o inspección de campo.

---

# Parte O - Resultados que deben acompañar siempre al informe

## 7.24. Paquete mínimo

```text
outputs/
├── rasters/
│   ├── ndvi_periodo_1.tif
│   ├── ndvi_periodo_2.tif
│   ├── delta_ndvi.tif
│   ├── bare_change.tif
│   └── water_change.tif
├── maps/
│   ├── rgb_periodo_1.png
│   ├── rgb_periodo_2.png
│   ├── delta_ndvi.png
│   ├── bare_change.png
│   └── time_series.png
├── tables/
│   ├── inventario_escenas.csv
│   ├── resultados_cantera.csv
│   └── serie_temporal.csv
└── report/
    └── informe_seguimiento_ambiental.pdf
```

---

# Parte P - Checklist técnico

- [ ] AOI exacto definido.
- [ ] Fechas comparables estacionalmente.
- [ ] Inventario de escenas guardado.
- [ ] SCL aplicado.
- [ ] Número de observaciones válidas revisado.
- [ ] NDVI, NDWI y NDBI guardados.
- [ ] Umbrales documentados.
- [ ] Áreas calculadas en CRS proyectado.
- [ ] Resultados expresados en ha y %.
- [ ] RGB antes/después revisado visualmente.
- [ ] Serie temporal incluida.
- [ ] Resultados y mapas aparecen juntos en el informe.
- [ ] Causalidad no inferida solo desde Sentinel-2.
- [ ] Limitaciones de 10 m explicadas.

---

# Parte Q - Función maestra

```python
def run_quarry_monitoring(config, aoi_path):
    aoi = load_aoi(aoi_path)

    period_1 = build_period_composite(
        aoi,
        *config["period_1"],
        config,
    )

    period_2 = build_period_composite(
        aoi,
        *config["period_2"],
        config,
    )

    indices_1 = calculate_indices(period_1)
    indices_2 = calculate_indices(period_2)

    change = calculate_quarry_change(
        indices_1,
        indices_2,
        config,
    )

    results = calculate_quarry_statistics(
        indices_1,
        indices_2,
        change,
        config,
    )

    export_rasters(indices_1, indices_2, change, config)
    export_result_tables(results, config)
    create_result_maps(indices_1, indices_2, change, config)
    build_environmental_report(results, config)

    return results
```

Uso:

```python
results = run_quarry_monitoring(
    CONFIG,
    "input/aoi_cantera.geojson",
)

print(pd.Series(results))
```

---

# Conclusión

Este capítulo convierte Sentinel-2 en una herramienta de **seguimiento**, no simplemente de visualización.

El producto final no es un mapa aislado, sino una cadena trazable:

```text
DATOS ABIERTOS
    ↓
MÉTODO REPRODUCIBLE
    ↓
MAPAS
    ↓
RESULTADOS NUMÉRICOS
    ↓
INTERPRETACIÓN PRUDENTE
    ↓
INFORME TÉCNICO
```

A partir de aquí podemos ampliar el manual con sensores de mayor resolución, Sentinel-1 radar, LiDAR, drones y modelos digitales del terreno para construir un sistema de vigilancia ambiental mucho más completo.

---

## Referencias

- Earth Search STAC v1: https://earth-search.aws.element84.com/v1
- Copernicus Sentinel-2: https://dataspace.copernicus.eu/explore-data/data-collections/sentinel-data/sentinel-2
- ICGC Geocoder: https://www.icgc.cat/es/Herramientas-y-visores/Herramientas/Geocodificador-ICGC
- Información pública sobre antiguas áreas de cantera en el nordeste de Tarragona: https://infominer.minercat.com/2025/02/09/la-pedrera-de-la-budellera-tarragona/

---

[← Volver al índice del manual](../README.md)
