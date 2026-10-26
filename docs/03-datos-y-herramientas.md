---
title: "More Than A Hack 2027 — Fase 1"
subtitle: "Datos, herramientas y entrega"
date: "25 de octubre de 2026"
---

# 1. Qué hay en la carpeta de datos

Ruta local del equipo:

```
/home/ian/gdrive-shared/More Than A Hack 2027 - Initial Challenge - 10/
  More Than A Hack 2027 - Initial Challenge - Dataset/
```

**19 GB en total**, de los cuales 18,9 GB son vídeo que probablemente no se
llegue a abrir.

| Fichero | Tamaño | Para qué |
|---|---|---|
| `tactical/BAR-ATH_tactical.mp4` | 5,2 GB | El vídeo de referencia. **No hace falta** |
| `tactical/BAR-ATH_tactical_atd.txt` | 196 MB | **El tracking.** La materia prima real |
| `tactical/BAR-ATH_tactical_atd.xml` | 59 KB | Metadatos: equipos y a qué equipo pertenece cada slot |
| `tactical/BAR-ATH_tactical_homography.csv` | 23 MB | Campo → píxeles. Solo si se toca vídeo |
| `tactical/BAR-ATH_tactical_smart_tagging.xml` | 261 KB | Las 638 fases tácticas |
| `broadcast/BAR-ATH_broadcast.mp4` | 14 GB | Segunda cámara. **No sincronizada**. Prescindible |
| `broadcast/BAR-ATH_broadcast_atd.txt` | 142 MB | Tracking del broadcast (164.488 frames) |
| `broadcast/BAR-ATH_broadcast_homography.csv` | 15 MB | Válido solo en el 65,8 % |

> **Trampa de formato:** si la entrega tiene ~164.000 filas, indexaste el
> broadcast. El validador lo rechaza. La referencia es **siempre** la táctica.

## Copia local obligatoria

Cinco personas leyendo 196 MB de un drive compartido van a ir lentas y a
bloquearse entre sí. **Una persona copia el bundle a disco local una vez** y el
resto apunta a esa copia con la variable `MTAH_DATA_DIR`.

---

# 2. Alineación de fotogramas

| Fuente | Índice | Para obtener la fila de la entrega |
|---|---|---|
| **Entrega** | fotograma de vídeo `0 … 199318` | — |
| **ATD** | `frame_id` `1 … 199319` | `video_frame = frame_id - 1` |
| **Smart Tagging** | segundos desde el inicio | `video_frame = round(segundos × 25)` |
| **Homografía** | `frame` `0 … 199318` | ya es la fila de la entrega |

No hay ningún desfase que descubrir: todo está en la línea de tiempo del vídeo
táctico. **El `-1` del ATD es el único error clásico.**

---

# 3. Cómo se cargan los datos

## Tracking, forma rápida (kloppy)

```python
from kloppy import metrica
from games import asset_paths

paths = asset_paths()
dataset = metrica.load_tracking_epts(
    meta_data=paths["atd_meta"], raw_data=paths["atd_raw"])
df = dataset.to_df()   # frame_id, ball_x, ball_y, 1_x, 1_y, 1_s, ...
```

## Tracking, sin dependencias

`starter/baseline.py` lo lee a mano y es el mejor ejemplo del formato: cada línea
es `<frame>:<jugador>;<jugador>;…:<balón>`, y cada grupo es `x,y,velocidad` con
`NaN` cuando falta la observación.

## Smart Tagging

```python
from kloppy import sportscode
events = sportscode.load(asset_paths()["smart_tagging"]).to_df()
```

Ojo: las columnas de etiquetas van en **Title Case** (`Team`, no `team`), los
códigos en MAYÚSCULAS y los valores en Title Case. Si no igualas mayúsculas y
minúsculas, obtendrás tablas vacías sin ningún error.

## Coordenadas

Vienen **normalizadas 0–1**. Para medir distancias en metros hay que multiplicar
por el campo: **105,3 × 68,0**. Si no se hace, las distancias se estiran en el eje
corto y se elige mal cerca de las bandas.

---

# 4. Las herramientas que ya existen en el repo

## `starter/baseline.py`

El suelo a batir. Genera cuatro CSVs y tarda un par de minutos (lee 199.319
líneas). Solo necesita numpy.

