# Práctica 2 — Ejercicio 2

> **Enunciado.** Un sistema de control cuenta con **4 procesos** que realizan chequeos en forma colaborativa. Reciben el **historial de fallos** del día anterior (de tamaño **N**). De cada fallo se conoce su **ID** y su **nivel de gravedad** (0=bajo, 1=intermedio, 2=alto, 3=crítico).
> **(a)** Imprimir en pantalla los ID de todos los errores críticos (no importa el orden).
> **(b)** Calcular la cantidad de fallos por nivel de gravedad, dejando los resultados en un **vector global**.
> **(c)** Ídem (b) pero **cada proceso debe ocuparse de contar los fallos de un nivel de gravedad determinado**.

---

## 0. Datos y andamiaje común

```
fallo historial[N];      // cada elemento tiene .ID y .gravedad
                         // COMPARTIDO pero de SÓLO LECTURA
```

**`historial` no lleva protección.** Es compartido, pero nadie lo modifica: los 4 procesos pueden leerlo simultáneamente sin interferencia posible. Ponerle un mutex serializa la lectura de todo el arreglo y arruina el ejercicio — es el error grave de este enunciado.

### El reparto por bloques (lo usan (a) y (b))

Cada proceso se ocupa de un tramo contiguo del arreglo:

```
int desde = id * (N/4);
int hasta = desde + (N/4) - 1;
```

| id | desde | hasta |
|---|---|---|
| 0 | 0 | N/4 − 1 |
| 1 | N/4 | 2N/4 − 1 |
| 2 | 2N/4 | 3N/4 − 1 |
| 3 | 3N/4 | N − 1 |

**Precondición obligatoria: `N mod 4 == 0`.**

Si no se cumple, la división entera se come el resto y **los últimos elementos no los recorre nadie**. Con `N = 10`: `N/4 = 2`, los bloques cubren los índices 0 a 7, y **los fallos 8 y 9 quedan sin contar**. No da "desparejo": da **mal**.

*(Si querés evitar la precondición, el reparto intercalado —`for j = id to N-1 step 4`— cubre cualquier N.)*

---

## 1. Ítem (a) — imprimir los IDs de los críticos

### Dónde está la sincronización

No hay ningún acumulador, así que es fácil pensar que no hace falta nada. Pero:

> **La pantalla es un recurso compartido.**

Nadie lo dice en el enunciado, y ese es justamente el punto del ítem.

### Qué pasa si no la protegés

**No es un problema de orden, es de mezcla.** Imprimir no es atómico: es una secuencia de escrituras de caracteres. Si un proceso imprime `1024` y otro `3387` al mismo tiempo, no obtenés `1024 3387` ni `3387 1024`. Podés obtener:

```
13038247
1032847
103 3824 7
```

Basura. No dos resultados en distinto orden: **un resultado ilegible**.

> Cuidado con leer el enunciado al revés: *"no importa el orden"* es un **permiso** —no hace falta que ordenes la salida— **no** una afirmación de que no hace falta proteger.

> **En criollo:** sin mutex no salen desordenados, salen **entreverados**.

### Solución

```
// PRECONDICIONES: N mod 4 == 0;  historial cargado antes de crear los procesos

fallo historial[N];        // compartido, SÓLO LECTURA -> sin protección
sem impresion = 1;         // la pantalla es un recurso compartido

Process Chequeador[id: 0..3]
{ int desde = id * (N/4);
  int hasta = desde + (N/4) - 1;
  for j = desde to hasta
     { if (historial[j].gravedad == 3)
          { P(impresion);
            mostrar(historial[j].ID);
            V(impresion);
          }
     }
}
```

Fijate que el `P`/`V` está **adentro del `if`**: sólo se entra a la sección crítica cuando efectivamente hay algo que imprimir. Sin las llaves, se entra en cada iteración.

### La variante: juntar y hacer una sola entrada

En vez de entrar a la sección crítica **una vez por cada crítico**, se pueden juntar los IDs en una lista **local** e imprimirlos todos de una:

```
Process Chequeador[id: 0..3]
{ lista locales;                              // LOCAL — nadie más la ve
  for j = desde to hasta
     if (historial[j].gravedad == 3)
        push(locales, historial[j].ID);       // sin semáforo: es local

  P(impresion);
  for each x in locales:  mostrar(x);         // UNA sola SC por proceso
  V(impresion);
}
```

Secciones críticas: **4** en total, contra una por cada fallo crítico.

**Pero acá es un trade-off genuino, no una mejora libre:**

| | A favor | En contra |
|---|---|---|
| Juntar y volcar | Menos entradas a la SC, menos contención | Memoria proporcional a los críticos; un proceso **acapara** la pantalla con su tanda entera; se pierde el goteo incremental si alguien mira la consola en vivo |

**Las dos versiones son correctas.** La primera es la esperada; mencionar la segunda en una línea de justificación muestra que evaluaste el criterio en vez de aplicarlo de memoria.

---

## 2. Ítem (b) — contar por nivel de gravedad

### El error que se penaliza

