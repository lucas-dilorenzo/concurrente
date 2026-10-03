# Práctica 1 — Ejercicio 1

> **Enunciado.** Para el siguiente programa concurrente suponga que todas las variables están inicializadas en `0` antes de empezar. Indique cuál/es de las siguientes opciones son verdaderas:
> **a)** En algún caso el valor de `x` al terminar el programa es **56**.
> **b)** En algún caso el valor de `x` al terminar el programa es **22**.
> **c)** En algún caso el valor de `x` al terminar el programa es **23**.

```
                                    Estado inicial:  x = 0,  y = 0

P1::                    P2::                    P3::
  If (x == 0) then        If (x > 0) then         x := (x*3) + (x*2) + 1;
     y := 4*2;               x := x + 1;
     x := y + 2;
```

**Respuesta: las tres son VERDADERAS.** Pero (a) sale hasta secuencialmente, mientras que (b) y (c) **sólo** son alcanzables con granularidad fina. Ese es el punto del ejercicio.

---

## 1. La receta para este tipo de ejercicio

| Paso | Qué hacer |
|---|---|
| 1 | **Numerar las acciones atómicas de grano fino.** Cada lectura de una variable compartida es una acción separada; cada asignación se descompone en Load / operación / Store. |
| 2 | **Ver qué escribe cada proceso** y con qué valores posibles. |
| 3 | **Razonar hacia atrás desde el valor objetivo:** ¿quién pudo hacer la *última* escritura? ¿qué tuvo que haber leído para producir ese número? |
| 4 | **Construir la historia** y verificar que las guardas sean verdaderas *en el momento en que se evalúan*. |

El paso 3 es la clave práctica: no se enumeran las ~2.500 historias posibles, se trabaja de atrás para adelante desde el número que te dan.

---

## 2. Descomposición en acciones atómicas

| # | Acción | Proceso |
|---|---|---|
| 1 | `read x → t` (guarda `t == 0`) | P1 |
| 2 | `y := 8` | P1 |
| 3 | `read y → u` (u = 8) | P1 |
| 4 | `x := u + 2` → **escribe 10** | P1 |
| 5 | `read x → v` (guarda `v > 0`) | P2 |
| 6 | `read x → w` | P2 |
| 7 | `x := w + 1` | P2 |
| 8 | `read x → p` (para el `x*3`) | P3 |
| 9 | `read x → q` (para el `x*2`) | P3 |
| 10 | `x := 3p + 2q + 1` | P3 |

### Lo que escribe cada proceso

| Proceso | Escribe | Observación |
|---|---|---|
| P1 | **siempre 10** | `y` es sólo suya, nadie interfiere: `y=8` y luego `x := 8+2` |
| P2 | `w + 1` | `w` es lo que leyó en la acción 6 — **no necesariamente** lo mismo que en la guarda (acción 5) |
| P3 | `3p + 2q + 1` | `p` y `q` **pueden ser distintos** |

---

## 3. La observación clave: la propiedad At-Most-Once (AMO)

Una **referencia crítica** es una referencia a una variable que *otro* proceso modifica.
Una asignación `x := e` cumple AMO si:
1. `e` tiene **a lo sumo una** referencia crítica **y** `x` no es referenciada por otros procesos, **o**
2. `e` no tiene ninguna referencia crítica.

Si cumple AMO, se comporta **como si fuera atómica**. Si no, hay que descomponerla.

| Sentencia | Referencias críticas en `e` | ¿AMO? |
|---|---|---|
| `y := 4*2` | 0 (constantes) | ✅ atómica |
| `x := y + 2` | 0 (`y` no la toca nadie más) | ✅ atómica — por eso P1 siempre escribe 10 |
| `x := x + 1` (P2) | 1 (`x`), **pero `x` sí es leída por otros** | ❌ **no** atómica |
| `x := (x*3)+(x*2)+1` (P3) | **2** (`x` dos veces) | ❌ **no** atómica |

> **P3 lee `x` dos veces en la misma expresión, y entre esas dos lecturas puede pasar cualquier cosa.** Ése es el corazón del ejercicio.

**Nota sobre las guardas.** `if (x == 0)` tiene una sola referencia crítica, así que la evaluación en sí es atómica (leés un valor consistente de `x`). Pero eso **no** protege el cuerpo: entre evaluar la guarda y ejecutar el `then`, otros procesos pueden correr y cambiar `x`.

---

## 4. Análisis de cada opción

Notación de historias: como en la explicación de la cátedra, la secuencia de números de acción. La barra `|` sólo agrupa visualmente.

### a) x = 56 → **VERDADERO**

