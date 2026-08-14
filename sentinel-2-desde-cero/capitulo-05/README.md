# Capítulo 5 - Clasificación del territorio con Machine Learning

Este capítulo cierra el recorrido del manual con un objetivo ambicioso: entrenar un modelo que asigne una clase de cobertura del suelo a cada píxel Sentinel-2.

No vamos a tratar el modelo como una caja mágica. El objetivo es entender todo el flujo:

```text
Sentinel-2 → bandas + índices → etiquetas OSM
           ↓
     dataset de entrenamiento
           ↓
   KNN → Random Forest
           ↓
validación → matriz de confusión
           ↓
 mapa de clases + incertidumbre
           ↓
 informe técnico automático
```

La idea central es sencilla:

> Cada píxel se convierte en una fila de datos y cada clase del terreno se convierte en una etiqueta.

---

## 5.1. Qué vamos a clasificar

Para un ejemplo urbano como Manhattan podemos trabajar con clases didácticas:

```text
0 = agua
1 = vegetación
2 = construido
3 = suelo / superficie abierta
```

En un proyecto real las clases dependen del objetivo. En minería, por ejemplo, podríamos distinguir:

```text
vegetación
suelo desnudo
escombrera
lámina de agua
infraestructura
frente de explotación
```

La calidad del resultado dependerá tanto de las etiquetas como del algoritmo.

---

# Parte A - Construir las variables de entrada

## 5.2. Feature vector por píxel

Reutilizamos la estructura del capítulo 2:

```text
[B02, B03, B04, B08, B11, NDVI, NDWI, NDBI]
```

En Python:

```python
features = np.stack([
    blue,
    green,
    red,
    nir,
    swir1_10m,
    ndvi,
    ndwi,
    ndbi,
], axis=-1)
```

Si la imagen mide 1000 x 800 píxeles:

```text
features.shape = (1000, 800, 8)
```

Y para scikit-learn:

```python
X_all = features.reshape(-1, features.shape[-1])
```

---

## 5.3. Eliminar píxeles inválidos

```python
valid = np.all(np.isfinite(X_all), axis=1)
X_valid = X_all[valid]
```

Es importante conservar la máscara `valid` para reconstruir después el mapa completo.

---

# Parte B - Crear etiquetas con OpenStreetMap

## 5.4. Por qué OSM

OpenStreetMap puede proporcionar geometrías útiles para crear etiquetas iniciales:

- parques y zonas verdes;
- masas de agua;
- edificios;
- superficies urbanizadas;
- usos del suelo.

Estas etiquetas no son una verdad absoluta. OSM puede estar incompleto, desactualizado o tener geometrías simplificadas. Debemos considerarlo una fuente de entrenamiento, no una referencia perfecta.

---

## 5.5. Descargar geometrías con OSMnx

```bash
pip install osmnx
```

Ejemplo conceptual:

```python
import osmnx as ox

place = "Manhattan, New York, USA"

parks = ox.features_from_place(
    place,
    tags={"leisure": "park"},
)

water = ox.features_from_place(
    place,
    tags={"natural": "water"},
)

buildings = ox.features_from_place(
    place,
    tags={"building": True},
)
```

Siempre conviene revisar visualmente las geometrías recuperadas.

---

## 5.6. Llevar OSM al CRS de Sentinel-2

```python
parks = parks.to_crs(raster_crs)
water = water.to_crs(raster_crs)
buildings = buildings.to_crs(raster_crs)
```

---

# Parte C - Rasterizar las etiquetas

## 5.7. Crear un raster de clases

```python
from rasterio.features import rasterize

label_raster = np.full(
    red.shape,
    fill_value=-1,
    dtype="int16",
)
```

Rasterizamos cada grupo:

```python
water_mask = rasterize(
    [(geom, 1) for geom in water.geometry if geom is not None],
    out_shape=red.shape,
    transform=red_profile["transform"],
    fill=0,
    dtype="uint8",
)

park_mask = rasterize(
    [(geom, 1) for geom in parks.geometry if geom is not None],
    out_shape=red.shape,
    transform=red_profile["transform"],
    fill=0,
    dtype="uint8",
)

building_mask = rasterize(
    [(geom, 1) for geom in buildings.geometry if geom is not None],
    out_shape=red.shape,
    transform=red_profile["transform"],
    fill=0,
    dtype="uint8",
)
```

Asignamos clases:

