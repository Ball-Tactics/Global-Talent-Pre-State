---
title: "More Than A Hack 2027 — Fase 1"
subtitle: "Plan de trabajo para un equipo de 5 personas"
date: "25 de octubre de 2026"
---

# Resumen ejecutivo

Quedan **23 días** (25 oct → 17 nov). El objetivo es entregar un CSV de 199.319
filas con etiquetas `H` / `A` / `D` / `X`, superando el suelo publicado de
**`spellF1` 0,2433**, más un repositorio reproducible y un documento técnico.

El plan se apoya en una lectura concreta del reto: **el ground truth es mecánico
del tracking corregido**, así que esto es reconstrucción de estado temporal, no
visión por computador. El trabajo se reparte por **etapas del pipeline**.

Lectura previa obligatoria: `01-que-hay-que-hacer.md` y `02-conceptos.md`.

---

# 1. La decisión que más condiciona el arranque

**¿Debe una persona hacer la base y luego empezar los demás?**

Sí, pero solo **el esqueleto**, y con una corrección importante: lo que bloquea es
**el contrato de datos**, no su implementación. Esperar a que "la base esté bien"
cuesta una semana de cinco personas.

## Qué bloquea de verdad

| Pieza | ¿Bloquea? | Por qué |
|---|---|---|
| El esquema del parquet | **Sí** | Sin él, cada uno inventa campos distintos |
| El scorer funcionando | **No** | Funciona en un clon limpio **sin datos**. Puede ir en paralelo desde el minuto uno |
| El parquet real, completo | **No** | Se trabaja con un dummy del mismo esquema |

## La regla: esqueleto en 48 horas

**Día 1**
- P1 escribe el esquema (aunque el cargador devuelva `NaN`).
- P5 monta el scorer y el validador en paralelo, sin tocar datos.
- P2, P3 y P4 escriben sus módulos con salida dummy válida.

**Día 2**
- Todos integran: dummy → CSV → `validate_submission.py` → score.
- **Objetivo: el CSV más tonto posible ya pasa por el scorer.** A partir de ahí,
  cada mejora es medible.

**Día 3 en adelante:** los cinco en paralelo, cada uno mejorando su pieza contra
el mismo contrato.

## Por qué no serializar

- Cuatro personas paradas 3-5 días de 23 es **~20 % de la capacidad del equipo**.
- El que hace la base se vuelve dependencia única: si se equivoca en el esquema,
  los otros cuatro construyen sobre arena y nadie lo detecta hasta la integración.
- "La base bien hecha" siempre tarda el doble.

## La salvaguarda

El esquema **no lo congela una persona sola**: lo propone P1 el día 1 y lo revisa
el equipo en 30 minutos (campos, tipos, unidades, qué significa `NaN`). Es el
único momento en que los cinco tenéis que estar de acuerdo.

---

# 2. Principio rector: un contrato, cinco dueños

El mayor riesgo de un equipo de cinco en tres semanas no es el algoritmo: es
pisarse. Todo el mundo escribe parquet en `cache/` con un esquema congelado, y
solo dos personas producen artefactos finales:

- **P4** es el único que produce el array de 199.319 etiquetas.
- **P5** es el único que escribe en `evaluation/results/` y el CSV de entrega.

```
ATD crudo
   │
   ├─ P1 data.py ──────────► cache/frames.parquet
   │
   ├─ P2 deadball.py ──────► cache/dead_segments.parquet
   │
   ├─ P3 attrib.py ────────► cache/frame_attr.parquet
   │
   └─ P4 segment.py ───────► cache/spells.parquet ──► labels[199319]
                                   │
                              P5 submit.py ───────► <equipo>.csv
```

Un PR por módulo. Ramas cortas. Nadie toca el fichero de otro.

---

# 3. Reparto de roles

