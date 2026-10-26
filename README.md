# Global Talent Pre-State

Documentación de trabajo del equipo **Ball Tactics** para la Fase 1 de
[More Than A Hack 2027](https://github.com/metrica-sports/more-than-a-hack-2027),
el reto de posesión del balón organizado por Metrica Sports, FC Barcelona y
Mobile World Capital Barcelona.

> **Objetivo:** decir, para cada fotograma de un partido, qué equipo tiene el
> balón (`H` / `A`), si el balón está muerto (`D`) o abstenerse (`X`).
>
> **Métrica que decide el ranking:** `spellF1` (posesiones completas recuperadas).
> **Suelo a batir:** `spellF1` 0,2433.

Este repositorio es la documentación **previa al desarrollo**: contexto, conceptos
y reparto de trabajo. No contiene código ni datos del partido.

---

## Documentos

| Documento | Contenido | Formato |
|---|---|---|
| [01 · Qué hay que hacer](docs/01-que-hay-que-hacer.md) | La competición explicada desde cero: qué te dan, qué entregas, cómo te puntúan | `.md` · `.docx` |
| [02 · Conceptos](docs/02-conceptos.md) | Los términos técnicos, con las cifras que los hacen verificables | `.md` · `.docx` |
| [03 · Datos y herramientas](docs/03-datos-y-herramientas.md) | Qué hay en la carpeta de datos, qué programas ya existen, formato de entrega | `.md` · `.docx` |
| [04 · Plan de equipo](docs/04-plan-equipo.md) | Roles, contrato de datos, calendario con puertas, riesgos | `.md` · `.docx` |

Los `.docx` están en `docs/` junto a los `.md`, con el mismo nombre.

**Si solo vas a leer una cosa:** empieza por el 01.

---

## Las cinco cifras que hay que saberse

| Cifra | Qué es |
|---|---|
| **199.319** | Filas del fichero de entrega (una por fotograma) |
| **158.857** | Fotogramas que realmente se puntúan |
| **41 %** | Fotogramas calificados en los que **falta el balón** |
| **17** | Posesiones reales en los 5 minutos públicos |
| **0,2433** | `spellF1` del suelo publicado. Hay que superarlo |

---

## La idea en una frase

No se puntúan fotogramas: se puntúan **posesiones completas**. El fichero tiene
199.319 líneas, pero la nota depende de unas decenas de trozos.

---

## Fechas

| Hito | Fecha |
|---|---|
| Inicio | 25 de octubre de 2026 |
| Puerta G1 (baseline reproducido) | 1 de noviembre |
| Puerta G2 (primera versión que bate el suelo) | 8 de noviembre |
| Puerta G3 (reproducibilidad y documentación) | 15 de noviembre |
| **Entrega** | **17 de noviembre de 2026** |

---

## Reglas que no se pueden romper

1. Las etiquetas las produce **un algoritmo**. Nada de anotar a mano.
2. **No se puede usar** el ground truth privado ni datos humanos sobre este partido.
3. Los 5 minutos públicos sirven para **comprobar** el pipeline, **no** para entrenar
   ni para elegir hiperparámetros.
4. **Los datos del partido no se suben a este repositorio**: no son redistribuibles.
