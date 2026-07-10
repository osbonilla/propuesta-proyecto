# Propuesta de Proyecto de Visualización de Datos
## El Niño en Ecuador: Visualizador Web de Anomalías Climáticas (ERA5 + ArcGIS + Svelte)

**Universidad San Francisco de Quito**

**Maestría en Ciencia de Datos — Curso de Visualización de Datos**

**Oldrin Santiago Bonilla Cáceres**

---

## Resumen de la propuesta

Se propone el desarrollo de una **aplicación web interactiva** que permita explorar cómo el fenómeno de El Niño–Oscilación del Sur (ENSO) se traduce en impactos climáticos diferenciados dentro de Ecuador. El mapa es el eje central de la exploración: una superficie continua de anomalía oceánica (causa) conectada visual e interactivamente con la respuesta terrestre. El procesamiento de datos se realiza íntegramente con Python dentro de ArcGIS Pro Notebooks (ArcPy), los servicios se publican en ArcGIS Online, y el entregable final es una app propia con **frontend en Svelte** (embebiendo el **ArcGIS Maps SDK for JavaScript**) y **backend en Python/FastAPI** para series y estadísticas contextuales. El proyecto está delimitado a un alcance realista y ejecutable.

---

# 1. Definición del Dominio (Domain Problem)

### 1.1 Pregunta central del proyecto

> ¿Cómo se traduce una misma anomalía oceánica de El Niño en respuestas climáticas y cómo puede comunicarse esa relación causa-efecto de forma visual e inmediata?

### 1.2 Contexto del problema

El Niño es un fenómeno de acoplamiento océano-atmósfera: un calentamiento anómalo del Pacífico ecuatorial que altera los patrones de lluvia y temperatura en gran parte del continente. Ecuador es uno de los países con mayor sensibilidad directa al fenómeno por su ubicación frente a la región oceánica **Niño 1+2** (la más próxima a la costa sudamericana), pero su territorio continental no responde de forma homogénea: la Costa históricamente experimenta excesos de precipitación e inundaciones durante eventos fuertes, mientras la Sierra y el Oriente muestran patrones distintos y menos estudiados públicamente.

A pesar de que los datos existen (reanálisis climático, temperatura superficial del mar, índices ENSO), **no hay una herramienta pública, interactiva y accesible que permita explorar esta relación causa-efecto de forma espacial**: la mayoría de fuentes muestran el índice ONI como una sola serie temporal nacional o global, sin desagregación territorial ni vínculo visual directo con el mapa.

### 1.3 Necesidad que motiva la visualización

La necesidad no es solo mostrar datos, sino **conectar dos escalas espaciales distintas en una sola narrativa visual**: el océano (causa, escala regional/global) y el territorio ecuatoriano (efecto, escala subnacional). Ningún gráfico de líneas puede sostener esta doble escala espacial simultáneamente; un mapa interactivo sí, porque permite alternar el foco entre el patrón oceánico y el patrón terrestre sin perder el contexto geográfico común.

### 1.4 Relevancia del proyecto

- Eventos históricos de El Niño (1982-83, 1997-98, 2015-16) han tenido consecuencias severas y bien documentadas en la Costa ecuatoriana (inundaciones, daños a infraestructura y agricultura), lo que da al proyecto un anclaje en problemas reales y no hipotéticos.
- La distinción entre el **índice ONI estándar** (región Niño 3.4) y la **anomalía en Niño 1+2** (más relevante para el impacto costero directo en Ecuador) es una diferenciación técnica genuina y poco comunicada al público, lo que le da al proyecto un ángulo original y defendible académicamente.
- El proyecto es reproducible con datos abiertos y no depende de un evento en curso: puede construirse y validarse con eventos históricos ya documentados, reduciendo el riesgo de factibilidad.

---

# 2. Abstracción de Datos (What?)

### 2.1 Fuentes de datos

| Dataset | Proveedor | Forma de acceso | Apertura |
|---|---|---|---|
| Índice ONI (Oceanic Niño Index) | NOAA Climate Prediction Center | Descarga directa (tabla de texto/CSV) | Público, sin restricciones |
| Anomalía de temperatura superficial del mar (SST), región Niño 1+2 | NOAA (ERSST / OISST) | NetCDF vía API/THREDDS | Público |
| Reanálisis atmosférico ERA5 (precipitación, temperatura 2m, viento u/v) | Copernicus Climate Data Store (ECMWF) | API `cdsapi` (requiere cuenta gratuita) | Público, licencia Copernicus |
| Límites de regiones naturales de Ecuador (Costa/Sierra/Oriente) | Fuente institucional/cartografía base oficial | Shapefile/GeoJSON | Público |
| Registro histórico de eventos ENSO (fechas, intensidad) | NOAA CPC / literatura climatológica | Tabla curada a partir de fuentes oficiales | Público |