| | Rol | Dueño de | Entregable | No hace |
|---|---|---|---|---|
| **P1** | Datos | `mtah/data.py` | `cache/frames.parquet` congelado + imputación del balón | algoritmo ni etiquetas |
| **P2** | Balón muerto | `mtah/deadball.py` | `cache/dead_segments.parquet` | decidir de quién es el balón |
| **P3** | Atribución | `mtah/attrib.py` | `cache/frame_attr.parquet` (etiqueta + confianza) | segmentar ni suavizar |
| **P4** | Segmentación | `mtah/segment.py` | `cache/spells.parquet` → array de etiquetas | datos, docs, CSV |
| **P5** | Medición y entrega | `eval/`, `Makefile`, repo, PDF | `make smoke`/`make score`, visor, reproducibilidad, documento | algoritmo |

## P1 — Datos

Es el **bloqueante del día 1**: sin el artefacto canónico, los demás trabajan a
ciegas.

- Cargar el ATD y alinear `frame_id → video_frame` (`-1`).
- Escalar a metros (105,3 × 68,0) antes de medir distancias.
- **Imputar el balón**: Kalman o velocidad constante con memoria corta, más un
  flag de confianza. No es un parche posterior: es parte del artefacto.
- Calcular distancias mínimas del balón a cada equipo, jugadores visibles y
  velocidades.
- Congelar el esquema **antes** de que esté bonito.

## P2 — Balón muerto

Decide **solo** `vivo / muerto`. Nunca decide de quién es el balón.

- Distinguir `D` real (fuera de juego, saque, córner, falta, gol, descanso) de
  "he perdido el balón", que **no** es `D`.
- Integrar Smart Tagging como evidencia, no como verdad.
- Manejar los reinicios y los saques de banda (`Team = N/A`).
- Producir intervalos con motivo y confianza, auditables.

Es el frente de **mayor retorno**: `D` es ~20 % de los fotogramas calificados y un
`D` de más parte una posesión en dos.

## P3 — Atribución

Decide de quién es el balón **en fotogramas con balón vivo**.

- Candidato por fotograma: slot más cercano, con umbral de distancia.
- Cambio de velocidad del balón como señal de toque (no solo proximidad).
- Confianza por fotograma, no solo etiqueta.
- Robustez a los huecos del balón usando el artefacto de P1.

## P4 — Segmentación

Convierte etiquetas por fotograma en **posesiones**. Es quien mueve el número.

- Histéresis y transición de equipo solo si hay evidencia de toque.
- Optimización global sobre todo el vídeo (es diferido: se puede usar el futuro).
- Reglas de `X` dentro de posesión (gratis) y en transición (hay que mojarse).
- **Cuidado con el mínimo de posesión**: en la verdad real hay posesiones de 0,8 s.
  Un mínimo en segundos destruye posesiones auténticas.
- Emitir `spells.parquet` con `start`, `end`, `label`, `confidence`.

## P5 — Medición y entrega

No es "el que hace los informes". Es **quien garantiza que el número que dice si
vais bien es real**.

El trabajo de P5, en una frase: cuando los otros cuatro terminan el pipeline, sale
un CSV… **¿es bueno?** Nadie lo sabe hasta que alguien lo mide bien.

Responsabilidades:

- `make smoke` (corrección, sin ground truth, en cada commit) y `make score`
  (puntuación real, solo en las puertas). Ver `03-datos-y-herramientas.md`.
- Test de regresión: si no se bate el suelo, el build falla.
- `validate_submission.py` en cada integración.
- Visor de diagnóstico: dibujar una posesión con balón, jugadores y etiqueta.
- Reproducibilidad desde clon limpio, en otra máquina.
- Repositorio público (sin datos) y documento técnico.
- Higiene de `evaluation/results/`: el scorer puntúa **todo** lo que haya ahí.

### Qué pasa si nadie hace esto

| Sin P5 | Coste |
|---|---|
| Dos módulos mejoran por separado, se juntan y el score **baja** sin saber cuál lo rompió | 2 días bisecando a mano |
| El CSV final tiene 164.488 filas porque alguien indexó el broadcast | **0 puntos** |
| Dos personas suben a `results/` a la vez y el marcador puntúa CSVs mezclados | Decisiones con datos falsos |
| El 15 de noviembre, clon limpio y el CSV **no sale igual** | Entregable incompleto |
| Nadie detecta que estáis optimizando contra 17 posesiones | Descubrirlo el día de los resultados |

