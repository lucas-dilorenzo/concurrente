# Práctica 2 — Ejercicio 3

> **Enunciado.** Un sistema operativo mantiene **5 instancias de un recurso** almacenadas en una **cola**. Además, existen **P procesos** que necesitan usar una instancia del recurso. Para eso, deben **sacar la instancia de la cola** antes de usarla. Una vez usada, la instancia debe ser **encolada nuevamente** para su reúso.

---

## 0. Lo que el enunciado está diciendo

El enunciado insiste con la cola: *"sacar de la cola"*, *"encolada nuevamente"*. Eso no es decorativo — significa que las instancias **tienen identidad**: no alcanza con saber que hay una libre, hay que saber **cuál** se agarró, porque después hay que devolver **esa misma**.

Por eso el problema tiene **dos partes separadas**, y cada una necesita su propio mecanismo:

| Necesidad | Mecanismo | Por qué |
|---|---|---|
| **Esperar** a que haya una instancia libre | `sem disponible = 5` | Contador de recursos |
| Saber **cuál** instancia me tocó | La cola + `sem mutex = 1` | La cola da la identidad; el mutex la protege |

### Por qué hace falta el contador si ya está la cola

Parece redundante —el semáforo cuenta lo mismo que el tamaño de la cola— pero no lo es, y la razón es una limitación real del mecanismo:

> **Con semáforos no podés esperar una condición arbitraria.** No existe "esperá hasta que la cola no esté vacía": sólo podés hacer `P` sobre un semáforo.

Así que `sem disponible = 5` **no es un contador de más**: es el **único mecanismo de bloqueo** disponible. La cola te da la identidad, el semáforo te da la espera. Trabajos distintos.

*(Y recordá que la práctica prohíbe consultar el valor de un semáforo, así que tampoco podrías "preguntarle" cuántos quedan.)*

---

## 1. Un semáforo en 5 ≠ cinco semáforos

Cuidado con cómo se enuncia en el ítem (a) de este tipo de ejercicios:

```
sem disponible = 5;              // UN semáforo contador, cuyo valor es 5
sem instancias[5] = ([5] 1);     // CINCO semáforos distintos, cada uno vale 1
```

Acá va el primero. El segundo modelaría "cinco recursos independientes, cada uno con su propio candado", que es otro problema.

---

## 2. Solución

```
// PRECONDICIONES: la cola contiene las 5 instancias antes de crear los procesos
//                 P > 0

cola recursos;                    // contiene las 5 instancias
sem disponible = 5;               // contador de recursos: cuántas quedan libres
sem mutex = 1;                    // protege la cola (pop y push)

Process Usuario[id: 1..P]
{ int rec;                                          // LOCAL: la instancia que me tocó
  P(disponible);                                    // espero a que haya alguna libre
  P(mutex);  pop(recursos, rec);  V(mutex);         // SC 1: la saco
  delay(tiempo);                                    // USO — afuera de toda SC
  P(mutex);  push(recursos, rec);  V(mutex);        // SC 2: la devuelvo
  V(disponible);                                    // aviso que quedó libre
}
```

**Dos secciones críticas, no una**, con el uso en el medio y afuera de ambas.

### Por qué cada pieza está donde está

| Línea | Por qué |
|---|---|
| `P(disponible)` **primero** | Si tomaras el mutex antes y no hubiera instancias, te bloquearías **con el candado en la mano** y nadie podría devolver la suya → **deadlock** |
| `pop` adentro del mutex | Dos procesos haciendo `pop` a la vez corrompen la cola, o sacan la misma instancia |
| `delay` **afuera** | Es la operación larga. Adentro, las 5 instancias valen por 1 |
| `push` adentro del mutex | Mismo motivo que el `pop` |
| `V(disponible)` **último** | Recién cuando la instancia ya está físicamente en la cola se puede anunciar que hay una libre |

> **La regla:** el candado se toma para **tocar la estructura**, no para **usar lo que sacaste**.

### Justificación (esto es lo que suma puntos)