### 2.2 Atributos y tipos de dato

Siguiendo la clasificación estándar de tipos de datos para visualización (categórico, ordinal, cuantitativo, temporal, espacial):

| Variable | Tipo de dato | Observación |
|---|---|---|
| Fase ENSO (El Niño / Neutro / La Niña) | Categórico (nominal) | Sin orden inherente entre las tres clases |
| Intensidad ENSO (débil / moderado / fuerte / muy fuerte) | Ordinal | Orden natural de magnitud |
| Valor del índice ONI (°C) | Cuantitativo continuo | Escala de razón, permite operaciones aritméticas |
| Anomalía SST en Niño 1+2 (°C) | Cuantitativo continuo + Espacial (grid) + Temporal (mensual) | Variable "causa" del proyecto |
| Anomalía de precipitación (mm o % respecto a normal) | Cuantitativo continuo + Espacial (ráster) + Temporal | Variable "efecto" principal |
| Anomalía de temperatura a 2m (°C) | Cuantitativo continuo + Espacial + Temporal | Variable "efecto" secundaria |
| Región natural (Costa / Sierra / Oriente) | Categórico (nominal) + Espacial (polígono) | Unidad de agregación para comparación |
| Fecha (mes/año) | Temporal (intervalo/ordinal) | Eje de animación y de filtrado |
| Componentes de viento (u, v) | Cuantitativo continuo, naturaleza vectorial + Espacial | Variable de apoyo físico/explicativo |

### 2.3 Justificación de la selección de variables

- **SST en Niño 1+2 en vez de (o además de) Niño 3.4**: es la métrica oceánica con mayor relación directa con el impacto costero en Ecuador; usar únicamente el índice global estándar (Niño 3.4) diluiría la especificidad regional que es el argumento central del proyecto.
- **Desagregación por región natural en vez de promedio nacional**: agregar todo el país en un solo número ocultaría precisamente el hallazgo que se busca comunicar (respuesta heterogénea). La unidad de agregación regional es la mínima necesaria para sostener la pregunta central sin diluir la señal.
- **ERA5 en vez de estaciones meteorológicas puntuales**: ofrece cobertura espacial continua sin vacíos, condición necesaria para construir una superficie cartográfica continua en el mapa (una red de estaciones dispersas no permite generar un mapa raster confiable sin fuerte interpolación adicional).
- **Viento (u, v) como variable de apoyo, no central**: se incluye porque explica físicamente el transporte de humedad hacia la Costa, reforzando la narrativa causal, pero no es la variable protagonista del análisis.

---

# 3. Abstracción de Tareas (Why?)

| Tarea (tipo) | Pregunta analítica que responde | Justificación / relación con el problema |
|---|---|---|
| **Detectar anomalías** | ¿En qué meses/años la anomalía SST en Niño 1+2 superó el umbral que define un evento El Niño (≥0.5 °C sostenido)? | Permite identificar automáticamente los periodos relevantes sin depender de una lista externa memorizada por el usuario |
| **Comparar** | ¿Cómo difieren las anomalías de precipitación y temperatura entre Costa, Sierra y Oriente durante un mismo evento El Niño? | Es la pregunta central del proyecto: la heterogeneidad regional frente a un mismo forzante climático |
| **Explorar** | ¿Qué patrón espacial de anomalía se observa en un mes específico, elegido libremente por el usuario, más allá de los eventos históricos predefinidos? | Habilita descubrimientos no anticipados por el equipo de diseño, propios de una herramienta de análisis (no solo de reporte) |
| **Identificar tendencias** | ¿Ha cambiado la frecuencia o intensidad de eventos El Niño fuertes en las últimas décadas? | Conecta el proyecto con la pregunta de fondo sobre variabilidad/cambio climático, dando profundidad analítica adicional |
| **Localizar** | Dentro de la Costa, ¿qué zonas específicas concentran las mayores anomalías de precipitación en los eventos históricos? | Da valor aplicado directo para priorización de gestión de riesgo a escala subregional |

Cada tarea está diseñada para ser resuelta primero **mirando el mapa**, y solo después consultando un gráfico contextual de apoyo. Esto asegura que la abstracción de tareas esté alineada con el principio de que el mapa es la visualización principal, no un complemento del gráfico.

---

# 4. Propuesta de Visualización (How?)

### 4.1 Wireframe / boceto preliminar

