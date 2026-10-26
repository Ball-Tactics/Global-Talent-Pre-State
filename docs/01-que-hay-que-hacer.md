---
title: "More Than A Hack 2027 — Fase 1"
subtitle: "Qué hay que hacer en la competición"
date: "25 de octubre de 2026"
---

# 1. La frase que lo resume todo

> Te dan un partido de fútbol grabado y el tracking automático de los 22 jugadores
> y del balón. Tienes que entregar **un fichero que diga, para cada instante del
> partido, qué equipo tiene el balón**.

Eso es todo. Un fichero. No hay presentación, no hay jurado, no hay demo en vivo.

---

# 2. Qué te dan

Un partido: **FC Barcelona vs Athletic Club**, LaLiga 2024-25.

| Qué | En qué consiste |
|---|---|
| **Vídeo** | 2 h 13 min, en dos cámaras (táctica y broadcast) |
| **Tracking** | 25 veces por segundo: dónde está cada jugador y el balón, en coordenadas del campo |
| **Etiquetado táctico** | 638 fases detectadas automáticamente: córners, saques, goles… |
| **Homografía** | Cómo pasar de coordenadas del campo a píxeles de la imagen |

Un detalle que lo condiciona todo: **el tracking está sin corregir**. Es lo que un
sistema automático detectó, con sus fallos. El balón, concretamente, **no aparece
en el 41 % de los instantes que se puntúan**.

---

# 3. Qué tienes que entregar

Un CSV de 199.319 líneas, una por fotograma del vídeo:

```
frame,label
11185,A
11186,A
11187,A
...
18240,D
...
18684,H
```

Cuatro etiquetas posibles:

| Letra | Significa |
|---|---|
| `H` | Tiene el balón el Barcelona (local) y el juego está vivo |
| `A` | Tiene el balón el Athletic (visitante) y el juego está vivo |
| `D` | El balón **no está en juego** |
| `X` | No contesto |

Y ya está. Ese fichero es la competición.

---

# 4. Qué significa "tener el balón" aquí

No es "quién lo tocó último". Es **posesión controlada**:

- Si un jugador lo controla medio segundo y lo pierde, **no** cambia la posesión.
- Un despeje que no se controla **no** cambia la posesión.
- Si el balón está suelto o disputado, **se queda con el equipo que lo tenía**.
- Si hay un saque, un córner, un fuera de juego o el balón sale, es `D`.

Y la trampa número uno:

> `D` **no** significa "no veo el balón". Significa "el juego está parado".

Un fuera de juego es `D` con el balón perfectamente visible. Una oclusión de dos
segundos en mitad de una jugada **no** es `D`: sigue siendo `H` o `A`.

---

# 5. Cómo te puntúan

Hay dos notas, pero **solo una decide el ranking**.

## `macroF1`: acierto por fotograma

De cada 1.000 fotogramas, cuántos etiquetaste bien. Suena razonable y no sirve
para nada: se puede acertar el 70 % de los fotogramas y describir el partido como
basura.

## `spellF1`: posesiones recuperadas ← **esto es lo que cuenta**

Te lo explico con la verdad real de la ventana pública (los 5 minutos que sí
conocemos), que son 300 segundos y **17 posesiones más 4 parones**:

| # | Etiqueta | Inicio (s) | Fin (s) | Duración |
|---|---|---|---|---|
| 1 | `A` | 0,0 | 6,6 | **6,6 s** |
| 2 | `H` | 6,6 | 14,7 | **8,1 s** |
| 3 | `A` | 14,7 | 17,6 | **2,8 s** |
| 4 | `H` | 17,6 | 59,2 | **41,6 s** |
| 5 | `A` | 59,2 | 66,6 | **7,4 s** |
| — | `D` | 66,6 | 72,2 | 5,5 s |
| 6 | `H` | 72,2 | 100,8 | **28,6 s** |
| 7 | `A` | 100,8 | 104,0 | **3,2 s** |
| 8 | `H` | 104,0 | 146,4 | **42,4 s** |
| 9 | `A` | 146,4 | 166,4 | **20,0 s** |
| 10 | `H` | 166,4 | 167,2 | **0,8 s** |
| 11 | `A` | 167,2 | 183,2 | **16,0 s** |
| 12 | `H` | 183,2 | 184,6 | **1,3 s** |
| 13 | `A` | 184,6 | 188,2 | **3,6 s** |
| — | `D` | 188,2 | 208,6 | 20,4 s |
| 14 | `A` | 208,6 | 244,3 | **35,7 s** |
| 15 | `H` | 244,3 | 247,8 | **3,5 s** |
| — | `D` | 247,8 | 266,1 | 18,3 s |
| 16 | `A` | 266,1 | 273,7 | **7,6 s** |
| — | `D` | 273,7 | 288,2 | 14,5 s |
| 17 | `H` | 288,2 | 300,0 | **11,8 s** |

