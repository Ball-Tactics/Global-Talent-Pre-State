---
title: "More Than A Hack 2027 — Fase 1"
subtitle: "Conceptos técnicos, con las cifras que los hacen verificables"
date: "25 de octubre de 2026"
---

# Para qué sirve este documento

Casi todos los errores de diseño en este reto vienen de confundir los términos
que aparecen en la documentación oficial. Este documento los fija, uno por uno,
antes de que nadie escriba una línea de código.

Cada concepto incluye la cifra o el ejemplo que lo hace verificable. Si algo aquí
contradice a `evaluation/possession_metrics.py`, **el código tiene razón**: ese
archivo es la especificación real de la métrica.

---

# 1. Frame, fps y frames calificados

El vídeo táctico tiene **199.319 fotogramas a 25 fps** (2 h 13 min). Solo
**158.857 están calificados**: los 106 minutos de juego.

| Bloque | Frames | Califica |
|---|---|---|
| Antes del partido | 11.185 | No |
| Primera parte | 79.467 | Sí |
| Descanso | 20.156 | No |
| Segunda parte | 79.390 | Sí |
| Después del partido | 9.121 | No |

Lo que se escriba en los 40.462 fotogramas no calificados da igual: no puntúa.
**El 25 % del fichero de entrega es relleno.**

> 25 frames = 1 segundo. Es la única conversión que hay que memorizar.

---

# 2. ATD: tracking automático frente a tracking corregido

El fichero `_atd.txt` es el tracking **automático y sin corregir** de Metrica:
huecos, trayectorias fragmentadas, cambios de identidad y pérdida del balón.

El ground truth, en cambio, se generó desde el tracking **corregido**:

> *"A frame is `D` where the corrected tracking marks the ball dead, and
> otherwise takes the team the tracking names as owning it. Nothing is smoothed,
> interpolated or hand-adjusted."* — `CHALLENGE.md`

**La consecuencia es la más importante de todo el reto:** el problema no es de
percepción visual, es de **reconstruir un estado a partir de un tracking ruidoso**.
El vídeo es opcional y probablemente innecesario.

---

# 3. Track slot ≠ jugador

Hay **46 slots** de tracking para 22 jugadores. Se llaman `Track 1` … `Track 46`,
y su `ShirtNumber` es **el índice del slot, no un dorsal**.

Un slot es un trozo de trayectoria: aparece, desaparece y cambia de persona cuando
el tracker pierde y recupera a alguien.

Lo importante: **el equipo del slot sí es fiable**. El metadatos asigna cada slot
a un equipo, y eso no cambia nunca.

| Se puede | No se puede |
|---|---|
| Saber a qué equipo pertenece un slot | Saber qué jugador es |
| Atribuir equipo con seguridad | Analizar a un jugador concreto a lo largo del partido |
| Trabajar de forma colectiva | Usar dorsales o identidades |

---

# 4. El balón falta en el 41 % de los fotogramas calificados

Es el dato más informativo del dataset y el más frágil:

- Presente en el **56,9 %** de todos los fotogramas (113.488 / 199.319).
- Presente en el **59,3 %** de los calificados: falta en el **41 %**.
- Y falta **más** en los fotogramas que importan: oclusiones, aglomeraciones y
  balón por el aire.

**Aquí está el 80 % de la dificultad.** Cualquier diseño que asuma que el balón
existe siempre se rompe el segundo día. Las observaciones ausentes son `NaN`,
tanto para el balón como para los jugadores.

---

# 5. Las cuatro etiquetas

| Etiqueta | Significado |
|---|---|
| `H` | Equipo local en posesión, balón vivo |
| `A` | Equipo visitante en posesión, balón vivo |
| `D` | Balón muerto: no está en juego |
| `X` | Sin respuesta |

**No hay más estados.** No existe "disputado", ni "en transición", ni "dudoso" en
el ground truth. Cuando el balón está suelto, la posesión se queda con el último
equipo que la tuvo.

---

# 6. Posesión = racha maximal de `H` o de `A`

`D` es una interrupción, **nunca** una posesión:

| Secuencia | Posesiones |
|---|---|
| `H H H A A` | 2 |
| `H H D D H H` | **2** (no 1) |
| `H H H H` | 1 |

Poner `D` de más **parte posesiones reales en dos** y hunde la nota, aunque por
fotograma parezca razonable. Es exactamente el fallo del baseline.

## Las duraciones reales

En los 5 minutos públicos, las 17 posesiones van de **0,8 s a 42,4 s** (factor
53). La distribución no es cómoda:

