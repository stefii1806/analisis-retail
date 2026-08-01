# 🛒 Segmentación de clientes mayoristas (wholesale)

Análisis de clustering y clasificación sobre el dataset público *Wholesale customers* (paquete `tclust`), evaluando qué tan bien el perfil de gasto de 440 clientes de un distribuidor mayorista predice su canal de venta (Horeca vs. Retail).

## 🎯 Objetivo

Determinar si existe una estructura de gasto que separe a los clientes Horeca de los clientes Retail, comparando cuatro métodos de clustering no supervisado entre sí y contra el canal real, y construir un clasificador supervisado que prediga el canal a partir de 6 variables de gasto anual.

## 🗂️ Estructura del repositorio

```
├── analysis/
│   └── analisis_retail.R   # Script completo: EDA, PCA, clustering, clasificación supervisada
├── output/                 # Gráficos exportados del análisis
├── REPORTE_HALLAZGOS.md    # Interpretación y conclusiones en detalle
└── README.md
```

> El dataset (`wholesale`) viene incluido en el paquete de R `tclust` — no hace falta descargar ningún archivo aparte.

## 🔎 Etapas del análisis

1. **Exploración inicial**: tipos de variable, transformación logarítmica (todas las variables de gasto tenían sesgo a la derecha), correlaciones por canal y región, normalidad multivariada y outliers (Mahalanobis).
2. **Reducción de dimensionalidad**: PCA sobre las 6 variables de gasto transformadas y escaladas.
3. **Clustering jerárquico** (Ward), eligiendo k mediante scree plot de alturas y silueta, comparado contra `Channel` con ARI.
4. **Comparación de 4 métodos de clustering**: k-means, GMM (Mclust), DBSCAN y OPTICS — evaluados entre sí y contra `Channel` con ARI y NMI.
5. **Clasificación supervisada**: partición 70/30, y comparación de LDA, EDDA (MclustDA) y mixturas gaussianas semisupervisadas (MclustSSC) para predecir `Channel`.

## 📈 Principales hallazgos

- **PCA**: PC1+PC2 explican el 71% de la varianza; PC1 separa parcialmente los canales (Retail hacia Milk/Grocery/Detergent, Horeca más disperso), con overlap considerable.
- **Clustering jerárquico (Ward, k=2)**: ARI = 0.415 vs. Channel — concordancia moderada.
- **De los 4 métodos de clustering comparados, k-means (k=2) es el más adecuado**: ARI = 0.513, NMI = 0.427 frente al canal real, con alta estabilidad bootstrap (Jaccard > 0.95).
- **GMM** encuentra una segmentación más fina (4 subperfiles, modelo VVE) que trasciende la dicotomía Horeca/Retail.
- **DBSCAN y OPTICS no logran segmentar el dataset** — encuentran un único grupo denso + un puñado de outliers como ruido, sin relación con el canal real.
- **Clasificación supervisada**: EDDA alcanza **92.4% de accuracy** en el conjunto de prueba (superando a LDA en ~2pp), validado además con 100 particiones aleatorias distintas. Las mixturas semisupervisadas igualan la accuracy pero redistribuyen levemente los errores entre canales.

📄 Ver [`REPORTE_HALLAZGOS.md`](REPORTE_HALLAZGOS.md) para la interpretación completa, con tablas comparativas por método.

## 🔧 Cómo correrlo

```r
install.packages(c(
  "GGally", "MVN", "factoextra", "cluster", "dendextend", "mclust",
  "aricode", "fpc", "dbscan", "caret", "ggplot2", "dplyr",
  "RColorBrewer", "tclust"
))

source("analysis/analisis_retail.R")
```

## 🛠️ Tecnologías

R · mclust (GMM, EDDA, MclustSSC) · factoextra · cluster · dbscan · caret (LDA, validación cruzada)

## 📬 Contacto

[Stefania Cuicchi](https://portfolio-stefania-cuicchi.vercel.app/) · [LinkedIn](https://linkedin.com/in/scuicchi)