Mira las duraciones: van de **0,8 s a 42,4 s**, un factor de 53. Por eso una
métrica por fotograma no puede representar la posesión: la posesión más larga
pesa 1.000 fotogramas y la más corta 20, aunque un entrenador cuente una y una.

**`spellF1` compara trozos, no fotogramas.** Empareja tus posesiones con las
reales por solapamiento temporal, una a una, y promedia a cinco exigencias de
solapamiento (30 %, 40 %, 50 %, 60 %, 70 %).

## El ejemplo que lo aclara

**La verdad:**

```
0s ──────── 14,7 ────── 20 ────────────── 38 ──────── 45
   │   A   │     H      │       D        │     H    │
   posesión 1            posesión 2      posesión 3
```

**Tu entrega, si vas mal:**

```
0s ── 5 ── 9 ────── 14,7 ── 20 ───── 30 ──── 38 ── 45
  │ A │ D │   A   │   D   │   H    │  ?  │  H  │
      ↑ falso D         ↑ falso D      ↑ hueco
```

Por fotograma has acertado bastante. Pero has **partido la posesión 1 en dos** y
la posesión 2 en dos. `spellF1` dirá: *"de las 3 posesiones, has recuperado
enteras 1"*.

Eso es lo que mide. No le importa un fotograma suelto; le importa si el relato
del partido es el correcto.

## Lo que se deduce

- **Poner `D` de más es el error más caro**: parte posesiones en dos.
- **Ir 1 fotograma tarde no cuesta nada** (0,04 s). Ir 1 segundo tarde baja la
  nota a 0,65.
- **No existe un mínimo de posesión seguro**: en la verdad real hay posesiones de
  0,8 s. Una regla de "mínimo 5 segundos" destruye posesiones auténticas.

---

# 6. El fichero tiene 199.319 líneas, pero se puntúan 21 trozos

De los 199.319 fotogramas, solo **158.857 están calificados** (los 106 minutos de
juego). Y de esos, lo que decide la nota son **unas decenas de posesiones**.

Esa es la paradoja del reto y el 90 % de lo que hay que entender: **el trabajo no
es etiquetar fotogramas, es reconstruir el relato del partido** (quién tenía el
balón y desde cuándo hasta cuándo).

---

# 7. Qué es difícil de verdad

Solo tres cosas:

1. **El balón falta el 41 % del tiempo**, y falta más justo en las jugadas
   confusas (oclusiones, aglomeraciones, balón por el aire).
2. **Distinguir `D` de "he perdido el balón"**: el baseline los confunde todos, y
   por eso inventa decenas de posesiones falsas.
3. **No trocear posesiones**: cada falso `D` rompe una posesión y cuesta una
   pieza entera de la nota.

Todo lo demás —vídeo, homografía, broadcast, pose, modelos de IA— es secundario o
directamente una pérdida de tiempo.

---

# 8. Qué hay que programar, en cuatro pasos

```
PASO 1   ¿En este fotograma el balón está en juego?       → sí / no
PASO 2   Si sí: ¿de quién es?                             → H / A
PASO 3   Agrupar en trozos: cada racha es una posesión    → lista de intervalos
PASO 4   Escribir el CSV                                  → 199.319 líneas
```

Eso es el trabajo entero. El paso 3 es el que decide la nota, y es justo el que
el baseline **no tiene**.

---

# 9. Los tres entregables

1. **El CSV** — lo que se puntúa.
2. **Un repositorio público** con el código que lo genera, reproducible.
3. **Un documento de 3-4 páginas** explicando cómo lo hiciste.

---

# 10. Qué NO es este reto

| No es… | Por qué |
|---|---|
| Un problema de visión por computador | El ground truth se generó desde el tracking, no desde píxeles |
| Un problema de tracking de jugadores | El tracking ya te lo dan; el reto lo excluye explícitamente |
| Un problema de detectar el balón | El balón te lo dan (aunque falte el 41 %) |
| Un problema de etiquetar fotogramas | Se puntúan posesiones completas |
| Un problema de machine learning | Sin datos etiquetados de este partido; es reconstrucción de estado |

---

# 11. Siguiente lectura

- `02-conceptos.md` — los términos técnicos, con las cifras que los hacen verificables.
- `03-datos-y-herramientas.md` — qué hay en la carpeta de datos y qué programas ya existen.
- `04-plan-equipo.md` — cómo se reparte el trabajo entre cinco personas.