| Rango | Cuántas |
|---|---|
| Menos de 2 s | 2 |
| Entre 2 y 8 s | 6 |
| Entre 8 y 20 s | 4 |
| Más de 20 s | 5 |

**Consecuencia de diseño:** no existe un "mínimo de posesión" seguro. Cualquier
regla de "mínimo 5 segundos para contar como posesión" **destruye posesiones
auténticas** (la 10 dura 0,8 s y la 12 dura 1,3 s). El mínimo, si se usa, tiene
que ser del orden de 1-2 fotogramas, no de segundos.

---

# 7. `macroF1` frente a `spellF1`

| | `macroF1` | `spellF1` |
|---|---|---|
| Qué mide | Acierto por fotograma | Posesiones enteras recuperadas |
| Cómo | Media de F1 de `H`, `A` y `D` | Emparejamiento uno a uno por solapamiento temporal, a 5 umbrales (30 %…70 %) |
| Rango real | ~0,16 – 1,0 | 0 – 1 |
| Banda de ruido | 0,01 | **0,04** |
| Decide el ranking | No | **Sí** |
| Desempata | No | `macroF1` |

El `macroF1` promedia las tres clases sin ponderar por fotogramas, para que
"todo muerto" no parezca un buen resultado: una apuesta constante saca ~0,16.

Coste de ir tarde, medido en `spellF1`:

| Retraso | `spellF1` |
|---|---|
| Perfecto | 1,0000 |
| 1 fotograma | **1,0000** (gratis) |
| 3 fotogramas | 0,9818 |
| 8 fotogramas | 0,8933 |
| 1 segundo | 0,6539 |
| Constante / aleatorio | 0,0000 |

**Interpretación:** ser impreciso en el borde de una posesión es barato; serlo en
el orden o en el número de posesiones es carísimo.

---

# 8. Qué hace `X` de verdad

`X` significa dos cosas distintas según la métrica:

- **`macroF1`**: un fallo, igual que una respuesta incorrecta.
- **`spellF1`**: **no existe**. Antes de calcular nada, el scorer rellena cada `X`
  con la última etiqueta no-`X` de esa parte del partido.

Dicho de otro modo: en la métrica que decide el ranking, `X` significa
**"repite mi respuesta anterior"**.

| Situación | Qué conviene |
|---|---|
| Oclusión **dentro** de una posesión | `X` es gratis y evita partir la posesión |
| **Cambio** de posesión | Hay que mojarse: el relleno arrastraría la etiqueta vieja |

`X` **no es una casilla de "no sé"**: es una herencia. Y en los fotogramas no
calificados (descanso, pre-partido) da igual: se puede rellenar con `X`.

---

# 9. La ventana pública no es un leaderboard

El ground truth público son **5 minutos: 7.500 fotogramas, 17 posesiones**, de
media 14,2 s. Está ahí para **comprobar que el pipeline funciona**, nada más.

Es una regla, no un consejo: **no se puede usar para entrenar ni para elegir
hiperparámetros**.

Y funciona como trampa demostrada. Con el mismo algoritmo y solo cambiando la
ventana de suavizado:

| Baseline | `spellF1` público | `spellF1` partido completo |
|---|---|---|
| `smooth50` | 0,3256 | **0,2433** ← mejor de verdad |
| `smooth125` | **0,4242** ← parece mejor | 0,2242 |

Elegir con la ventana pública elige la versión equivocada. Una sola posesión
perdida mueve unos 6 puntos ahí: 17 muestras no ordenan nada.

---

# 10. `D` (balón muerto) ≠ "no veo el balón"

Son dos cosas distintas, y confundirlas es el error de diseño más común:

| Caso | Etiqueta correcta | Balón visible |
|---|---|---|
| Fuera de juego | `D` | Sí |
| Saque de banda | `D` | Sí |
| Córner, falta, saque de puerta | `D` | Sí |
| Descanso, pre-partido | `D` (no califica) | — |
| Oclusión de 2 s en juego | `H` o `A` | **No** |
| Balón en el aire, no detectado | `H` o `A` | **No** |

El baseline escribe `D` siempre que pierde el balón. Por eso su clase `D` es floja
y por eso inventa decenas de posesiones falsas (131 donde hay 17).

**Mantener separados el estado del balón y la visibilidad del balón es el frente
de trabajo más rentable del proyecto.**

---

# 11. Smart Tagging (etiquetado táctico)

638 fases de juego anotadas **automáticamente** por Metrica: córners, saques,
build-up, transiciones, goles… con equipo y tipo.

Sirve como **evidencia** para `D` y para los reinicios, con dos trampas:

1. **Es inferido, no verdad.** Tiene falsos positivos y falsos negativos.
2. **`Team` no siempre es un equipo**: 36 instancias son `N/A`, e incluyen
   **todos los saques de banda** — justo donde más falta haría.

Los tiempos son segundos desde el primer fotograma: `frame = round(segundos × 25)`.

Y ojo: **las fases no son posesiones**. Un "Build Up" puede abarcar una posesión,
quedarse corto o cubrir dos. Copiar un intervalo como etiqueta importa sus errores
en bloque.

---

# 12. Homografía

Una matriz 3×3 por fotograma que pasa de coordenadas de campo (0–1) a píxeles. Es
lo que conecta el tracking con el vídeo.

| | Táctica | Broadcast |
|---|---|---|
| Filas | 199.319 | 164.488 |
| `valid = 1` | **100 %** | 65,8 % |

Hay que mirar `valid` antes de usar una matriz: donde vale 0, los nueve
coeficientes son `NaN`. `reprojection_error` dice cuánto se puede confiar en ella
(más bajo, mejor; en el broadcast llega a 304).

El broadcast **no está sincronizado** con el vídeo táctico y tiene cortes y
repeticiones. Mapearlo es problema de quien lo use.

---

# 13. Las reglas de anotación

Dos reglas definen la competición, no solo el trabajo:

1. **Las etiquetas las produce un algoritmo.** Anotar a mano no es una entrega, y
   **no se pueden usar etiquetas manuales de este partido** ni para entrenar ni
   para validar.
2. **Prohibido el ground truth privado** y cualquier dato producido por humanos
   sobre este partido, venga de donde venga.

Sí está permitido: cualquier método, cualquier modelo, y **datos externos de otros
partidos** (anotados por humanos o no).

---

# 14. El baseline (el suelo a batir)

No es un modelo: es *"lo más simple que no es una apuesta constante"*. Dos pasos,
sin entrenamiento, solo numpy.

**Paso 1.** En cada fotograma con balón visible, se queda el equipo del slot más
cercano. Sin balón → `D`. (Escala a metros antes de medir: las coordenadas vienen
normalizadas 0–1 y el campo no es cuadrado.)

**Paso 2.** Voto mayoritario en una ventana centrada de ±N fotogramas.

Las cuatro variantes son **el mismo algoritmo cambiando un solo número**:

| Variante | Ventana | `macroF1` | `spellF1` | Posesiones en 5 min (reales: 17) |
|---|---|---|---|---|
| `baseline_raw` | ±0 | 0,6532 | **0,0486** | 131 |
| `baseline_smooth25` | ±1 s | **0,6752** | 0,1996 | 36 |
| `baseline_smooth50` | ±2 s | 0,6724 | **0,2433** ← suelo publicado | 26 |
| `baseline_smooth125` | ±5 s | 0,6356 | 0,2242 | 16 |

Lo que **no** hace, y por tanto dónde están las aperturas:

- No tiene noción de balón muerto más allá de "perdí el balón".
- No imputa el balón: en el 41 % de los fotogramas escribe `D` por defecto, y eso
  parte posesiones reales en dos.
- No modela estructura de posesión: puede cambiar de equipo en cualquier fotograma.
- No usa Smart Tagging, ni homografía, ni vídeo.
- No tiene umbral de distancia: el más cercano gana aunque esté a 40 metros.
- En caso de empate, elige local.

Lo único que acierta de base: **el equipo sale del slot, y el slot siempre es del
mismo equipo**, así que los cambios de identidad no le afectan al equipo.

---

# 15. Chuleta final

| Pregunta | Respuesta corta |
|---|---|
| ¿Cuánto es un segundo? | 25 fotogramas |
| ¿Cuántos fotogramas puntúan? | 158.857 de 199.319 |
| ¿Qué decide el ranking? | `spellF1` |
| ¿Cuánto es ruido en `spellF1`? | 0,04 |
| ¿Qué es una posesión? | Racha maximal de `H` o `A` |
| ¿`H D H` cuántas posesiones son? | Dos |
| ¿Qué significa `X` en `spellF1`? | "Repite mi última respuesta" |
| ¿El balón está siempre? | No: falta en el 41 % de los fotogramas calificados |
| ¿`D` es "no veo el balón"? | No. Es "el juego está parado" |
| ¿Puedo usar los 5 minutos para tunear? | No. Solo para comprobar el pipeline |
| ¿Suelo a batir? | `spellF1` 0,2433 |
| ¿Puedo poner un mínimo de posesión de 5 s? | No: hay posesiones reales de 0,8 s |
| ¿Hace falta vídeo? | Muy probablemente no |