```python
label_raster[water_mask == 1] = 0
label_raster[park_mask == 1] = 1
label_raster[building_mask == 1] = 2
```

Podemos reservar `3` para suelo abierto a partir de otra fuente o de polígonos de entrenamiento manuales.

---

## 5.8. Resolver conflictos de etiquetas

Un píxel puede coincidir con varias geometrías OSM.

Por ejemplo:

```text
parque + edificio
agua + muelle
zona verde + infraestructura
```

Debemos definir una prioridad o eliminar píxeles ambiguos.

Una estrategia conservadora es marcar como inválidos los solapes.

```python
overlap = (
    water_mask.astype(int)
    + park_mask.astype(int)
    + building_mask.astype(int)
) > 1

label_raster[overlap] = -1
```

---

# Parte D - Crear X e y

## 5.9. Extraer solo píxeles etiquetados

```python
y_all = label_raster.ravel()

train_mask = (
    valid
    & (y_all >= 0)
)

X = X_all[train_mask]
y = y_all[train_mask]

print(X.shape)
print(y.shape)
```

---

## 5.10. Revisar equilibrio de clases

```python
import pandas as pd

class_counts = pd.Series(y).value_counts().sort_index()
print(class_counts)
```

Si tenemos 200.000 píxeles construidos y 3.000 de agua, una accuracy global puede resultar engañosa.

---

# Parte E - El problema de la fuga espacial

## 5.11. Train/test aleatorio puede ser demasiado optimista

Los píxeles vecinos se parecen muchísimo.

Si dividimos al azar:

```python
from sklearn.model_selection import train_test_split

X_train, X_test, y_train, y_test = train_test_split(
    X,
    y,
    test_size=0.25,
    random_state=42,
    stratify=y,
)
```

podemos acabar entrenando con un píxel y evaluando con el píxel de al lado.

Eso puede inflar artificialmente el rendimiento.

Para teledetección es mejor, cuando sea posible, separar espacialmente las zonas de entrenamiento y validación.

---

## 5.12. Validación por bloques

Una estrategia sencilla consiste en dividir el raster en bloques espaciales y reservar bloques completos para test.

Conceptualmente:

```text
Norte → test
Centro → train
Sur → train
```

O utilizar grupos:

```python
from sklearn.model_selection import GroupShuffleSplit
```

Cada muestra debe tener asociado un `group_id` espacial.

---

# Parte F - Primer modelo: KNN

## 5.13. Por qué empezar con KNN

KNN es sencillo y permite establecer una línea base.

```python
from sklearn.preprocessing import StandardScaler
from sklearn.pipeline import Pipeline
from sklearn.neighbors import KNeighborsClassifier

knn = Pipeline([
    ("scale", StandardScaler()),
    ("model", KNeighborsClassifier(n_neighbors=7)),
])

knn.fit(X_train, y_train)
```

Predicción:

```python
y_pred_knn = knn.predict(X_test)
```

---

## 5.14. Evaluar KNN

```python
from sklearn.metrics import classification_report

print(classification_report(
    y_test,
    y_pred_knn,
    digits=3,
))
```

No debemos mirar solo `accuracy`.

Nos interesan:

```text
precision
recall
f1-score
support
```

por cada clase.

---

# Parte G - Random Forest

## 5.15. Entrenar el modelo

```python
from sklearn.ensemble import RandomForestClassifier

rf = RandomForestClassifier(
    n_estimators=300,
    max_depth=None,
    min_samples_leaf=2,
    class_weight="balanced",
    random_state=42,
    n_jobs=-1,
)

rf.fit(X_train, y_train)
```

Predicción:

```python
y_pred_rf = rf.predict(X_test)
```

---

## 5.16. Comparar KNN y Random Forest

```python
from sklearn.metrics import f1_score

f1_knn = f1_score(
    y_test,
    y_pred_knn,
    average="macro",
)

f1_rf = f1_score(
    y_test,
    y_pred_rf,
    average="macro",
)

print("Macro F1 KNN:", f1_knn)
print("Macro F1 RF:", f1_rf)
```

`macro F1` da el mismo peso a cada clase y es útil cuando hay desequilibrio.

---

# Parte H - Matriz de confusión

## 5.17. Ver exactamente dónde falla

```python
from sklearn.metrics import confusion_matrix

cm = confusion_matrix(y_test, y_pred_rf)
print(cm)
```

Visualización:

```python
from sklearn.metrics import ConfusionMatrixDisplay
import matplotlib.pyplot as plt

labels = ["agua", "vegetación", "construido", "suelo"]

ConfusionMatrixDisplay(
    confusion_matrix=cm,
    display_labels=labels,
).plot(values_format="d")

plt.title("Matriz de confusión - Random Forest")
plt.tight_layout()
plt.savefig("matriz_confusion.png", dpi=220)
plt.close()
```

La matriz nos ayuda a responder preguntas concretas:

```text
¿el agua se confunde con sombra?
¿el construido se confunde con suelo desnudo?
¿la vegetación urbana se detecta bien?
```

---

# Parte I - Importancia de variables

## 5.18. Qué utiliza el bosque

```python
feature_names = [
    "B02", "B03", "B04", "B08",
    "B11", "NDVI", "NDWI", "NDBI",
]

importance = pd.Series(
    rf.feature_importances_,
    index=feature_names,
).sort_values(ascending=False)

print(importance)
```

Gráfico:

```python
ax = importance.sort_values().plot.barh(figsize=(8, 5))
ax.set_xlabel("Importancia")
ax.set_title("Importancia de variables")
plt.tight_layout()
plt.savefig("feature_importance.png", dpi=220)
plt.close()
```

La importancia del Random Forest es descriptiva del modelo, no una demostración causal.

---

# Parte J - Clasificar toda la escena

## 5.19. Predicción por píxel

```python
prediction_flat = np.full(
    X_all.shape[0],
    -1,
    dtype="int16",
)

prediction_flat[valid] = rf.predict(X_all[valid])

classification = prediction_flat.reshape(red.shape)
```

---

## 5.20. Guardar el mapa

```python
profile = red_profile.copy()
profile.update(
    dtype="int16",
    count=1,
    nodata=-1,
    compress="deflate",
)

with rasterio.open(
    "landcover_rf.tif",
    "w",
    **profile,
) as dst:
    dst.write(classification, 1)
```

---

# Parte K - Probabilidades e incertidumbre

## 5.21. No todas las predicciones son igual de seguras

Random Forest puede devolver probabilidades:

```python
proba = rf.predict_proba(X_all[valid])
confidence = np.max(proba, axis=1)
```

Reconstruimos el mapa:

```python
confidence_flat = np.full(
    X_all.shape[0],
    np.nan,
    dtype="float32",
)

confidence_flat[valid] = confidence
confidence_map = confidence_flat.reshape(red.shape)
```

Un píxel con:

```text
0.98
```

tiene más consenso entre árboles que uno con:

```text
0.39
```

Esto no equivale automáticamente a probabilidad calibrada, pero sí es una medida útil de confianza interna del modelo.

---

# Parte L - El fallo honesto a 10 m

## 5.22. Por qué una clasificación urbana puede fallar

Un píxel Sentinel-2 de 10 m cubre aproximadamente 100 m².

Dentro puede haber simultáneamente:

```text
árbol + acera + coche + fachada + sombra
```

Ese píxel es **mixto**.

OSM, en cambio, puede describir polígonos con límites mucho más precisos.

Cuando rasterizamos OSM a 10 m estamos imponiendo una etiqueta discreta sobre una señal que puede contener varias coberturas.

---

## 5.23. Ejemplo del problema

Supongamos un píxel con:

```text
45 % copa de árbol
35 % pavimento
20 % edificio
```

OSM puede etiquetarlo como parque.

Sentinel-2 observa la mezcla espectral completa.

El modelo no está necesariamente «equivocado» si su señal se parece también al construido.

---

## 5.24. Sombras urbanas

Los edificios altos generan sombras intensas.

Una sombra urbana puede parecer espectralmente más cercana al agua que al edificio que la produce.

Esto puede provocar:

```text
sombra → agua
```

si el dataset no contiene suficientes ejemplos.

---

## 5.25. Desfase temporal

OSM y Sentinel-2 no tienen necesariamente la misma fecha.

Un solar puede aparecer como edificio en OSM pero estar en obras en la imagen, o al revés.

Debemos documentar la fecha de ambos conjuntos.

---

## 5.26. Accuracy alta no significa mapa perfecto

Podemos obtener:

```text
accuracy = 0.91
```

y seguir teniendo un problema grave si la clase menos frecuente tiene recall muy bajo.

Ejemplo:

```text
agua recall = 0.98
vegetación recall = 0.94
construido recall = 0.95
suelo recall = 0.31
```

La cifra global oculta el fallo.

---

