# Contexto para agentes de IA

Este repositorio es la documentación de preparación del equipo **Ball Tactics**
para la Fase 1 de *More Than A Hack 2027* (Metrica Sports + FC Barcelona + MWC).
Lee esto antes de tocar nada; después, el documento que corresponda en `docs/`.

---

## La tarea, en dos líneas

Dado un partido completo (vídeo + tracking automático de jugadores y balón +
etiquetado táctico + homografía), producir **un CSV de 199.319 filas**
(`frame,label`) que diga, para cada fotograma, quién tiene el balón: `H` (local),
`A` (visitante), `D` (balón muerto) o `X` (abstención).

No hay entrenamiento con datos de este partido: **está prohibido anotar a mano** y
prohibido usar el ground truth privado. El público (5 minutos) solo sirve para
comprobar que el pipeline es correcto.

---

## Los cinco datos que condicionan cualquier decisión

| Dato | Valor | Consecuencia |
|---|---|---|
| Fotogramas totales | 199.319 (25 fps) | Un segundo = 25 filas |
| Fotogramas calificados | 158.857 | El 25 % del fichero no puntúa |
| Balón ausente | 41 % de los calificados | No se puede asumir que el balón existe |
| Posesiones en 5 min públicos | 17 (+4 parones) | La muestra es demasiado pequeña para elegir hiperparámetros |
| Suelo a batir | `spellF1` 0,2433 | `baseline_smooth50` del repo oficial |

---

## La métrica, y por qué se falla

Se rankea por **`spellF1`**: empareja posesiones completas por solapamiento
temporal, a 5 umbrales. `macroF1` (por fotograma) se reporta pero no decide.

Tres formas de arruinar `spellF1` sin que `macroF1` se entere:

1. **Emitir `D` de más** → parte posesiones en dos (`H D H` son dos posesiones).
2. **Inventar posesiones cortas** → el baseline crudo produce 131 donde hay 17.
3. **Confundir `D` con "he perdido el balón"** → son cosas distintas: un fuera de
   juego es `D` con el balón visible; una oclusión de 2 s no es `D`.

Banda de ruido: **0,04** en `spellF1` (0,01 en `macroF1`). Por debajo de eso, no
es una mejora.

---

## Lo que NO hay que hacer

- **No** usar vídeo, crops ni pose: el ground truth se generó desde el tracking,
  no desde píxeles.
- **No** montar tracking propio ni re-ID: el reto lo excluye.
- **No** sincronizar el broadcast: no está alineado y no aporta a esta etiqueta.
- **No** poner un mínimo de posesión de varios segundos: hay posesiones reales de
  0,8 s.
- **No** tunear contra los 5 minutos públicos.

---

## Arquitectura prevista

```
ATD crudo
  → frames.parquet        (balón crudo + imputado, distancias, confianza)
  → dead_segments.parquet (intervalos D con motivo)
  → frame_attr.parquet    (H/A + confianza por fotograma)
  → spells.parquet        (posesiones: start, end, label, confidence)
  → <equipo>.csv          (199.319 filas)
```

Cinco roles, documentados en `docs/04-plan-equipo.md`:

| Rol | Responsabilidad |
|---|---|
| P1 | Datos y artefacto canónico |
| P2 | Balón muerto (`D`) |
| P3 | Atribución de posesión |
| P4 | Segmentación temporal (quien mueve el número) |
| P5 | Medición, reproducibilidad y entrega |

---

## Documentos

| Documento | Cuándo leerlo |
|---|---|
| `docs/01-que-hay-que-hacer.md` | Siempre, primero |
| `docs/02-conceptos.md` | Antes de tocar datos o métricas |
| `docs/03-datos-y-herramientas.md` | Antes de escribir código |
| `docs/04-plan-equipo.md` | Antes de cambiar el reparto o el calendario |

---

## Repositorio oficial del reto

`metrica-sports/more-than-a-hack-2027`. La especificación real de la métrica es
`evaluation/possession_metrics.py`: si la documentación y ese archivo discrepan,
manda el archivo.