```
┌─────────────────────────────────────────────────────────────────────┐
│  El Niño en Ecuador · Visualizador Climático      [i] Acerca de     │
├─────────────────────────────────────┬───────────────────────────────┤
│                                     │  FASE ENSO ACTUAL             │
│                                     │  ●  El Niño                   │
│                                     │  ONI actual: +1.2 °C (EDA)    │
│                                     ├───────────────────────────────┤
│                                     │  SERIE HISTÓRICA ONI          │
│         MAPA PRINCIPAL              │  (1950 – hoy)                 │
│    (ArcGIS Maps SDK for JS)         │   ╭╮      ╭──╮       ╭╮       │
│                                     │ ──╯╰──────╯  ╰───────╯╰──     │
│  · Superficie de anomalía SST       │    82-83   97-98    15-16     │
│    (offshore, "causa")              ├───────────────────────────────┤
│  · Anomalía precipitación/          │  REGIÓN: COSTA (seleccionada) │
│    temperatura por región           │                               │
│    ("efecto") — clic = detalle      │  Costa    ████████████  +85%  │
│  · Límites Costa/Sierra/Oriente     │  Sierra   ███           +12%  │
│  · Vectores de viento               │  Oriente  ██████        +30%  │
├─────────────────────────────────────┴───────────────────────────────┤
│  <────────●─────────────────────────────────────────────────>[Play] |
│  1950         1982-83      1997-98      2015-16             2026    │
│               TIME SLIDER — sincroniza mapa, panel ONI y barras     │
└─────────────────────────────────────────────────────────────────────┘
```

*Nota: este es un boceto de baja fidelidad, propio de la etapa de propuesta; el diseño visual definitivo (paleta, tipografía, jerarquía) se desarrolla en la fase de implementación*

### 4.2 Justificación de codificaciones visuales

| Elemento visual | Variable codificada | Codificación elegida | Justificación perceptual |
|---|---|---|---|
| Superficie continua del mapa (color) | Anomalía de precipitación/temperatura (cuantitativo continuo, espacial) | Color sobre área continua (paleta divergente) | Para datos cuantitativos continuos distribuidos espacialmente, la codificación por posición+color sobre superficie es la de mayor precisión perceptual disponible |
| Polígonos de región | Región natural (categórico) | Color por matiz (hue) + borde | Variables categóricas de bajo cardinal (3 clases) se codifican mejor por matiz que por tamaño o posición |
| Indicador de fase ENSO | Fase ENSO (categórico) | Color semáforo + ícono | El color es la codificación preatentiva más rápida de detectar; adecuada para un estado que debe leerse "de un vistazo" |
| Serie ONI | Valor ONI en el tiempo (cuantitativo + temporal) | Línea (posición Y vs. tiempo en X) | La codificación por posición en eje continuo es la más precisa para leer tendencia y magnitud simultáneamente |
| Comparación regional | Anomalía por región (categórico × cuantitativo) | Barras agrupadas (longitud) | Para comparar magnitudes exactas entre pocas categorías, la longitud de barra supera en precisión al color o al área |
| Campo de viento | Componentes u/v (vector) | Flechas (dirección + magnitud) | Es la única codificación que preserva simultáneamente las dos dimensiones físicas del dato (no puede reducirse a un solo escalar sin pérdida) |

### 4.3 Interacciones

| Interacción | Tarea que habilita | Descripción |
|---|---|---|
| **Time Slider** (navegar/animar fecha) | Identificar tendencias, Explorar | Sincroniza simultáneamente el mapa, el indicador ONI y el gráfico de barras regional |
| **Clic en región del mapa** | Comparar, Localizar | Actualiza el panel lateral con el detalle de esa región específica (brushing & linking mapa → gráfico) |
| **Selección de rango en la serie ONI** | Detectar anomalías | Filtra el mapa para mostrar únicamente el periodo seleccionado, resaltando automáticamente los meses que superan el umbral de evento El Niño |
| **Swipe / comparación de dos fechas** | Comparar | Permite contrastar visualmente dos momentos específicos (ej. inicio vs. pico de un evento histórico) |
| **Hover sobre el mapa** | Explorar | Tooltip con el valor exacto de anomalía en el punto señalado, como apoyo a la lectura precisa sin saturar el mapa de etiquetas |

---

# 5. Factibilidad del Proyecto

### 5.1 Alcance 

Para mantener el proyecto factible dentro de un semestre académico, se fija explícitamente el siguiente alcance mínimo viable:

- **Cobertura geográfica**: Ecuador continental, con las tres regiones naturales (Costa, Sierra, Oriente) como unidad mínima de agregación —no se desagrega a nivel cantonal en esta primera versión.
- **Cobertura temporal para el mapa animado**: 3-4 eventos históricos documentados (1982-83, 1997-98, 2015-16, y el estado más reciente disponible), en lugar de animar continuamente toda la serie 1950-presente, lo cual sería costoso de procesar y de poco valor incremental para la narrativa. La serie completa del índice ONI sí se muestra como gráfico de línea (dato liviano, sin necesidad de animación cartográfica).
- **Variables**: precipitación, temperatura y viento; se deja explícitamente fuera de este alcance inicial variables adicionales como humedad de suelo o caudales hidrológicos.

### 5.2 Cronograma propuesto

| Fase | Fechas | Entregable |
|---|---|---|
| 1. Definición y obtención de datos | Sáb 11 – Dom 12 jul (2 días) | Datasets descargados y validados (ONI, SST, ERA5, límites regionales) |
| 2. Procesamiento piloto (ArcPy) | Lun 13 – Mié 15 jul (3 días) | Notebook funcional con cálculo de anomalías para un evento de prueba |
| 3. Publicación de servicios (AGOL) | Jue 16 – Vie 17 jul (2 días) | Image Service y Feature Service con tiempo habilitado, validados y sincronizados |
| 4. Backend (FastAPI) | Sáb 18 – Dom 19 jul (2 días) | Endpoints funcionando con datos reales (sección 6) |
| 5. Frontend (Svelte + ArcGIS Maps SDK) | Lun 20 – Mié 22 jul (3 días) | App integrada: mapa + panel + time slider funcionando end-to-end |
| 6. Pruebas y ajuste de diseño | Jue 23 jul (1 día) | Ajustes de paleta, leyenda, usabilidad tras retroalimentación |
| 7. Documentación y presentación final | Vie 24 – Sáb 25 jul (2 días) | Informe, repositorio y demo funcional |

### 5.3 Riesgos y estrategias de mitigación

| Riesgo | Probabilidad / Impacto | Mitigación |
|---|---|---|
| Volumen de datos NetCDF grande y tiempos de descarga largos en el CDS | Media / Media | Acotar área geográfica y periodo desde la fase piloto; solicitar descargas con anticipación |
| Desincronización entre el ráster animado y las capas vectoriales (isolíneas/vectores) en el Dashboard/app | Media / Alta | Validar en el piloto el patrón de feature class acumulada con campo `timestamp` antes de escalar (ver sección 6.2) |
| Curva de aprendizaje de integrar ArcGIS Maps SDK for JavaScript dentro de la reactividad de Svelte | Media / Media | Empezar por el ejemplo oficial mínimo de Esri antes de construir componentes a medida |
| Límites de cuota en cuentas de desarrollador de ArcGIS Online | Baja / Media | Verificar con anticipación el nivel gratuito/educativo disponible; diseñar la app para bajo volumen de solicitudes |
| Ampliación no controlada del alcance ("scope creep") al querer cubrir todos los eventos y variables posibles | Alta / Alta | Mantener estrictamente el alcance mínimo viable definido en 5.1; documentar extensiones como trabajo futuro |

---

# 6. Arquitectura de Solución

### 6.1 Diagrama de arquitectura 

```
┌────────────────────────────────────────────────────────────────────┐
│  CAPA 1 · FUENTES DE DATOS (externas, solo descarga)               │
│  · NOAA CPC → índice ONI          · NOAA OISST/ERSST → SST         │
│  · Copernicus CDS → ERA5 (precipitación, temperatura, viento)      │
└───────────────────────────────┬────────────────────────────────────┘
                                ▼
┌────────────────────────────────────────────────────────────────────┐
│  CAPA 2 · PROCESAMIENTO (ArcGIS Pro Notebooks — Python + ArcPy)    │
│  · Cálculo de anomalías, índices, series por región                │
│  · arcpy.md.* (Multidimensional Tools) sobre cubos NetCDF          │
│  · Publicación de Image Services / Feature Services (tiempo habil.)│
│  · Exportación de series resumen (CSV/Parquet) para el backend     │
└───────────┬──────────────────────────────────┬─────────────────────┘
            ▼                                  ▼
┌───────────────────────────┐   ┌─────────────────────────────────────┐
│  CAPA 3a · ArcGIS Online  │   │  CAPA 3b · Backend propio (FastAPI) │
│  · Image Services (SST,   │   │  · Sirve series y estadísticas ya   │
│    precipitación, temp.)  │   │    procesadas (nunca el ráster)     │
│  · Feature Services       │   │  · Endpoints REST (ver 6.3)         │
│    (regiones, eventos)    │   │                                     │
└─────────────┬─────────────┘   └───────────────────┬─────────────────┘
              ▼                                     ▼
┌────────────────────────────────────────────────────────────────────┐
│  CAPA 4 · FRONTEND (App web en Svelte)                             │
│  · <MapView> → ArcGIS Maps SDK for JavaScript (consume Capa 3a)    │
│  · <TimelineControl>, <RegionChart>, <ONIPanel> (consumen Capa 3b) │
└────────────────────────────────────────────────────────────────────┘
```