### Cuánto tiempo

- **Semanas 1-2: ~50-60 %.** Scorer, `smoke`, visor, CI.
- **Semana 3: 100 %.** Reproducibilidad, documento, entrega, ablaciones.

**Regla mental:** P4 mueve el número; P5 decide si el número significa algo.

---

# 4. Contrato de datos

## `cache/frames.parquet` (P1)

Una fila por fotograma, 0 … 199.318.

| Campo | Tipo | Notas |
|---|---|---|
| `frame` | int32 | índice del vídeo táctico |
| `ball_seen` | bool | observación real, sin imputar |
| `ball_x`, `ball_y` | float32 | crudo, normalizado 0–1, `NaN` si falta |
| `ball_x_imp`, `ball_y_imp` | float32 | imputado |
| `ball_speed` | float32 | m/s |
| `ball_conf` | float32 | 0–1, confianza de la imputación |
| `dist_home_min`, `dist_away_min` | float32 | metros al jugador más cercano de cada equipo |
| `n_home`, `n_away` | int8 | jugadores visibles por equipo |

## `cache/dead_segments.parquet` (P2)

| Campo | Tipo | Notas |
|---|---|---|
| `start` | int32 | inclusivo |
| `end` | int32 | exclusivo |
| `reason` | string | `throw_in`, `corner`, `goal`, `halftime`, `lost_ball`, … |
| `confidence` | float32 | 0–1 |

## `cache/frame_attr.parquet` (P3)

`frame`, `code` (`H`/`A`/nada), `confidence`.

## `cache/spells.parquet` (P4)

`spell_id`, `start`, `end` (exclusivo), `label`, `confidence`.

**Ese es todo el contrato.** Con él, cinco personas trabajan sin bloquearse.

---

# 5. Calendario con puertas

## Semana 1 · 25 oct – 1 nov — Fundaciones

Nadie escribe algoritmo todavía.

| Quién | Qué |
|---|---|
| P1 | Entorno con `uv venv --python 3.11`, copia local de los 19 GB, esquema congelado, primer `frames.parquet` |
| P5 | `make smoke`, CI mínimo, **reproducir el scoreboard exacto de los cuatro baselines** |
| P4 | Esqueleto del segmentador **sobre la salida cruda del baseline** |
| P2, P3 | Leer `possession_metrics.py` y escribir sus módulos con salida dummy válida |

> **Puerta G1 — 1 de noviembre.** `make score` reproduce **0,2433** y el CSV pasa
> `validate_submission.py`. Hasta aquí no se avanza.

## Semana 2 · 2 – 8 nov — Balón muerto y atribución

- **P2:** `D` v1 — huecos de balón, Smart Tagging, saques, goles, descanso.
- **P3:** atribución v1 — umbral de distancia, cambio de velocidad del balón, confianza.
- **P4:** segmentación v1 — histéresis, mínimo de posesión muy corto, transición solo con toque.
- **P1:** imputación del balón afinada + flag de confianza.
- **P5:** visor de posesión y conteo de posesiones por versión.

> **Puerta G2 — 8 de noviembre.** Versión fea end-to-end que supere
> **0,2433 + 0,04** (el margen de ruido).

## Semana 3 · 9 – 15 nov — Robustez y cierre

- Casos raros: descanso, goles, rojas, cambios, saques, córners, balón en el aire.
- Ablaciones por pieza, para el documento técnico.
- Reproducibilidad desde clon limpio en otra máquina.
- Documento técnico y repositorio público sin datos.

> **Puerta G3 — 15 de noviembre.** Tres ejecuciones idénticas byte a byte y clon
> limpio que produce el mismo CSV.

## 16 nov — entrega. 17 nov — fecha límite (colchón de un día).

---

# 6. Reglas de convivencia

