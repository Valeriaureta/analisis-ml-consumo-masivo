# analisis-ml-consumo-masivo

Una empresa peruana de consumo masivo buscaba entender la caída comercial de una familia prioritaria durante el periodo 2024-2025.

El problema no podía explicarse solo observando la venta total. Era necesario analizar ventas históricas, lanzamientos comerciales y variables externas para identificar qué zonas, clientes y SKUs concentraban la pérdida, y qué señales podían anticipar oportunidades de recuperación. Por ello, el proyecto buscó pasar de un diagnóstico descriptivo a una herramienta de priorización comercial, construyendo un modelo para identificar combinaciones `SKU-zona` con mayor probabilidad de alcanzar ventas altas en el siguiente mes.

Nota: Por motivos de confidencialidad, el nombre de la empresa, las familias de producto y los códigos de SKU fueron anonimizados. Los nombres utilizados en este repositorio son referenciales y mantienen la estructura del análisis original sin exponer información sensible del negocio.

---

## Estructura del proyecto

```text
.
├── data
│   ├── raw
│   │   ├── df_ventas.csv
│   │   ├── bd-vars-lanzamiento.xlsx
│   │   ├── bd-vars-macroeconomicas.csv
│   │   └── BCRP.csv
│   │
│   └── processed
│       ├── df_ventas1.csv
│       ├── df_integrado_frescura.csv
│       ├── df_integrado_frescura_macro.csv
│       ├── df-base-modelado.csv
│       └── ranking_oportunidades_frescura.csv
│
├── notebooks
│   ├── EDA_ventas1.ipynb
│   ├── EDA_varslanzamiento.ipynb
│   ├── EDA_varsmacro.ipynb
│   ├── Integracion.ipynb
│   └── modelos_ml_frescura.ipynb
│
├── anonimizar_ventas.py
└── README.md
```

---

# Desarrollo del proyecto

El proyecto se desarrolló en seis etapas principales:

1. **EDA de ventas**
   Se analizó el desempeño comercial por familia, zona, cliente, SKU, canal, lista de precio, grupo de precio, descuentos y costos asociados.

2. **EDA de lanzamientos**
   Se revisaron eventos comerciales, duración de campañas y periodos posteriores a lanzamientos para evaluar su relación con el comportamiento mensual de ventas.

3. **EDA de variables macroeconómicas**
   Se analizaron variables externas como tipo de cambio, inflación, precios y commodities como contexto adicional del desempeño comercial.

4. **Diagnóstico comercial de la familia prioritaria**
   Se identificaron los principales focos de caída de `Frescura`, profundizando el análisis por zonas, clientes y SKUs críticos.

5. **Integración y construcción de base predictiva**
   Se consolidaron las fuentes relevantes y se agregaron ventas a nivel `periodo + SKU + zona`. Luego se crearon variables comerciales, históricas y de lanzamiento.

6. **Modelamiento predictivo**
   Se entrenaron modelos de clasificación para estimar la probabilidad de que una combinación `SKU-zona` alcance una venta alta en el siguiente mes y generar un ranking de oportunidades comerciales.

---

# Principales hallazgos del EDA

El análisis exploratorio permitió identificar que la caída no era homogénea, sino que se concentraba en focos específicos del negocio.

| Dimensión                 | Hallazgo clave                                                                                                                                                             |
| ------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Familia prioritaria       | `Frescura` fue seleccionada como foco principal porque concentraba cerca del **40%** de la venta total y presentó una caída cercana al **47%** frente al periodo anterior. |
| Evolución mensual         | La caída mostró volatilidad y quiebres relevantes desde el cierre de 2024, por lo que no bastaba analizar únicamente el resultado acumulado anual.                         |
| Zonas                     | `CUSCO` y `LIMA` tuvieron alta relevancia comercial, con reducciones importantes en el periodo evaluado.                                                                   |
| Clientes                  | Se observó una menor base activa en 2025, incluyendo clientes con venta en 2024 que no registraron venta en el año siguiente.                                              |
| SKUs                      | La reducción no se explicó únicamente por el SKU líder; varios productos contribuyeron a la caída.                                                                         |
| Lanzamientos              | Los eventos activos y los periodos posteriores al lanzamiento mostraron señales asociadas a variaciones en la venta.                                                       |
| Variables macroeconómicas | Se analizaron como contexto externo; sin embargo, después de evaluar su cobertura y aporte al modelo, se decidió no incorporarlas en el modelamiento final.                |

---

# Pruebas estadísticas e hipótesis

Además del EDA, se aplicaron pruebas estadísticas para respaldar las principales decisiones del análisis:

* Se utilizó `ANOVA` para comparar la venta promedio según familia, zona, SKU, grupo de precio y lista de precio.
* Los resultados mostraron diferencias significativas en las principales dimensiones comerciales evaluadas (`p-value < 0.001`).
* En `Frescura`, también se encontraron diferencias significativas por zona y SKU (`p-value < 0.001`).
* Esto respaldó la decisión de construir el modelo a nivel `SKU-zona`, en lugar de mantener el análisis solo a nivel familia.
* La comparación entre 2024 y 2025 mostró una diferencia significativa en ventas (`p-value < 0.001`).
* Los lanzamientos y eventos comerciales mostraron asociación positiva con las ventas, por lo que se incorporaron como señales complementarias en la base de modelado.