**Principio de diseño**: el mapa (Capa 4, componente `<MapView>`) habla directo con ArcGIS Online — nunca pasa por el backend Python. El backend FastAPI solo sirve datos ya resumidos (series, estadísticas, metadatos de eventos), por lo que permanece liviano y no reprocesa nada en cada visita del usuario.

### 6.2 Consideración técnica crítica: sincronización tiempo-mapa

Para que el ráster (anomalía) y las capas vectoriales (límites de región resaltados, marcadores de eventos) se animen **sincronizadamente** en el Time Slider de la app web, ambas deben publicarse como servicios con la propiedad de tiempo (`Time`) habilitada en ArcGIS Online — no basta con que se vean animadas dentro de ArcGIS Pro de escritorio. Si las capas vectoriales se generan por paso de tiempo, deben acumularse en una única feature class con un campo `timestamp` (mediante `arcpy.management.Append` en cada iteración del procesamiento) antes de publicarse, de forma que un solo servicio de tiempo controle ambas capas de forma coherente. Este es el criterio de validación más importante del piloto técnico antes de construir la app completa.

### 6.3 Endpoints sugeridos del backend (FastAPI)

| Endpoint | Método | Descripción |
|---|---|---|
| `/api/oni` | GET | Serie histórica completa del índice ONI con clasificación de fase |
| `/api/oni/actual` | GET | Fase y valor ONI más reciente |
| `/api/region/{nombre}/serie` | GET | Serie temporal de anomalía para una región (Costa/Sierra/Oriente), parámetros `desde`/`hasta` |
| `/api/region/{nombre}/resumen` | GET | Estadísticas agregadas para el rango de fechas seleccionado en el mapa |
| `/api/eventos` | GET | Lista de eventos históricos El Niño/La Niña (fecha inicio/fin, intensidad) para marcar hitos en la línea de tiempo |
| `/api/comparar` | GET | Compara dos periodos (ej. 1997-98 vs. 2015-16), devolviendo diferencias agregadas por región |

---

# 7. Tecnologías Utilizadas

| Categoría | Herramienta | Rol en el proyecto |
|---|---|---|
| Fuente/acceso a datos | NOAA CPC, NOAA OISST/ERSST, Copernicus CDS (`cdsapi`) | Descarga de ONI, SST y ERA5 |
| Procesamiento geoespacial | **ArcGIS Pro Notebooks**, **ArcPy** (`arcpy.md.*`, `arcpy.sa.*`) | Cálculo de anomalías, cubos multidimensionales, isolíneas, vectores |
| Integración tabular-espacial | **ArcGIS API for Python** (`arcgis.features.GeoAccessor`) | Conversión de tablas a capas espaciales |
| Publicación de servicios | **ArcGIS Online** (Image Services, Feature Services con tiempo habilitado) | Hosting del mapa y sus capas |
| Mapa interactivo (frontend) | **ArcGIS Maps SDK for JavaScript** (`@arcgis/core`) | Renderiza el mapa dentro de la app Svelte |
| Framework de frontend | **Svelte** (o SvelteKit) | Estructura de componentes y estado reactivo de la app |
| Gráficos contextuales | **D3.js** o **Chart.js** | Serie ONI, comparación de barras por región |
| Backend / API | **FastAPI** (Python) | Sirve series y estadísticas ya procesadas |
| Almacenamiento ligero | **SQLite** o archivos **Parquet** | Series agregadas para respuesta rápida del backend |
| Manipulación de datos | **Pandas**, **NumPy** | Limpieza y cálculos auxiliares en el pipeline |

---

# 9. Conclusión

Esta propuesta define un dominio de problema real, relevante y con audiencia identificada; realiza una abstracción de datos rigurosa, especificando fuentes, atributos y tipos de dato con justificación de selección; define tareas analíticas explícitas y vinculadas al problema; propone un diseño preliminar con codificaciones e interacciones justificadas perceptualmente; y delimita un alcance realista, con cronograma y riesgos mitigables. La arquitectura de solución y el stack tecnológico demuestran que el proyecto no es solo conceptualmente sólido sino **técnicamente ejecutable de principio a fin** dentro del ecosistema ArcGIS combinado con un frontend propio en Svelte.
