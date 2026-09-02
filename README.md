# Modelado predictivo del sub-reporte de causa en incendios forestales (FPA-FOD)

**Seminario Final — Licenciatura en Ciencia de Datos, Universidad Nacional Guillermo Brown (UNaB)**

Facundo Tarizzo

> ⚠️ Trabajo en curso. La hipótesis y el enfoque están definidos; el modelo está entrenado y
> validado, pero el análisis sigue en desarrollo y algunos resultados pueden ajustarse.

---

## Hipótesis

El 8,9% de los incendios de la base FPA-FOD (166.723 de 1,88 millones) no tiene causa de
ignición reportada. La hipótesis de este trabajo es que esa ausencia **no es aleatoria**:
responde a factores administrativos —jurisdicción, período y agencia responsable del
reporte— y no a características físicas del incendio. Si eso es cierto, debería ser
predecible usando únicamente esas variables.

La hipótesis surgió de la exploración del dataset, no al revés (ver
[Cómo llegué a esta hipótesis](#cómo-llegué-a-esta-hipótesis)).

---

## Datos

**FPA-FOD** (Fire Program Analysis Fire-Occurrence Database), compilada por Karen Short para
el USDA Forest Service: 1,88 millones de incendios forestales en Estados Unidos entre 1992 y
2015, con ubicación, fecha, tamaño, jurisdicción y causa.

- Formato: SQLite, tabla `Fires`
- Fuente: Kaggle — *1.88 Million US Wildfires*
- El archivo no está incluido en el repo por su tamaño (~800 MB)

Variable objetivo: `ES_MISSING`, construida como
`STAT_CAUSE_DESCR == 'Missing/Undefined'`.

---

## Hallazgos exploratorios

**Concentración geográfica-temporal.** El missing no se distribuye parejo: al calcular el
porcentaje de causa faltante por combinación Estado-Año, la distribución resulta bimodal
(los estados-año reportan casi todo, o casi nada). 62 combinaciones Estado-Año superan el
80% de missing y concentran el **60,6%** de todos los registros sin causa del dataset.

**Patrón institucional.** La tasa de missing varía enormemente según el tipo de agencia
que genera el reporte:

| Agencia | Significado | % Missing | n registros |
|---|---|---|---|
| FS | Forest Service | 0,03% | 220.497 |
| BIA | Bureau of Indian Affairs | 0,40% | 119.943 |
| BLM | Bureau of Land Management | 5,86% | 97.034 |
| ST/C&L | Estado / Condado / Local | 9,92% | 1.377.090 |
| IA | Interagency Organization | 99,88% | 21.841 |

Las agencias con procesos de reporte estandarizados (Forest Service) casi nunca omiten la
causa; los reportes generados por coordinación entre múltiples organismos (IA) la omiten
casi siempre. `ST/C&L` domina el volumen del dataset (73%), por lo que aun con una tasa
moderada es la que más aporta al total de faltantes en términos absolutos.

---

## Metodología

**Modelo:** clasificación binaria con CatBoost (gradient boosting), comparado contra Random
Forest como referencia.

**Validación temporal**, no aleatoria: entrenamiento con 1992–2010, prueba con 2011–2015.
Un split aleatorio sobre datos con autocorrelación espacial y temporal sobreestima la
performance, porque el conjunto de prueba termina conteniendo casos casi idénticos a los de
entrenamiento (Roberts et al., 2017). La caída de métricas al pasar de K-fold aleatorio a
validación temporal se verificó empíricamente en este dataset.

**Variables:**

- Físicas / temporales: `FIRE_YEAR`, `DISCOVERY_DOY`, `LATITUDE`, `LONGITUDE`, `FIRE_SIZE`
- Administrativas: `STATE`, `OWNER_DESCR`, `NWCG_REPORTING_AGENCY`
- Derivadas (*target encoding*): tasa histórica de missing por Estado, por Agencia, y por
  Estado × Agencia — **calculadas exclusivamente sobre el período de entrenamiento** para
  evitar fuga de información hacia el período de prueba.

**Métricas:** AUC-ROC y AUC-PR (*average precision*). Se prioriza AUC-PR porque con 8,9% de
clase positiva el accuracy y el AUC-ROC pueden verse inflados por lo fácil que resulta
acertar en la clase mayoritaria.

---

## Resultados

| Métrica | Resultado | Referencia (azar) |
|---|---|---|
| AUC-ROC | 0,8306 | 0,500 |
| AUC-PR | 0,4282 | 0,089 |

El AUC-PR mejora aproximadamente **4,8x** sobre el azar.

**Umbral de decisión.** Se evaluaron distintos puntos de corte en lugar del 0,5 por defecto:

| Umbral | Precisión | Recall | F1 |
|---|---|---|---|
| 0,10 | 0,297 | 0,596 | 0,397 |
| 0,20 | 0,430 | 0,516 | 0,469 |
| **0,30** | **0,535** | **0,505** | **0,519** |
| 0,40 | 0,491 | 0,369 | 0,422 |

El umbral 0,30 ofrece el mejor balance. Para un uso tipo alerta temprana (priorizar revisión
de reportes en riesgo de quedar incompletos) convendría un umbral más bajo, privilegiando
recall sobre precisión.

**Importancia de variables:**

| Variable | Importancia |
|---|---|
| Tasa histórica de missing (Estado × Agencia) | 50,3% |
| `FIRE_YEAR` | 33,6% |
| Tasa histórica de missing (Estado) | 4,9% |
| `OWNER_DESCR` | 4,4% |
| `LATITUDE` | 3,3% |
| `LONGITUDE` | 1,9% |
| `STATE` | 1,0% |
| **`FIRE_SIZE`** | **0,3%** |
| **`DISCOVERY_DOY`** | **0,0%** |

Este es el resultado que más respalda la hipótesis: el tamaño del incendio y la
estacionalidad **no aportan poder predictivo**. Lo que predice el sub-reporte es dónde,
cuándo y quién generó el registro.

**Comparación de modelos:**

| Modelo | AUC-ROC | AUC-PR |
|---|---|---|
| CatBoost | 0,8306 | 0,4282 |
| Random Forest | 0,8570 | 0,4159 |

Random Forest ordena mejor el conjunto completo (AUC-ROC), pero CatBoost es más preciso en
la clase minoritaria (AUC-PR), que es la que interesa. Se adopta CatBoost como modelo
principal por esa razón.

---

## Cómo llegué a esta hipótesis

El enfoque inicial era predecir la **causa** del incendio (13 categorías). Se probó un
pipeline jerárquico en dos etapas:

- Etapa 1 — Natural (rayo) vs. Antrópico: **macro-F1 = 0,82**
- Etapa 2 — Causa específica dentro de las antrópicas (Top 5 + Otras): **macro-F1 = 0,37–0,39**

El techo de la Etapa 2 se mantuvo pese a probar balanceo de clases, codificación cíclica de
la estacionalidad, comparación entre modelos y búsqueda de hiperparámetros con Optuna.

Al contrastar con literatura publicada, ese techo resultó consistente con el estado del arte:
Pourmohamad et al. (2025), usando la versión ampliada del dataset con ~270 covariables
ambientales, sociales y de manejo, alcanzan 93% separando natural de antrópico pero solo 55%
discriminando entre 11 causas humanas. Coffield et al. (2019), prediciendo tamaño final de
incendio al momento de ignición, reportan 50,4% en tres categorías.

Es decir: el límite no está en el modelo ni en el enfoque, sino en la información disponible
en la versión pública del dataset. Ese diagnóstico motivó el cambio de pregunta hacia el
sub-reporte, que sí resultó predecible con las variables disponibles.

---

## Limitaciones

- Las variables de tasa histórica de missing capturan en buena medida la **persistencia
  temporal del propio fenómeno** ("lo que falló antes tiende a seguir fallando"), más que el
  descubrimiento de una causa oculta. Es un modelo de perfil de riesgo histórico —válido y de
  uso práctico, como el scoring crediticio— pero conviene presentarlo con esa aclaración.
- Algunas agencias tienen muestras muy pequeñas (DOD: 81 registros, BOR: 14, DOE: 2). Sus
  tasas extremas de missing son reales pero no generalizables.
- La clasificación de causa específica (enfoque inicial) mantiene su techo de macro-F1
  ~0,37–0,39. Se conserva como antecedente metodológico, no como resultado central.

---

## Estructura del repositorio

```
├── notebooks/
│   └── 01_analisis_y_modelo.ipynb    Análisis exploratorio y modelo final
├── docs/
│   └── plan_de_trabajo.pdf           Plan de trabajo formal
└── figuras/                          Gráficos generados
```

---

## Referencias

- Short, K. C. (2017). *Spatial wildfire occurrence data for the United States, 1992–2015
  [FPA_FOD_20170508]* (4ª ed.). Fort Collins, CO: Forest Service Research Data Archive.
- Pourmohamad, Y., Abatzoglou, J. T., Fleishman, E., Short, K. C., Shuman, J., AghaKouchak,
  A., Williamson, M., Seydi, S. T., & Sadegh, M. (2025). Inference of wildfire causes from
  their physical, biological, social and management attributes. *Earth's Future*, 13(1).
  https://doi.org/10.1029/2024EF005187
- Coffield, S. R., Graff, C. A., Chen, Y., Smyth, P., Foufoula-Georgiou, E., & Randerson,
  J. T. (2019). Machine learning to predict final fire size at the time of ignition.
  *International Journal of Wildland Fire*, 28(11), 861–873.
  https://doi.org/10.1071/WF19023
- Roberts, D. R., Bahn, V., Ciuti, S., Boyce, M. S., Elith, J., Guillera-Arroita, G.,
  Hauenstein, S., Lahoz-Monfort, J. J., Schröder, B., Thuiller, W., Warton, D. I.,
  Wintle, B. A., Hartig, F., & Dormann, C. F. (2017). Cross-validation strategies for data
  with temporal, spatial, hierarchical, or phylogenetic structure. *Ecography*, 40(8),
  913–929. https://doi.org/10.1111/ecog.02881