```
Process Chequeador[id: 0..3]
{ for j = desde to hasta
     { P(acumulador);                              ← la puerta ADENTRO del for
       gravedades[historial[j].gravedad]++;
       V(acumulador);
     }
}
```

Es **funcionalmente correcto** — el resultado da bien — pero anula por completo la concurrencia.

**La analogía.** Cuatro personas cuentan votos. Hay **una sola planilla compartida** y sólo una puede tocarla a la vez.

- **Así:** cada una agarra **una** boleta, camina hasta la planilla, espera si hay alguien, anota, se va, agarra la siguiente, vuelve a hacer la cola… Con 1000 boletas son **1000 viajes**, y mientras una anota **las otras tres están detenidas**.
- **Como corresponde:** cada una cuenta su pila **en su propio papel**, sin tocar la planilla. Recién al final hace **un viaje** y suma sus totales. **4 viajes**, y las cuatro contaron **al mismo tiempo**.

| | Versión con la SC adentro del loop | Con acumulador local |
|---|---|---|
| Veces que se entra a la SC | **N** | **4** |
| Procesos trabajando a la vez | **1** (los otros 3 bloqueados) | **4** |
| Tiempo total | Como si hubiera **un solo** proceso | La cuarta parte |

Peor aún: acá **todo fallo tiene una gravedad**, así que no hay ningún `if` que filtre — se entra a la sección crítica en **cada elemento**, y hasta la lectura `historial[j].gravedad` queda adentro. O sea que **todo** el trabajo útil está serializado. La versión termina siendo **más lenta que hacerlo con un solo proceso**, porque encima paga N pares `P`/`V`.

### Aclaración importante: esto NO es busy waiting

Son dos problemas distintos y conviene no mezclarlos:

| | Busy waiting | Bloqueado en un `P` |
|---|---|---|
| Qué hace el proceso | Gira en un loop chequeando una condición | Está **suspendido** |
| ¿Consume CPU? | **Sí**, la desperdicia | **No** |
| Quién lo reactiva | Él mismo, al ver la condición | El `V` de **otro** proceso |
| ¿La práctica lo prohíbe? | **Sí** | No — es lo normal |

La versión de arriba **cumple** la restricción de la práctica: no hay busy waiting en ningún lado. De lo que es culpable es de **serialización**:

- **Busy waiting** → desperdicia **CPU** del que espera.
- **Serialización** → desperdicia **paralelismo**: el trabajo se hace de a uno aunque tengas cuatro procesos.

### Solución

```
// PRECONDICIONES: N mod 4 == 0;  gravedades inicializado en 0 antes de crear los procesos

fallo historial[N];                   // compartido, sólo lectura
int gravedades[0:3] = ([4] 0);        // compartido, escrito por los 4
sem acumulador = 1;

Process Chequeador[id: 0..3]
{ int parcial[0:3] = ([4] 0);          // LOCAL — cuatro acumuladores propios
  int desde = id * (N/4);
  int hasta = desde + (N/4) - 1;

  for j = desde to hasta
     parcial[historial[j].gravedad]++;  // local -> SIN semáforo

  P(acumulador);                        // UNA sola SC por proceso
  for k = 0 to 3
     gravedades[k] = gravedades[k] + parcial[k];
  V(acumulador);
}
```

`parcial` es local porque está declarado **adentro** del proceso: cada uno de los 4 tiene el suyo, así que incrementarlo no necesita protección.

> **La regla, en una línea:** el semáforo protege lo compartido. Si trabajás en algo tuyo, no hace falta — así que hacé todo lo que puedas en algo tuyo, y tocá lo compartido lo menos posible.

**No olvidar inicializar `gravedades` en 0.** Si no, las sumas arrancan de basura. Va como precondición explícita.

### ¿Un mutex o cuatro (uno por posición)?

La respuesta **depende del otro diseño**:

| Con la SC adentro del loop | Con acumulador local |
|---|---|
| Cuatro mutex **ayudarían mucho**: un proceso contando graves no bloquearía a otro contando leves | **Uno alcanza y sobra**: el merge son 4 sumas, ejecutadas 4 veces en todo el programa |

Los cuatro mutex son un parche para un diseño que ya está mal. Si arreglás el diseño, el parche deja de hacer falta.

> **Criterio:** antes de afinar la granularidad del candado, preguntate si podés **reducir las veces que entrás**. Casi siempre rinde más.

---

## 3. Ítem (c) — cada proceso cuenta un nivel

Hay 4 niveles de gravedad y 4 procesos: el enunciado los hizo coincidir a propósito. El proceso 0 cuenta los leves, el 1 los intermedios, el 2 los altos, el 3 los críticos.

### Cambia el criterio de reparto

```
índice:    0  1  2  3  4  5  6  7  8  9 10 11
gravedad: [2, 0, 3, 1, 0, 2, 3, 3, 1, 0, 2, 1]
```

**En (b)** se corta **el arreglo** en 4 pedazos; cada proceso mira su pedazo y cuenta **todos** los niveles que hay ahí:

```
           P0          P1          P2          P3
        [2, 0, 3] [ 1, 0, 2 ] [ 3, 3, 1 ] [ 0, 2, 1 ]
```

