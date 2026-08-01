# Reporte de hallazgos — Segmentación de clientes mayoristas (wholesale)

Dataset: 440 clientes de un distribuidor mayorista (0 valores faltantes), con 6 variables de gasto anual (Fresh, Milk, Grocery, Frozen, Detergents_Paper, Delicatessen) y dos variables categóricas: `Channel` (Horeca / Retail) y `Region` (Lisboa / Oporto / Otras).

## Exploración

Las 6 variables de gasto tienen sesgo marcado a la derecha; al no haber ceros en ninguna, se aplicó una transformación logarítmica que mejora notablemente tanto los histogramas como los Q-Q plots. Separando las correlaciones por `Channel` se observa mejor diferenciación entre grupos que separando por `Region` (cuyas distribuciones se solapan bastante), por lo que se usa `Channel` como variable de referencia en el resto del análisis.

El energy test rechaza la normalidad multivariada (p < 0.001), y el Q-Q plot de Mahalanobis vs. chi-cuadrado confirma que los puntos se alejan de la línea teórica. Con el umbral chi-cuadrado (0.975, df=6) se detectan **30 posibles outliers multivariados** — se toman solo como indicio, dado que no hay normalidad multivariada de base.

## PCA

PC1 + PC2 explican el **71.12%** de la varianza (Kaiser-Guttman también retiene 2 componentes).
- **PC1**: mayor peso de Milk, Grocery, Detergent y Delicatessen.
- **PC2**: mayor peso de Fresh y Frozen (y, en menor medida, Delicatessen).

El biplot muestra que PC1 separa parcialmente los canales: Retail tiende hacia valores positivos (asociados a Milk/Grocery/Detergent), mientras que Horeca aparece más disperso y hacia valores negativos o centrales. La separación es clara pero incompleta, con overlap considerable en la zona central — anticipando lo que después se confirma en el clustering.

## Clustering jerárquico (Ward)

Ningún heatmap de distancias (euclidiana ni Manhattan) muestra estructura de bloques clara. Entre los 4 linkages probados, Ward produce el dendrograma más ordenado e interpretable. Tanto el scree plot de alturas como la silueta promedio sugieren **k = 2**, aunque la silueta máxima es baja (0.26) — hay agrupamiento real, pero poco definido.

- Grupos de tamaño 206 y 234.
- Silueta promedio del cluster 2 (~0.326) más alta que la del cluster 1.
- 36 observaciones (8.18%) con silueta negativa, es decir, mal ajustadas a su cluster.
- **ARI vs. Channel = 0.415**: coincidencia moderada, por encima del azar pero lejos de perfecta. Horeca se recupera razonablemente bien, pero Retail queda bastante mezclado entre ambos clusters.

## Comparación de 4 métodos de clustering (k-means, GMM, DBSCAN, OPTICS)

**K-means.** Los tres criterios de selección de k (WSS, silueta, gap statistic) coinciden en **k=2**. Grupos de 188 y 252 observaciones, con 30% de variabilidad explicada por la partición.
- Cluster 1 ≈ Retail (71.28% de sus miembros), Cluster 2 ≈ Horeca (96.8% de sus miembros).
- Alineando clusters con el canal mejor representado: 85.91% de acierto (378/440).
- **ARI vs. Channel = 0.513, NMI = 0.427** — la mejor concordancia entre los 4 métodos.
- Silueta promedio = 0.29. Estabilidad bootstrap muy alta (Jaccard: 0.963 y 0.953).
- ARI vs. clustering jerárquico = 0.639 — coincidencia moderada, no idéntica.

**GMM (Mclust).** El mejor modelo por BIC es **VVE con 4 componentes** (grupos elipsoidales, distinto volumen y forma, misma orientación) — más flexible que k-means, y encuentra el doble de grupos. El ICL, algo más bajo que el BIC, indica cierto solapamiento entre componentes.
- Comparando contra Channel, los grupos 3 y 4 son mayoritariamente Horeca, el grupo 2 se asocia más a Retail, y el grupo 1 no muestra una división clara — el GMM está subdividiendo en perfiles que trascienden el canal de venta.
- **ARI vs. Channel = bajo** (esperable, al comparar 4 grupos contra 2 niveles). ARI vs. k-means = 0.409, vs. jerárquico = 0.236 (menor aún, por la diferencia en cantidad de grupos).
- La reducción MclustDR muestra que Dir1 (dominada por Detergent) y Dir2 (dominada por Milk) acumulan 99.69% de la separación entre los 4 componentes.

**DBSCAN.** Con minPts=5 y eps cercano al codo del gráfico de k-distancias (~1.8-2.0), el método encuentra **un único grupo denso** (418 obs) + 22 observaciones de ruido, para todo el rango de eps probado. No logra segmentar en grupos comparables con Channel — el ARI resulta negativo y la NMI, prácticamente nula. El método basado en densidad no resulta adecuado para este dataset.