# Parte M - Métricas recomendadas

## 5.27. Resumen de evaluación

Como mínimo debemos informar:

```text
accuracy
macro F1
precision por clase
recall por clase
F1 por clase
número de muestras por clase
matriz de confusión
```

Código:

```python
from sklearn.metrics import (
    accuracy_score,
    f1_score,
    classification_report,
)

metrics = {
    "accuracy": accuracy_score(y_test, y_pred_rf),
    "macro_f1": f1_score(
        y_test,
        y_pred_rf,
        average="macro",
    ),
}

report = classification_report(
    y_test,
    y_pred_rf,
    target_names=labels,
    output_dict=True,
)
```

---

# Parte N - Guardar el modelo

## 5.28. Persistencia

```python
import joblib

joblib.dump(rf, "random_forest_landcover.joblib")
```

Cargar después:

```python
rf = joblib.load("random_forest_landcover.joblib")
```

Debemos guardar también:

```text
lista de features
orden de features
clases
fecha de entrenamiento
parámetros
versión de dependencias
```

---

# Parte O - Informe técnico automático

## 5.29. Estructura recomendada

```text
PORTADA
1. RESUMEN EJECUTIVO
2. OBJETO
3. ÁMBITO DE ESTUDIO
4. DATOS
   4.1 Sentinel-2
   4.2 OpenStreetMap
5. VARIABLES DE ENTRADA
6. CREACIÓN DE ETIQUETAS
7. DISEÑO DE VALIDACIÓN
8. MODELOS
   8.1 KNN
   8.2 Random Forest
9. RESULTADOS
   9.1 Accuracy y macro F1
   9.2 Métricas por clase
   9.3 Matriz de confusión
   9.4 Importancia de variables
   9.5 Mapa clasificado
   9.6 Confianza
10. ANÁLISIS DE ERRORES
11. LIMITACIONES
12. CONCLUSIONES
13. REPRODUCIBILIDAD
```

---

## 5.30. Crear un mapa PNG para el informe

```python
plt.figure(figsize=(10, 9))
plt.imshow(classification)
plt.title("Clasificación de cobertura - Random Forest")
plt.axis("off")
plt.tight_layout()
plt.savefig("mapa_clasificacion.png", dpi=250)
plt.close()
```

Mapa de confianza:

```python
plt.figure(figsize=(10, 9))
plt.imshow(confidence_map, vmin=0, vmax=1)
plt.colorbar(label="Confianza interna")
plt.title("Confianza de clasificación")
plt.axis("off")
plt.tight_layout()
plt.savefig("mapa_confianza.png", dpi=250)
plt.close()
```

---

## 5.31. Crear texto automático de resultados

```python
summary_text = f"""
El modelo Random Forest alcanzó una exactitud global de
{metrics['accuracy']:.3f} y un macro F1 de {metrics['macro_f1']:.3f}
sobre el conjunto de validación definido.

Las métricas deben interpretarse junto con la matriz de confusión y el
rendimiento por clase, especialmente debido al desequilibrio de muestras,
la mezcla espectral a 10 m y las diferencias temporales y geométricas entre
Sentinel-2 y las etiquetas derivadas de OpenStreetMap.
"""
```

---

## 5.32. PDF con ReportLab

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


def build_ml_report(metrics, output="informe_clasificacion.pdf"):
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
        "Clasificación de cobertura del suelo con Sentinel-2",
        styles["Title"],
    ))

    story.append(Spacer(1, 0.5*cm))
    story.append(Paragraph("1. Resumen ejecutivo", styles["Heading1"]))

    story.append(Paragraph(
        f"El modelo Random Forest obtuvo una exactitud global de "
        f"{metrics['accuracy']:.3f} y un macro F1 de "
        f"{metrics['macro_f1']:.3f}.",
        styles["BodyText"],
    ))

    story.append(PageBreak())
    story.append(Paragraph("2. Matriz de confusión", styles["Heading1"]))
    story.append(Image(
        "matriz_confusion.png",
        width=15*cm,
        height=12*cm,
    ))

    story.append(PageBreak())
    story.append(Paragraph("3. Importancia de variables", styles["Heading1"]))
    story.append(Image(
        "feature_importance.png",
        width=15*cm,
        height=9*cm,
    ))

    story.append(PageBreak())
    story.append(Paragraph("4. Mapa clasificado", styles["Heading1"]))
    story.append(Image(
        "mapa_clasificacion.png",
        width=15*cm,
        height=14*cm,
    ))

    story.append(PageBreak())
    story.append(Paragraph("5. Confianza", styles["Heading1"]))
    story.append(Image(
        "mapa_confianza.png",
        width=15*cm,
        height=14*cm,
    ))

    story.append(Paragraph("6. Limitaciones", styles["Heading1"]))
    story.append(Paragraph(
        "La resolución espacial de 10 m produce píxeles mixtos en entornos "
        "urbanos. Las etiquetas OSM pueden contener omisiones, errores o "
        "desfases temporales. Las métricas dependen del diseño de validación "
        "y una división aleatoria de píxeles puede sobreestimar el rendimiento "
        "por autocorrelación espacial.",
        styles["BodyText"],
    ))

    doc.build(story)