> `rec` es local a cada proceso: como el uso ocurre fuera de la sección crítica, una variable compartida sería pisada por otro proceso durante el uso, y se devolvería una instancia equivocada. `recursos` es compartida y se modifica, así que `pop` y `push` van protegidos por `mutex`. `disponible` es un contador de recursos que provee la demora cuando las 5 instancias están en uso —no se puede esperar sobre la cola directamente— y se incrementa recién después del `push`, para que nunca se anuncie una instancia que todavía no volvió a la cola.

---

## 3. El error que se penaliza: el uso adentro del mutex

```
P(disponible);
P(mutex);
pop(recursos, rec);
delay(tiempo);          ← MAL: el uso adentro de la SC
push(recursos, rec);
V(mutex);
V(disponible);
```

**No corrompe nada** —el resultado es correcto— pero **anula por completo el paralelismo**.

### La analogía

Una biblioteca con **5 ejemplares** del mismo libro y **un solo mostrador**.

- **Versión mala:** pedís el libro en el mostrador y te quedás **leyéndolo parado ahí**. La fila no avanza: nadie puede pedir su ejemplar, aunque haya cuatro libres en el estante.
- **Versión correcta:** pedís el libro en el mostrador, **te vas a tu mesa a leerlo**, y volvés sólo para devolverlo.

### La traza que lo demuestra

Con 5 procesos:

| # | Qué pasa | `disponible` | Procesos usando un recurso |
|---|---|---|---|
| 1 | Los 5 pasan `P(disponible)` | 0 | 0 |
| 2 | Los 5 llegan a `P(mutex)`; entra uno | 0 | 0 |
| 3 | Ese hace `pop`, **usa** (largo rato), `push`, `V(mutex)` | 0 | **1** |
| 4 | Entra el segundo, mismo ciclo | 0 | **1** |

**Nunca hay más de un proceso usando un recurso.** Las 5 instancias se comportan como **una sola**.

Hay un síntoma que lo delata antes de trazarlo: si el `pop` y el `push` están en la **misma** sección crítica, la cola nunca queda realmente con menos de 5 elementos. Si el recurso nunca sale de la cola, el contador de disponibles no cuenta nada.

### La corrección que NO funciona

Tentación natural: *"muevo el `V(mutex)` antes del `push`"*.

```
delay(tiempo);
V(mutex);
push(recursos, rec);      ← ahora sin protección
```

**Peor.** Dos procesos podrían hacer `push` al mismo tiempo y corromper la cola, que es justo lo que el mutex evitaba.

El problema no es *dónde termina* la sección crítica: es que hacen falta **dos**.

---

## 4. Deadlock e inanición

**Deadlock:** imposible con el orden correcto de los `P`. Ningún proceso espera `disponible` teniendo `mutex`, y el `mutex` sólo se retiene durante operaciones que terminan siempre (`pop`, `push`).

**Con el orden invertido (`P(mutex)` antes que `P(disponible)`) sí hay deadlock:** un proceso se queda con el candado esperando una instancia, y el único que podría devolver una necesita ese mismo candado para hacer el `push`. Nadie avanza.

**Inanición:** posible. Si varios procesos esperan en `P(disponible)` o en `P(mutex)`, no está definido cuál despierta primero — la práctica aclara que **no se puede suponer que los semáforos sean FIFO**. No es un defecto de la solución: el enunciado no pide ningún orden. Si lo pidiera, habría que construirlo con una cola de espera explícita.

---

## 5. Chequeo de sanidad

- **Contá instancias en cualquier momento:** las que están en la cola más las que están en uso tienen que dar **exactamente 5**, y ninguna puede aparecer dos veces.
- Con **P ≤ 5** ningún proceso espera en `P(disponible)`: todos consiguen su instancia enseguida.
- Todo `P` tiene su `V` correspondiente en el mismo proceso. Si un camino hace `P(disponible)` y no llega al `V`, esa instancia se pierde para siempre.

---

## 6. Los tres chequeos del corrector

1. **¿El uso del recurso está fuera de la sección crítica?** Es el error central del ejercicio: adentro, las 5 instancias funcionan como 1.
2. **¿El orden de los dos `P` es `disponible` y después `mutex`?** Al revés es deadlock.
3. **¿`rec` es local?** Compartida, otro proceso la pisa durante el uso y se devuelve la instancia equivocada.