**OPTICS.** Confirma el diagnóstico de DBSCAN: con cortes bajos, deja gran parte de las observaciones como ruido; con cortes altos (incluyendo el mismo eps=1.8 de DBSCAN), vuelve a unir casi todo en un solo grupo (418 obs + 22 ruido). ARI y NMI vs. Channel son insignificantes.

**Conclusión de esta etapa**: k-means (k=2) es el método más adecuado, con el mayor ARI (0.513) y NMI (0.427) frente a Channel. GMM encuentra una segmentación más fina (4 grupos) pero con menor concordancia con el canal real. DBSCAN y OPTICS no son adecuados para este dataset.

| Método | Grupos | Ruido | ARI vs. Channel | NMI vs. Channel |
|---|---|---|---|---|
| k-means | 2 | 0 | 0.513 | 0.427 |
| GMM | 4 | 0 | bajo | — |
| DBSCAN | 1 | 22 | negativo | ~0 |
| OPTICS | 1 | 22 | -0.0135 | 0.00185 |

## Clasificación supervisada (Channel ~ variables de gasto)

Partición 70/30 (308 train / 132 test), con proporciones de Channel muy similares en ambos conjuntos (68.18%/31.81% en train vs. 66.66%/33.33% en test). El centrado y escalado se ajustó solo con el conjunto de entrenamiento para evitar data leakage.

**LDA.** LD1 está dominada principalmente por Detergent, y en menor medida por Milk y Grocery. En test: 120/132 aciertos (accuracy = 0.909, error = 0.09). La validación cruzada leave-one-out da un resultado muy similar (accuracy ≈ 0.91, Kappa = 0.80), igual que k-fold de 10 pliegues (accuracy ≈ 0.92, Kappa ≈ 0.82) y bootstrap de 200 réplicas (accuracy ≈ 0.91, Kappa ≈ 0.79) — el modelo es estable frente a distintos esquemas de validación.

**EDDA (MclustDA).** Selecciona el mismo modelo de covarianza VVE que apareció en el GMM no supervisado. En test: 122/132 aciertos (accuracy = 0.924, error = 0.076, Kappa = 0.84, sensibilidad = 0.91, especificidad = 0.95, balanced accuracy = 0.93) — **supera a LDA en casi 2 puntos porcentuales**. El error aparente (sobre el propio entrenamiento) es ~9.42%, similar al error en test, lo que descarta sobreajuste relevante. Una validación con 100 particiones aleatorias distintas confirma que la accuracy se mantiene consistentemente entre ~0.90 y ~0.92 — el modelo no depende de la partición inicial.

**Mixturas gaussianas semisupervisadas (MclustSSC).** Usa las 308 observaciones etiquetadas de entrenamiento y trata las 132 de test como no etiquetadas. Selecciona el mismo modelo VVE que EDDA. Resultado: **misma accuracy global que EDDA (0.92)**, pero con distribución de errores distinta — MGS clasifica mejor los casos de Channel 2 (43/44 aciertos) a costa de cometer un error adicional en Channel 1. Esto es coherente con que ambos métodos convergen a la misma estructura VVE de un componente por clase, por lo que incorporar las observaciones no etiquetadas no mejora la accuracy global, aunque sí redistribuye levemente los errores entre canales.

| Método | Accuracy | Error |
|---|---|---|
| LDA | 0.909 | 0.091 |
| EDDA | 0.924 | 0.076 |
| MGS | 0.924 | 0.076 |

**Conclusión de esta etapa**: tanto EDDA como MGS superan a LDA en accuracy y error. Al ser EDDA el modelo más simple de los dos (no requiere el paso semisupervisado adicional), se elige como el clasificador final para predecir `Channel` a partir del perfil de gasto.

## Conclusión general

El canal de venta (Horeca vs. Retail) deja una huella real en el perfil de gasto de los clientes mayoristas, visible tanto en el PCA (PC1) como en el clustering (k-means recupera el 85.9% de los casos alineando clusters con canales) y, sobre todo, en la clasificación supervisada: EDDA alcanza 92.4% de accuracy en el conjunto de prueba a partir de solo 6 variables de gasto. Sin embargo, la señal no es perfecta — hay overlap considerable entre canales (silueta máxima de 0.26-0.29 en todos los métodos de clustering no supervisado), producto de clientes cuyo perfil de gasto no encaja claramente en ninguno de los dos canales. El GMM sugiere, además, que existe una segmentación más fina (4 subperfiles) que trasciende la dicotomía Horeca/Retail, una vía natural para profundizar el análisis más allá del canal declarado.