**En (c)** nadie corta el arreglo. **Los cuatro leen el arreglo entero**, pero cada uno cuenta **sólo su nivel**:

```
P0 recorre los 12 y cuenta los 0  →  3
P1 recorre los 12 y cuenta los 1  →  3
P2 recorre los 12 y cuenta los 2  →  3
P3 recorre los 12 y cuenta los 3  →  3
```

### El error: mezclar los dos repartos

```
Process Chequeador[id: 0..3]
{ int desde = id * (N/4);             ← corte del arreglo
  int hasta = desde + (N/4) - 1;
  for j = desde to hasta
     if (historial[j].gravedad == id)  ← Y ADEMÁS filtro por nivel
        total++;
}
```

Con las dos cosas juntas **se pierden fallos**:

| Proceso | Su pedazo | Busca gravedad | Encuentra |
|---|---|---|---|
| P0 | `[2, 0, 3]` | 0 | **1** |
| P1 | `[1, 0, 2]` | 1 | **1** |
| P2 | `[3, 3, 1]` | 2 | **0** |
| P3 | `[0, 2, 1]` | 3 | **0** |

Da `[1, 1, 0, 0]` cuando lo correcto es `[3, 3, 3, 3]`.

El fallo leve del índice 4 está en el pedazo de P1, pero P1 sólo cuenta intermedios; y P0, que es quien cuenta leves, nunca mira ese pedazo. **Ese fallo no lo cuenta nadie.**

> El reparto por pedazos y el reparto por nivel son **dos formas distintas** de dividir el trabajo. Elegís una. Si aplicás las dos, quedan huecos.

### Solución

```
// PRECONDICIONES: historial cargado antes de crear los procesos
//                 (acá NO hace falta N mod 4 == 0: nadie corta el arreglo)

fallo historial[N];               // compartido, sólo lectura
int gravedades[0:3];              // compartido, pero cada casillero tiene un solo dueño

Process Chequeador[id: 0..3]
{ int total = 0;                  // LOCAL
  for j = 0 to N-1
     { if (historial[j].gravedad == id)
          total++;
     }
  gravedades[id] = total;
}
```

### La sorpresa: no hay semáforos

Ninguno. Variable por variable:

| Variable | ¿Compartida? | ¿Quién la escribe? | ¿Protección? |
|---|---|---|---|
| `historial` | sí | **nadie** — sólo se lee | No |
| `total` | no, es local | el proceso dueño | No |
| `gravedades[id]` | sí | **sólo el proceso `id`** | **No** |

La clave es la última fila: el proceso 2 escribe **únicamente** `gravedades[2]`. Ningún otro toca esa posición. Aunque el vector sea global, **cada casillero tiene un solo dueño**, así que no hay nada que proteger.

> **La lección del ítem:** cambiar **cómo repartís el trabajo** puede hacer que la sincronización **desaparezca**. No siempre se puede, pero cuando se puede vale más que cualquier candado inteligente.

### Detalle: `=` y no `+=`

```
gravedades[id] = total;              // correcto
gravedades[id] = gravedades[id] + total;   // incorrecto acá
```

En (b) hacía falta sumar porque cada proceso aportaba un **parcial** y había que juntar cuatro. En (c) el proceso `id` calcula **el total completo** de su nivel él solo: no hay nada a qué sumarle.

> Si escribís `+=` estás diciendo *"esto es un pedazo"*. Si escribís `=` estás diciendo *"esto es el resultado final"*. Cuál corresponde lo define el reparto.

### ¿(c) es mejor que (b)?

No necesariamente, y conviene poder justificarlo:

| | (b) | (c) |
|---|---|---|
| Elementos que lee **cada** proceso | N/4 | **N** |
| Lecturas totales | N | **4N** |
| Sincronización | mutex + merge | **ninguna** |
| Velocidad real | **~4× más rápido** | más lento |
| Precondición | `N mod 4 == 0` | ninguna |

En (c) los cuatro leen todo el arreglo, o sea **cuatro veces** el trabajo de lectura. Se gana en simplicidad —cero sincronización, imposible equivocarse— y se paga en tiempo.

---

## 4. Chequeo de sanidad

- Sumar `gravedades[0] + gravedades[1] + gravedades[2] + gravedades[3]` tiene que dar **exactamente N** en (b) y en (c). Si da menos, hay elementos que no mira nadie (típicamente la precondición `N mod 4 == 0` sin cumplir). Si da más, hay doble conteo.
- En (a), la cantidad de líneas impresas tiene que ser igual a `gravedades[3]` de (b).

---

## 5. Los tres chequeos del corrector

1. **¿Dejó `historial` sin protección?** Es compartido pero de sólo lectura. Un mutex ahí serializa todo y arruina la solución.
2. **¿Acumuló en variables locales y entró a la SC una sola vez por proceso?** Entrar por elemento es funcionalmente correcto pero anula la concurrencia.
3. **¿Se dio cuenta de que (c) no necesita semáforos?** Y en (a), ¿vio que la pantalla es un recurso compartido?
