# Concurrencia — Repaso rápido

> **La idea de toda la clase:** ejecutar varios algoritmos simples a la vez es más natural que escribir un algoritmo complejo que los ordene. El precio es el **no determinismo** — misma entrada, distinta salida. Todo lo demás (sincronización, atomicidad, semáforos) existe para pagar ese precio.

---

## 1. Las 4 distinciones que hay que tener claras

| | |
|---|---|
| **Concurrencia** vs. **paralelismo** | Concurrencia es **software** (procesos + comunicación + sincronización), no depende del hardware. Paralelismo es **hardware** (M > 1 procesadores) y busca **reducir el tiempo**. → *Todo paralelo es concurrente; no todo concurrente es paralelo.* |
| **Proceso** vs. **hilo** | Ambos tienen contador de programa y pila propios. El hilo **comparte espacio de direcciones y recursos**. → Por eso el hilo necesita mecanismos contra interferencia. |
| **Comunicación** vs. **sincronización** | Comunicación = **cómo se pasan datos** (memoria compartida / pasaje de mensajes). Sincronización = **cómo coordinan el orden**. Son ejes independientes. |
| **Exclusión mutua** vs. **por condición** | Exclusión mutua: uno solo a la vez en la sección crítica. Por condición: **demorarse** hasta que algo sea cierto. |

**M procesadores, N procesos:** M > 1 → concurrente **y** paralelo. M = 1 → concurrente pero **no** paralelo (time slicing + context switch).

---

## 2. Notación — cheat sheet

| Construcción | Qué hace |
|---|---|
| `co S1 // S2 // Sn oc` | Concurrente, **espera a que todas terminen** (fork-join). |
| `process A {...}` | Concurrente, corre **en background**, no bloquea. |
| `if B1 → S1 □ B2 → S2 fi` | Guardas. Elección **no determinística**. Si ninguna es true → **no hace nada** (no es error). |
| `do B1 → S1 □ B2 → S2 od` | Igual, pero itera **hasta que todas las guardas sean falsas**. |
| `< e >` | Evaluar `e` **atómicamente**. |
| `< await (B) S >` | Demora hasta `B`, después ejecuta `S` indivisible. |
| `skip` | No hace nada. Se usa como cuerpo de spin: `do (not B) → skip od`. |

**La diferencia que preguntan:** `co` espera, `process` no.

---

## 3. Atomicidad — la parte que se cae en el parcial

**Regla base: una línea de código NO es indivisible.**

`x = x + 1` son tres instrucciones (load, add, store) y el interleaving se mete en el medio.

### Propiedad ASV ("a lo sumo una vez")

> **A lo sumo una variable compartida, referenciada a lo sumo una vez.**
> Si se cumple → la asignación **parece atómica** y no hace falta protegerla.

*Referencia crítica* = leer una variable que **otro proceso modifica**.

`x = e` cumple ASV si:
- `e` tiene **≤ 1** referencia crítica **y** `x` no la lee nadie más, **o**
- `e` **no tiene** referencias críticas (ahí `x` sí puede ser leída por otros).

### Los tres casos canónicos

| Código | ¿ASV? | Resultado |
|---|---|---|
| `co x=x+1 // y=y+1 oc` | ✅ Sin refs. críticas | Siempre `x=1, y=1` |
| `co x=y+1 // y=y+1 oc` | ✅ 1 ref. crítica, `x` no se lee | `y=1`; `x` puede ser **1 o 2** — ambos correctos |
| `co x=y+1 // y=x+1 oc` | ❌ Ninguna cumple | Esperado `1,2` o `2,1`. Pero puede dar **`1,1` → ERROR** |

### El ejercicio tipo

```
x=0; y=4; z=2;
co  x = y+z  (1)  //  y = 3  (2)  //  z = 4  (3)  oc
```

`y=3` y `z=4` **siempre**. `x` puede ser **5, 6, 7 u 8** según el orden — y bajando a grano fino (load/add/store de la (1)) salen combinaciones que el interleaving de sentencias completas no daba.

> **Moraleja:** para razonar bien hay que bajar al grano fino. *No se puede confiar en la intuición.*

---

## 4. Los tres problemas

**Interferencia** — un proceso invalida las suposiciones de otro.
- `A1: y = 0` mientras `A2` hace `if (y≠0) z = x/y` → *check-then-act* roto.
- Dos procesos incrementando un contador compartido → se pierden incrementos.

**Deadlock** — ambos esperan que el otro libere un recurso. *La ausencia de deadlock es una propiedad necesaria.*
Las 4 condiciones: recursos reusables serialmente · adquisición incremental · no-preemption · espera cíclica.

**No determinismo** — dos ejecuciones del mismo programa no son idénticas → **debug difícil**. Es inherente, no un bug.

---

## 5. El ejemplo que resume todo

Productor/consumidor con buffer de tamaño N:

```
process Productor                              process Consumidor
{ while (true)                                 { while (true)
    Generar Elemento                               <await (cant>0); pop(buffer,e); cant-->
    <await (cant<N); push(buffer,e); cant++>       Consumir Elemento
}                                              }
```

| Qué mirar | Por qué |
|---|---|
| `await (cant<N)` / `await (cant>0)` | Sincronización **por condición** — demora si está lleno / vacío. |
| Los `< >` | Exclusión mutua — el `push` y el `cant++` son **una sola** unidad. |
| `Generar`/`Consumir` **afuera** | La sección crítica se mantiene **lo más chica posible**. Ese es el criterio de diseño de toda la materia. |

---

## 6. Frases para tener en la punta de la lengua

- La **sincronización por condición restringe las historias** posibles para asegurar el orden temporal necesario.
- Una **historia** (trace) es una ejecución con un interleaving particular. Hay muchísimas y **no todas son válidas**.
- Una acción atómica de **grano fino** la implementa el hardware; la de **grano grueso** es una secuencia que *aparenta* ser indivisible.
- `await` tiene **alto poder expresivo pero costo de implementación alto** → por eso vienen locks, semáforos y monitores.
- `< await (s>0) s=s-1 >` es literalmente la operación **P de un semáforo**.
- Un lenguaje concurrente debe proveer 3 cosas: **indicar** qué corre concurrente, **sincronización**, **comunicación**.

---

## 7. Lo que está en Teoría 1 y NO se vio en clase

Hardware (UMA/NUMA, jerarquía de memoria) · **Safety vs. liveness** · **Fairness** (incondicional / débil / fuerte; round-robin es práctica pero **no** fuertemente fair) · granularidad · inanición y overloading · `fa` (for-all) con `st`.
