# Práctica 1 — Ejercicio 4

> **Enunciado.** Resolver con SENTENCIAS AWAIT (`<>` y `<await B; S>`). Un sistema operativo mantiene **5 instancias de un recurso** almacenadas en una cola; cuando un proceso necesita usar una instancia del recurso la **saca** de la cola, la **usa**, y cuando termina de usarla la **vuelve a depositar**.

---

## 0. El punto de partida: la solución secuencial

Con un solo proceso el problema es trivial:

```
rec = pop(recursos);
usar(rec);
push(recursos, rec);
```

Esto **no se tira** — es el algoritmo completo, y sobrevive intacto. Lo concurrente son sólo tres decisiones alrededor.

---

## 1. En qué se diferencia del Ejercicio 3

Conviene marcarlo porque el instinto es copiar la solución del Productor/Consumidor, y la forma es distinta:

| | Ej. 3 — Productor/Consumidor | Ej. 4 — Pool de recursos |
|---|---|---|
| Roles | **Dos** tipos de proceso, asimétricos | **Uno** solo: todos hacen lo mismo |
| Flujo | Los datos van en una dirección | Las instancias **vuelven** — es un ciclo |
| Condiciones | **Dos**: buffer lleno y buffer vacío | **Una** sola: que haya instancia disponible |
| Qué administra el `await` | Lugares donde poner datos | **Permiso para usar algo** |

Esa última fila es el salto conceptual: acá el `await` no cuenta huecos, **administra permisos**. Es el primer ejercicio con forma de semáforo — de hecho es un semáforo general inicializado en 5, que es adonde va la materia en la unidad siguiente (§6).

---

## 2. Las tres decisiones

| Decisión | Respuesta | Por qué |
|---|---|---|
| **¿Quién espera y dónde?** | En el `pop`, con la condición *"la cola no está vacía"* | Si las 5 instancias están en uso, no hay nada que sacar |
| **¿Qué es atómico?** | `pop` y `push` | Ambos son read-modify-write sobre la cola |
| **¿Qué es local?** | `rec` | `usar(rec)` queda **fuera** del `< >`, así que la variable está expuesta |

### El detalle del punto 1: nunca se espera con el recurso en la mano

> Un proceso espera **para conseguir** la instancia, la usa, y la devuelve. Jamás se queda esperando mientras retiene una.

Esto vale escribirlo porque resuelve el análisis de deadlock de un plumazo (§7).

### El detalle del punto 2: el `push` es asimétrico respecto del `pop`

El `pop` puede tener que esperar. El `push` **no puede fallar nunca**, por el invariante:

```
instancias_en_cola + instancias_en_uso = 5      (siempre)
```

El proceso que devuelve tiene una instancia en la mano, así que en la cola hay a lo sumo 4 y siempre entra.

**Pero igual necesita `<>`:** dos procesos devolviendo al mismo tiempo corrompen la cola igual que dos sacando. Esa es exactamente la distinción de la Clase 1:

- `<await (B); S>` → acción atómica **condicional** (guarda + exclusión mutua) → el `pop`
- `<S>` → acción atómica **incondicional** (sólo exclusión mutua) → el `push`

---

## 3. Solución

```
// ---------- PRECONDICIONES ----------
//   la cola contiene las 5 instancias antes de crear los procesos
//   N > 0   (cantidad de procesos usuarios, arbitraria)
//   cada proceso devuelve toda instancia que saca (no se "queda" con ninguna)

cola recursos;      // compartida, inicializada con las 5 instancias

Process Usuario [i: 1..N]
{ int rec;                                                  // LOCAL
  while (true)
   { <await (not vacia(recursos)); rec = pop(recursos)>     // atómica CONDICIONAL
     usar(rec);                                             // AFUERA del <>
     <push(recursos, rec)>                                  // atómica INCONDICIONAL
   }
}
```

### Justificación (esto es lo que suma puntos)

