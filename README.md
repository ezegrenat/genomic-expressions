# Clasificación de expresiones genómicas para pronóstico oncológico

## Descripción general

Clasificadores supervisado para predecir si una lesión pre-tumoral tiene buen o mal pronóstico, a partir de la expresión de 200 genes medidos con tecnología NGS. Comparamos seis familias de modelos con búsqueda de hiperparámetros, analizamos el trade-off sesgo-varianza de los mejores y evaluamos el modelo final sobre un conjunto held-out. El mejor modelo, que fue SVC con kernel RBF, alcanzó un AUC-ROC de 0.9386 en datos no vistos.

El dataset fue desarrollado por el [CONICET](https://www.conicet.gov.ar/) y el proyecto fue presentado para una competencia dentro de la [Facultad de Ciencias Exactas y Naturales](https://exactas.uba.ar/) de la Universidad de Buenos Aires.

---

## Objetivos del proyecto

- Diseñar una estrategia de partición del dataset que contemple el desbalanceo de clases y la independencia entre atributos y target
- Construir y evaluar árboles de decisión mediante K-fold cross-validation, explorando distintas combinaciones de profundidad y criterio de corte
- Comparar múltiples familias de clasificadores (Decision Tree, KNN, SVM, LDA, Naive Bayes y Random Forest) con búsqueda aleatoria de hiperparámetros
- Diagnosticar el trade-off entre sesgo y varianza de los mejores modelos mediante curvas de complejidad y de aprendizaje
- Seleccionar el mejor modelo y evaluar su performance sobre datos held-out no utilizados durante el desarrollo

---

## Dataset

| Dimensión | Detalle |
|-----------|---------|
| Instancias | 500 muestras de pacientes con lesiones pre-tumorales |
| Features | 200  |
| Target | Binario: `1` = buen pronóstico (sin recaída) y `0` = mal pronóstico (con recaída) |
| Desbalanceo | 349 instancias clase 0 y 151 instancias clase 1 |


En un análisis exploratorio inicial verificamos que las correlaciones de Pearson entre los 200 atributos y el target se distribuyen como una normal centrada en 0 (σ ≈ 0.26), lo que indica que no hay features fuertemente predictivas de forma individual. Esto nos llevó a centrar la estrategia de partición en el desbalanceo de clases y a descartar la selección de atributos por correlación.

---

## Metodología

### Partición de datos

Hicimos una separación manual estratificada (sin `train_test_split`) manteniendo la proporción de clases del dataset original:

- Desarrollo: 80%, 400 muestras (entrenamiento + validación cruzada)
- Control (held-out): 20%, 100 muestras (reservadas para evaluación final, sin tocar durante el desarrollo)

Elegimos la proporción 80 y 20 como balance entre tener suficientes datos para entrenar y mantener un conjunto de control suficientemente representativo.

### Evaluación de modelos

- Método: K-fold cross-validation con K = 5
- Métrica principal: AUC-ROC
- Métricas secundarias: Accuracy y AUPRC

### Búsqueda de hiperparámetros

Usamos `RandomizedSearchCV` con 10 iteraciones por modelo. Los hiperparámetros explorados para cada familia:

- Árboles de decisión: `max_depth`, `criterion` y `max_features`
- KNN: `n_neighbors` de 1 al 20% del train, `weights` y `metric`
- SVM: `kernel`, `degree`, `class_weight` y `C`

LDA y Naive Bayes los evaluamos únicamente con hiperparámetros default para comparación baseline.

### Random Forest

Entrenamos con 200 árboles y exploramos el hiperparámetro `max_features` mediante curvas de complejidad para entender su efecto en sesgo y varianza.

---

## Resultados

| Modelo | Configuración | AUC-ROC (validación) |
|--------|--------------|----------------------|
| Decision Tree | optimizado (`max_depth=5`, `criterion=log_loss` y `max_features=0.36`) | ~0.58 |
| Decision Tree | default (`max_depth=3`) | ~0.58 |
| KNN | optimizado (`weights=distance`, `n_neighbors=17` y `metric=manhattan`) | ~0.84 |
| **SVC** | optimizado (`kernel=rbf`, `C=2.80` y `class_weight=balanced`) | ~0.88 |
| LDA | default | ~0.60 |
| Naive Bayes | default | ~0.64 |
| Random Forest | 200 árboles, `max_features` explorado | ~0.70 |

La evaluación final en held-out en el SVC con mejores hiperparámetros nos dio AUC-ROC = 0.9386. Las predicciones están guardadas en `10_y_pred_held_out_9386.csv`.

### Por qué SVC fue el mejor modelo

> El kernel RBF permite una separación más flexible entre clases en comparación con otros tipos de kernel. Dado que los atributos del dataset siguen distribuciones aproximadamente normales y no encontramos correlaciones fuertes entre features y target, por lo que creemos que SVC con kernel radial puede capturar relaciones no lineales que los demás algoritmos no modelan bien. Además, tiene menor sensibilidad a outliers que KNN.

---

## Diagnóstico sesgo-varianza (resumen)

- Árbol de decisión: `max_depth=4` resultó el punto de equilibrio óptimo. Profundidades mayores mejoran el AUC en train pero lo reducen en validación, lo cual es un sobreajuste clásico.
- SVM: variar C no genera cambios significativos en la varianza del modelo. Valores de C menores a 2 muestran subajuste. A partir de C=2 el AUC de validación se estabiliza.
- Random Forest: la curva de aprendizaje no muestra crecimiento significativo para n = 250, lo que sugiere que agregar más datos no sería especialmente beneficioso. La curva de train permanece cercana a 1, indicando sobreajuste en entrenamiento.
- SVC: muestra pendiente positiva en la curva de test para n=250, lo que sugiere que más datos sí podrían mejorar su generalización.

---

## Conclusiones

El objetivo del trabajo fue encontrar el mejor modelo de clasificación binaria para predecir si una lesión pre-tumoral tiene buen o mal pronóstico a partir de la expresión de 200 genes. Exploramos árboles de decisión, KNN y SVM mediante búsqueda de hiperparámetros, y los comparamos contra LDA y Naive Bayes con configuración default.

El mejor modelo resultó ser SVC con kernel RBF (C=2.80 y class_weight=balanced), alcanzando un AUC-ROC de 0.9386 en el conjunto held-out. Sospechamos que esto se debe a la naturaleza de la distribución de los datos: los features siguen un comportamiento aproximadamente normal, lo que le permite al kernel radial hacer una buena separación entre clases. A esto se suma su menor sensibilidad a outliers respecto a KNN.

Un problema que enfrentamos fue la gran cantidad de features (200), que dificultó la visualización gráfica de patrones. Para trabajos futuros, quedaría pendiente un análisis más profundo de los modelos LDA y Naive Bayes, que en este trabajo solo se evaluaron con hiperparámetros default.

---

## Estructura del repositorio

 - clasificacion_expresiones_genomicas.ipynb:  notebook con todo el análisis                                                       
 - data.csv:                                   dataset original (500 muestras x 200 features + target)                                       - desarrollo.csv:                             subconjunto de desarrollo (400 muestras 80% de data.csv)
 - control.csv:                                subconjunto held-out (100 muestras, 20% de data.csv)
 - 10_y_pred_held_out_9386.csv:                predicciones sobre X_held_out externo (AUC 0.9386)





---


## Limitaciones y mejoras posibles

- La gran cantidad de features (200) dificultó la visualización gráfica de patrones entre atributos.
- LDA y Naive Bayes los evaluamos solo con hiperparámetros default, por lo que queda pendiente explorar sus espacios de búsqueda queda como trabajo pendiente.

---

> Proyecto realizado en colaboración con compañeros para la materia Aprendizaje Automático I, FCEN-UBA 1c 2025.