1. **Reunión diaria de 15 minutos.** Qué entregué, qué me bloquea, qué toco hoy.
2. **Ramas cortas, un PR por módulo.** Nadie toca el fichero de otro.
3. **La ventana pública solo valida corrección**, nunca elige entre dos versiones.
   Si un cambio mejora en público pero no hay argumento físico detrás, se descarta.
4. **El número lo mide P5, lo interpreta P4.** Nadie más opina con autoridad.
5. **Una sola copia local de los 19 GB.** Nada de cinco personas leyendo del drive.
6. **Nada de vídeo, crops ni pose.** El ground truth es mecánico del tracking.

---

# 7. Riesgos

| Riesgo | Probabilidad | Mitigación |
|---|---|---|
| P4 bloqueado esperando datos | Alta | Empieza sobre la salida del baseline en la semana 1 |
| P2 y P3 se solapan en "quién tiene el balón" | Alta | Frontera dura: P2 solo vivo/muerto; P3 solo quién, y solo en fotogramas vivos |
| Sobreajuste a los 5 minutos | Media | Solo P5 ejecuta el scorer; `smoke` no usa ground truth |
| Cinco personas y un solo número: discusiones | Media | Puertas G1/G2/G3 con criterio escrito de antemano |
| Entrega perdida por formato | Baja pero fatal | `validate_submission.py` en cada push |
| Descubrir el 15 de noviembre que no reproduce | Media | Clon limpio en la semana 1, no en la 3 |

---

# 8. Qué NO hacer (y por qué)

| Tentación | Por qué no |
|---|---|
| Leer el vídeo, crops o pose | El ground truth sale del tracking; el techo es bajísimo para el coste |
| Montar tracking propio o re-ID | El reto lo excluye explícitamente |
| Sincronizar el broadcast | No está alineado y aporta poco a esta etiqueta |
| Etiquetar a mano fotogramas difíciles | Prohibido por las reglas |
| Tunear con los 5 minutos | Prohibido y demostradamente engañoso |
| Pedir GPU | 196 MB de texto y 12 núcleos: sobra CPU |
| Empezar por el modelo | Sin artefacto canónico ni scorer fiable, no se puede medir |
| Poner un mínimo de posesión de varios segundos | Hay posesiones reales de 0,8 s |

---

# 9. Las primeras 48 horas

**Día 1 (26 oct), mañana**

- P1: `git clone`, `uv venv --python 3.11`, dependencias, copia local de los
  19 GB, `python starter/baseline.py`.
- P5: `python evaluation/run_evaluation.py` y apuntar el scoreboard exacto.
- P2, P3, P4: leer `CHALLENGE.md`, `EVALUATION.md` y `possession_metrics.py`. Sin
  escribir código todavía.

**Día 1, tarde**

- P1: primer `frames.parquet` (aunque sea feo) y esquema publicado en el repo.
- P5: `make smoke` reproducible.
- P4: esqueleto de `segment.py` que consuma el parquet y emita `spells.parquet`.

**Día 2**

- Todos integran su dummy end-to-end: CSV válido → `validate_submission.py` → score.
- **Objetivo del día 2: el CSV más tonto posible ya pasa por el scorer.**

---

# 10. Definición de "hecho" por rol

| Rol | Hecho cuando |
|---|---|
| P1 | `frames.parquet` se regenera con un comando y su esquema está documentado |
| P2 | Cada intervalo `D` tiene motivo y confianza, y se puede dibujar sobre el partido |
| P3 | La confianza por fotograma correlaciona con el acierto |
| P4 | El número de posesiones está en el orden correcto (decenas, no cientos) |
| P5 | Un clon limpio, con `MTAH_DATA_DIR`, genera el mismo CSV byte a byte |

---

# 11. Entregables finales

1. **`<equipo>.csv`** — 199.319 filas, `frame,label`, validado.
2. **Repositorio público** — sin datos, dependencias fijadas, un comando que
   regenera el CSV.
3. **Documento técnico** — enfoque, activos usados y descartados, datos de
   entrenamiento (ninguno de este partido), reglas de posesión implementadas,
   limitaciones y reproducción.

Los tres se suben por el mismo formulario. Una respuesta por equipo.