```bash
python starter/baseline.py
```

## `evaluation/run_evaluation.py`

**El scorer oficial.** El mismo archivo que decide la competición.

```bash
python evaluation/run_evaluation.py              # contra la ventana pública
python evaluation/run_evaluation.py --bootstrap 1000
```

Puntúa **todos** los `*.csv` que haya en `evaluation/results/`. Eso tiene una
consecuencia práctica: **si dejas un CSV de ayer ahí, el marcador te miente**.

## `evaluation/validate_submission.py`

Ejecuta **exactamente** las comprobaciones del scorer oficial. Tarda dos segundos
y es la forma más común de perder una buena entrega.

```bash
python evaluation/validate_submission.py mi_entrega.csv
```

---

# 5. `make score` no existe (todavía)

Conviene decirlo claro: **`make score` no está en el repositorio**. Es un atajo
que el plan propone construir, porque a mano hay que encadenar tres comandos y es
fácil olvidar uno:

```
python -m mtah.submit --out evaluation/results/equipo.csv   # generar
python evaluation/validate_submission.py evaluation/results/equipo.csv
python evaluation/run_evaluation.py
```

## Pero conviene separarlo en dos

Si se puntúa contra el ground truth en cada commit, el equipo acaba **eligiendo
hiperparámetros contra la ventana pública**: prohibido, y demostradamente
engañoso. Por eso:

| Comando | Cuándo | ¿Usa ground truth? | Qué comprueba |
|---|---|---|---|
| `make smoke` | **Cada commit** | **No** | El CSV se genera, valida, es determinista y el número de posesiones es sensato |
| `make score` | **Solo en las puertas del plan** | Sí | Cuánto valemos de verdad |

`make smoke` es la pieza importante: comprueba corrección **sin** poder tunear
nada. Debería verificar:

1. Existen 199.319 filas y `frame` va de 0 a 199318.
2. `label` solo contiene `H`, `A`, `D`, `X` y no hay celdas vacías.
3. Ejecutar dos veces produce el mismo fichero (determinismo).
4. El número de posesiones está en un orden de magnitud razonable (decenas, no
   cientos).
5. La duración media de las posesiones no es absurda (la referencia pública son
   14,2 s).

---

# 6. Formato de entrega y errores fatales

| Requisito | Por qué es estricto |
|---|---|
| Cabecera exactamente `frame,label` | Cualquier otra cosa se rechaza |
| Exactamente **199.319** filas de datos | Una por fotograma del vídeo táctico |
| `frame` de 0 a 199318, **en orden** | Convierte un desfase de uno en un error duro en vez de una nota silenciosamente mala |
| `label` ∈ {`H`, `A`, `D`, `X`} | Se recorta y se pasa a mayúsculas antes de comprobar |
| Sin celdas vacías | Un campo vacío es un error; para abstenerte se escribe `X` |
| UTF-8, sin columna de índice | Cualquier cosa que `pandas.read_csv` parsee |

Un fichero que falla la validación **se descarta** con una línea de motivo: cero
puntos, independientemente de lo bueno que sea el algoritmo.

---

# 7. Entorno de trabajo

| Cosa | Valor |
|---|---|
| Python del sistema | 3.14 — **demasiado nuevo** para este repo |
| Python recomendado | **3.11**, con `uv venv --python 3.11` |
| Dependencias del scorer | numpy y pandas, nada más |
| Dependencias de los notebooks | kloppy y matplotlib |
| CPU | 12 núcleos, sobra |
| GPU | **No hace falta** |
| RAM | El ATD son 196 MB de texto: cabe entero |

---

# 8. Los tres entregables y dónde van

| Entregable | Formato | Nota |
|---|---|---|
| Fichero de posesión | `<equipo>.csv` (~1,9 MB) | Es lo que se puntúa |
| Repositorio público | URL | **Sin los datos dentro**; no son redistribuibles |
| Documento técnico | `.pdf` o `.md` | 3-4 páginas |

Se suben por un mismo formulario de Google. **Una respuesta por equipo**, editable
hasta la fecha límite. Conviene descubrir el día 20 que hace falta cuenta de
Google, no el día 16.

Fecha límite: **17 de noviembre de 2026**. No hay leaderboard en vivo ni
feedback: la ventana pública y el validador son la única señal antes del cierre.