Sale con la ejecución secuencial P1 → P2 → P3:

```
1-2-3-4 | 5-6-7 | 8-9-10

  x = 0
  P1:  guarda 0 == 0 ✓  →  y = 8,  x = 10
  P2:  guarda 10 > 0 ✓  →  x = 11
  P3:  p = 11, q = 11   →  x = 3(11) + 2(11) + 1 = 33 + 22 + 1 = 56
```

No requiere granularidad fina.

### b) x = 22 → **VERDADERO**

Razonando hacia atrás: la última escritura es de P2 (`w + 1 = 22` → `w = 21`). ¿De dónde sale 21? De P3: `3p + 2q + 1 = 21` → `3p + 2q = 20` → con **`p = 0`, `q = 10`**.

O sea: **P3 tiene que leer `x` cuando vale 0, y volver a leerla cuando ya vale 10.**

```
8 | 1-2-3-4 | 9 | 10 | 5-6-7

   8:    p = 0                    (x todavía vale 0; es sólo una lectura, no la cambia)
   1-4:  P1 ve x == 0 ✓  →  x = 10
   9:    q = 10                   (segunda lectura de P3, ya con el valor nuevo)
   10:   x = 3(0) + 2(10) + 1 = 21
   5-7:  P2 ve 21 > 0 ✓  →  x = 22
```

### c) x = 23 → **VERDADERO**

Igual que (b), pero P3 hace su **segunda** lectura después de que P2 ya incrementó. La última escritura ahora es de P3: `3p + 2q + 1 = 23` → `3p + 2q = 22` → con **`p = 0`, `q = 11`**.

```
8 | 1-2-3-4 | 5-6-7 | 9 | 10

   8:    p = 0
   1-4:  P1 ve x == 0 ✓  →  x = 10
   5-7:  P2 ve 10 > 0 ✓  →  x = 11
   9:    q = 11
   10:   x = 3(0) + 2(11) + 1 = 0 + 22 + 1 = 23
```

---

## 5. Por qué importa la granularidad: el contraste

Si cada sentencia fuera atómica (es decir, si el programa estuviera escrito con `<>`), P3 leería el **mismo** valor `v` las dos veces y daría `3v + 2v + 1 = 5v + 1`. Los seis órdenes posibles:

| Orden | Traza | x final |
|---|---|---|
| P1, P2, P3 | 0 → 10 → 11 → 5(11)+1 | **56** |
| P1, P3, P2 | 0 → 10 → 5(10)+1 = 51 → 52 | **52** |
| P2, P1, P3 | guarda 0>0 ✗ → 10 → 5(10)+1 | **51** |
| P2, P3, P1 | ✗ → 5(0)+1 = 1 → guarda 1==0 ✗ | **1** |
| P3, P1, P2 | 1 → guarda ✗ → 1>0 ✓ → 2 | **2** |
| P3, P2, P1 | 1 → 2 → guarda ✗ | **2** |

Valores alcanzables con todo atómico: **{1, 2, 51, 52, 56}**.

> **22 y 23 no están.** Son alcanzables *únicamente* porque `(x*3)+(x*2)+1` no es atómica y P3 lee `x` en dos momentos distintos. El ejercicio está diseñado para que se note exactamente eso.

---

## 6. Errores típicos

| Error | Consecuencia |
|---|---|
| Evaluar `(x*3)+(x*2)+1` como si leyera **un solo** valor de `x` | Se pierden (b) y (c) → se responde que son falsas |
| Suponer que la guarda de P2 y el `x` del cuerpo leen lo mismo | Se pierden historias válidas |
| Creer que P1 puede escribir algo distinto de 10 | `y` no es compartida: `y := 4*2` y `x := y+2` son atómicas por AMO |
| Olvidar verificar la guarda **en el momento de su acción** | Se construyen historias imposibles (p. ej. P2 arrancando con `x = 0`) |

---

## 7. Resumen

| Opción | Valor | ¿Verdadera? | Requiere grano fino | Historia testigo |
|---|---|---|---|---|
| a | 56 | ✅ | No | `1-2-3-4-5-6-7-8-9-10` |
| b | 22 | ✅ | **Sí** | `8-1-2-3-4-9-10-5-6-7` |
| c | 23 | ✅ | **Sí** | `8-1-2-3-4-5-6-7-9-10` |

**La idea a llevarse:** el no determinismo no viene sólo del orden en que corren los procesos, sino de que **una sola sentencia puede partirse por la mitad**. Antes de razonar sobre un programa concurrente hay que decidir cuál es su granularidad, y la propiedad AMO es el criterio formal para saber qué se puede tratar como atómico y qué no.