> `recursos` es la única variable compartida y se accede exclusivamente dentro de acciones atómicas, de modo que no puede corromperse ni entregar la misma instancia dos veces. El `pop` es condicional porque la cola puede estar vacía; el `push` es incondicional porque, por el invariante `en_cola + en_uso = 5`, quien devuelve siempre tiene lugar. `rec` es local a cada proceso: como `usar(rec)` se ejecuta fuera de la sección crítica —para no serializar el uso y desperdiciar las otras 4 instancias—, una variable compartida sería pisada por otro proceso durante el uso.

---

## 4. Decisión de granularidad, variable por variable

| Variable | ¿Compartida? | ¿Se escribe? | ¿`<>`? | Por qué |
|---|---|---|---|---|
| `recursos` (la cola) | **sí** | **sí** | **Sí** | `pop`/`push` son read-modify-write sobre la misma estructura |
| `rec` | no (local) | sí | No | Cada proceso tiene la suya. **Si fuera compartida se rompe** (§5) |
| `i` (el id) | no (local) | no | No | Sólo distingue procesos; acá ni se usa |

---

## 5. Los tres errores que se penalizan

### 5.1 Poner `usar(rec)` adentro del `< >`

```
<await (not vacia(recursos)); rec = pop(recursos); usar(rec); push(recursos, rec)>   // MAL
```

Es **funcionalmente correcto** —no se corrompe nada— pero destruye el problema: sólo un proceso puede estar usando un recurso a la vez, así que el programa se comporta **como si hubiera una sola instancia** y las otras 4 son decorativas.

Es el mismo criterio que `produce`/`consume` fuera del `< >` en el Ejercicio 3: **la operación larga nunca adentro de la reserva**.

### 5.2 No encerrar el `pop`

`recursos = [1,2,3,4,5]`, dos procesos, `pop` sin `<>` (leer la cabeza, después avanzarla):

| # | Proceso | Acción | Resultado |
|---|---|---|---|
| 1 | P1 | lee la cabeza | obtiene la instancia **1** |
| 2 | P2 | lee la cabeza | obtiene la instancia **1** también |
| 3 | P1 | avanza la cabeza | cola = `[2,3,4,5]` |
| 4 | P2 | avanza la cabeza | cola = `[3,4,5]` |

Dos procesos usando la instancia 1 al mismo tiempo —justo lo que el mecanismo existía para impedir— y la instancia 2 desapareció sin que nadie la use. El pool se degrada solo.

### 5.3 Dejar `rec` compartida

El más sutil, porque los `<>` **están puestos y bien puestos**. `recursos = [1,2,3,4,5]`, `pop` y `push` atómicos, pero `rec` global:

| # | Proceso | Acción | `rec` | cola |
|---|---|---|---|---|
| 1 | P1 | `<rec = pop(recursos)>` | 1 | `[2,3,4,5]` |
| 2 | P2 | `<rec = pop(recursos)>` | **2** ← pisó el 1 | `[3,4,5]` |
| 3 | P1 | `usar(rec)` | 2 | usa la instancia **2** |
| 4 | P2 | `usar(rec)` | 2 | usa la instancia **2** |
| 5 | P1 | `<push(recursos, rec)>` | 2 | `[3,4,5,2]` |
| 6 | P2 | `<push(recursos, rec)>` | 2 | `[3,4,5,2,2]` |

La instancia **1 se perdió** para siempre, la **2 quedó duplicada**, y dos procesos la usaron simultáneamente.

> **En criollo:** el `<>` protege la cola, no lo que sacaste de ella. Lo que tenés en la mano es tuyo, y va en una variable tuya.

Es exactamente el mismo error que `elemento` compartida en el Ejercicio 3 (b), y aparece por la misma causa: **la operación larga está afuera del `< >`, así que la variable queda expuesta durante todo ese rato.**

---

## 6. Contador vs. cola — qué modela cada uno

Pregunta natural: ¿no alcanza con `int disponibles = 5` en vez de la cola?

```
<await (disponibles > 0); disponibles-->
usar(???)                                  // <- ¿qué le paso?
<disponibles++>
```

**No hay truco: el contador funciona.** Pero controla otra cosa.