---

# Definición de la variable objetivo

Para el modelamiento, se definió una venta alta usando el criterio de rango intercuartílico de Tukey. Este criterio permitió identificar ventas inusualmente altas en relación con el comportamiento histórico de cada combinación `SKU-zona`.

La regla utilizada fue:

```text
Límite superior = Q3 + 1.5 * IQR
```

Una observación fue clasificada como venta alta cuando:

```text
venta_mensual > límite superior
```

Luego, la etiqueta fue desplazada un mes hacia adelante para construir la variable:

```text
venta_alta_proximo_mes
```

De esta forma, el modelo no solo describe ventas altas pasadas, sino que busca anticipar oportunidades comerciales futuras.

---

# Variables utilizadas para el modelado

Para construir la base predictiva, se trabajó a nivel `periodo + SKU + zona`, ya que esta granularidad permite generar recomendaciones accionables para el negocio.

La base final combinó tres tipos de información:

* **Variables comerciales originales:** producto, zona, grupo de precio, lista de precio y métricas agregadas de venta.
* **Variables históricas derivadas:** rezagos de venta, medias móviles, tendencia reciente y frecuencia de venta.
* **Variables de lanzamiento:** indicadores de campañas, lanzamientos y eventos comerciales disponibles por periodo.

La variable objetivo fue `venta_alta_proximo_mes`, construida a partir del criterio de Tukey para identificar ventas inusualmente altas y desplazar esa etiqueta al mes siguiente.

---

# Separación entrenamiento-prueba

Se utilizó un corte temporal para simular un escenario real de predicción.

```text
Corte temporal: agosto de 2025
```

La separación fue:

| Partición | Periodo de predicción | Filas | Casos positivos | Tasa de positivos |
| --------- | --------------------- | ----: | --------------: | ----------------: |
| Train     | 2024-02 a 2025-07     | 4,464 |              83 |             1.86% |
| Test      | 2025-08 a 2025-12     | 1,240 |              26 |             2.10% |

Esta decisión permitió entrenar con información histórica y evaluar sobre periodos futuros, evitando fuga de información.

---

# Modelos evaluados y resultados

Se compararon tres enfoques:

* **Baseline de negocio:** regla simple basada en señales comerciales recientes.
* **Regresión Logística:** modelo interpretable para estimar probabilidad de oportunidad.
* **Random Forest:** modelo no lineal para capturar patrones más complejos entre historial de ventas, SKU, zona y lanzamientos.

Las métricas se calcularon usando un enfoque operativo Top-K, donde el modelo prioriza las principales oportunidades comerciales.

| Modelo              | Precision | Recall | F1-score | ROC-AUC | PR-AUC |
| ------------------- | --------: | -----: | -------: | ------: | -----: |
| Baseline de negocio |      0.02 |   0.04 |     0.03 |    0.77 |   0.04 |
| Regresión Logística |      0.00 |   0.00 |     0.00 |    0.55 |   0.03 |
| Random Forest       |      0.06 |   0.12 |     0.08 |    0.73 |   0.06 |

También se evaluó el desempeño bajo un enfoque Top-K:

| Modelo              | Precision Top-K | Recall Top-K | Uso de negocio                       |
| ------------------- | --------------: | -----------: | ------------------------------------ |
| Baseline de negocio |            0.02 |         0.04 | Referencia inicial                   |
| Regresión Logística |            0.00 |         0.00 | Priorización interpretable           |
| Random Forest       |            0.06 |         0.12 | Ranking de oportunidades comerciales |

El modelo `Random Forest` obtuvo el mejor desempeño bajo el enfoque de priorización, alcanzando el mayor `PR-AUC` y la mejor combinación de `Precision Top-K` y `Recall Top-K`.

Aunque las métricas absolutas son moderadas, el problema es altamente desbalanceado: solo alrededor del **2%** de las observaciones del conjunto de prueba corresponden a ventas altas. Por ello, el valor del modelo está en mejorar la priorización frente a una revisión no dirigida.

---

# Hallazgos importantes para el negocio

El proyecto permitió convertir una caída comercial en una herramienta de diagnóstico y priorización.

El principal entregable fue:

```text
ranking_oportunidades_frescura.csv
```

Este archivo prioriza combinaciones `SKU-zona` según su probabilidad estimada de alcanzar una venta alta en el siguiente mes.

El valor generado para el negocio fue:

* identificar qué combinaciones `SKU-zona` deberían revisarse primero;
* focalizar recursos comerciales en oportunidades con mayor potencial;
* complementar el seguimiento de lanzamientos y campañas;
* priorizar acciones sobre zonas y productos críticos;
* apoyar la planificación comercial mensual con una salida accionable basada en datos.

En lugar de limitarse a explicar una caída pasada, el proyecto genera una base priorizada que puede apoyar decisiones comerciales del siguiente periodo.