```

---

# Parte P - Productos finales

## 5.33. Carpeta recomendada

```text
outputs/
├── training_samples.csv
├── class_distribution.csv
├── random_forest_landcover.joblib
├── landcover_rf.tif
├── confidence.tif
├── matriz_confusion.png
├── feature_importance.png
├── mapa_clasificacion.png
├── mapa_confianza.png
├── metrics.json
└── informe_clasificacion.pdf
```

---

# Parte Q - Checklist profesional

## 5.34. Antes de afirmar que el modelo funciona

- [ ] Clases definidas explícitamente.
- [ ] Fuente y fecha de etiquetas documentadas.
- [ ] Sentinel-2 y OSM comparados temporalmente.
- [ ] Solapes de etiquetas tratados.
- [ ] Píxeles nodata y nubes excluidos.
- [ ] Distribución de clases revisada.
- [ ] Train/test espacial cuando sea posible.
- [ ] KNN usado como línea base.
- [ ] Random Forest evaluado con métricas por clase.
- [ ] Macro F1 calculado.
- [ ] Matriz de confusión inspeccionada.
- [ ] Mapa de confianza generado.
- [ ] Errores urbanos a 10 m discutidos.
- [ ] Modelo, parámetros y features guardados.
- [ ] Resultados reproducibles.

---

# Parte R - Pipeline final del libro

## 5.35. Todo junto

```python
# 1. Sentinel-2
bands = load_clean_sentinel_bands(aoi, dates)

# 2. Índices
indices = calculate_indices(bands)

# 3. Features
features = build_feature_cube(bands, indices)

# 4. Etiquetas OSM
labels = build_osm_training_labels(aoi)

# 5. Dataset
X, y, groups = build_training_dataset(features, labels)

# 6. Split espacial
X_train, X_test, y_train, y_test = spatial_split(X, y, groups)

# 7. Baseline
knn.fit(X_train, y_train)

# 8. Random Forest
rf.fit(X_train, y_train)

# 9. Validación
y_pred = rf.predict(X_test)
metrics = evaluate_model(y_test, y_pred)

# 10. Clasificación completa
classification = classify_scene(rf, features)

# 11. Confianza
confidence = build_confidence_map(rf, features)

# 12. Exportar
save_classification(classification)
save_confidence(confidence)

# 13. Informe
build_ml_report(metrics)
```

---

# Conclusión del manual

Hemos recorrido todo el proceso desde cero:

```text
RASTER
  ↓
STAC
  ↓
SENTINEL-2 REAL
  ↓
BANDAS Y REFLECTANCIA
  ↓
NDVI / NDWI / NDBI
  ↓
SERIES TEMPORALES
  ↓
CAMBIO
  ↓
POBLACIÓN Y H3
  ↓
INDICADORES TERRITORIALES
  ↓
MACHINE LEARNING
  ↓
MAPAS + INFORMES TÉCNICOS
```

La parte más importante no es haber entrenado un Random Forest.

Es entender qué entra en el modelo, qué sale, qué errores puede cometer y qué afirmaciones podemos defender con los datos.

Ese es el salto entre **usar una herramienta** y **construir un flujo geoespacial reproducible**.

---

## Referencias técnicas

- Sentinel-2 / Copernicus Data Space: https://dataspace.copernicus.eu/explore-data/data-collections/sentinel-data/sentinel-2
- OpenStreetMap: https://www.openstreetmap.org/
- OSMnx: https://osmnx.readthedocs.io/
- scikit-learn: https://scikit-learn.org/
- Rasterio: https://rasterio.readthedocs.io/
- ReportLab: https://docs.reportlab.com/

---

[← Volver al índice del manual](../README.md)