| | `int disponibles = 5` | `cola recursos` |
|---|---|---|
| Qué controla | **cuántos** entran a la vez | **cuál** te tocó |
| Qué te da | un permiso | el objeto |
| Alcanza si | las instancias son **anónimas** e intercambiables | tenés que **nombrar** la que usás |
| Es, en realidad | un semáforo general en 5 | un pool de recursos |

Si el recurso es anónimo —5 licencias, 5 slots de ancho de banda— el contador **no sólo alcanza: es mejor**. Menos estado compartido, nada que corromper, y `usar()` no necesita argumento. Si las instancias tienen identidad —5 impresoras, 5 conexiones distintas— el contador te dice *"pasá"* pero no te dice *a cuál*.

El enunciado dice **"almacenadas en una cola"** y **"la saca / la vuelve a depositar"**: esa redacción modela identidad, así que la cola es lo que piden.

### El punto fino: no lleves las dos

> La cola hace **los dos trabajos en una sola acción atómica**: `not vacia(recursos)` es la condición **y** `pop` entrega la identidad.

Por eso agregar un contador **al lado** de la cola sería un error de diseño:

```
<await (disponibles > 0); disponibles--; rec = pop(recursos)>    // dos fuentes de verdad
```

`disponibles` y `tamaño(recursos)` tienen que coincidir siempre. Mantener dos fuentes de verdad sincronizadas es **exactamente lo que se rompió en el Ejercicio 3**, donde `cant` y el contenido real del buffer se desincronizaban. Acá no fallaría, porque van adentro del mismo `< >`, pero es estado redundante: información que ya está en la cola, duplicada.

> **Criterio general:** si podés preguntarle el estado a la estructura misma, no lleves un contador aparte.

Y si insistís con el contador pero necesitás la identidad, tenés que agregar un arreglo de flags:

```
<await (disponibles > 0); disponibles--; rec = (algún j con libre[j]); libre[rec] = false>
```

...donde ahora `disponibles` es redundante con "cuántos `libre[j]` están en true". Si lo tirás, lo que queda **es la cola otra vez**, escrita como arreglo.

### Hacia dónde va esto

Lo que aparece acá es la diferencia entre los dos mecanismos que vienen en la materia:

- **Semáforo general** = el contador. Controla *cuántos*. La versión de arriba es literalmente `sem disponibles = 5` con `P()` y `V()`.
- **Monitor / pool** = la cola. Entrega *cuál*.

Este es de los últimos ejercicios que se resuelven con `await` crudo antes de que la cátedra dé el semáforo hecho.

---

## 7. Deadlock e inanición

**Deadlock: imposible, y se demuestra en una línea.** De las 4 condiciones necesarias de Teoría 1, falla la **adquisición incremental**: ningún proceso retiene un recurso mientras espera otro. Espera **antes** de adquirir, con las manos vacías. Sin esa condición no puede haber deadlock, sin importar cuántos procesos haya.

**Inanición: sí es posible.** Con muchos procesos compitiendo por el mismo `< >`, uno puntual podría no entrar nunca. Depende de la **fairness** con que esté implementado el `await` (incondicional / débil / fuerte, Teoría 1), **no del diseño de la solución**. En un parcial alcanza con nombrarlo y hacer esa distinción.

---

## 8. Chequeo de sanidad

Con `N = 1` la solución colapsa a la versión secuencial de §0: un solo proceso, la cola nunca está vacía cuando la pide, el `await` nunca demora.

El más útil: **contá instancias en cualquier momento.** Si parás el programa entre dos acciones atómicas, la suma de las que están en la cola más las que están en uso tiene que dar **exactamente 5**, y ninguna instancia puede aparecer dos veces. Los tres errores de §5 rompen alguna de esas dos cosas.

---

## 9. Los tres chequeos del corrector

1. **¿`usar(rec)` está fuera del `< >`?** Si está adentro, la solución es correcta pero inútil: 5 instancias que se comportan como 1.
2. **¿`rec` es local?** Es lo que separa la respuesta completa de la que parece bien pero pierde instancias.
3. **¿Distingue el `pop` condicional del `push` incondicional?** Poner `await` en el `push` no rompe nada, pero muestra que no se entendió el invariante `en_cola + en_uso = 5`.
