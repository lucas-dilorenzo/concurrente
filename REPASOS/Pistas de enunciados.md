# Pistas: del enunciado al mecanismo

> Diccionario armado sobre los enunciados reales de **Práctica 1** (variables compartidas) y **Práctica 2** (semáforos). Para repasar el día antes del parcial.

**Antes de las pistas puntuales**, el orden en que conviene leer cualquier enunciado:

1. **¿Quiénes son los procesos?** Los sustantivos que *hacen* algo repetidamente (Persona, Camión, Preparador, Enfermera).
2. **¿Qué se comparte?** Los sustantivos que se *usan* (impresora, detector, BD, depósito, cola, vector).
3. **¿Dónde alguien tiene que esperar?** Cada espera distinta va a ser un semáforo distinto.

---

## 1. ¿`while (true)` o no?

De las que más puntos sacan y casi nadie mira.

| Si el enunciado dice… | Entonces… |
|---|---|
| "**continuamente**", "cada vez que", "va y vuelve" | `while (true)` |
| "**N personas que deben imprimir un trabajo cada una**" | **SIN** `while(true)`. Cada proceso hace su cosa **una vez** y termina |
| "N personas deben **ser chequeadas** antes de ingresar al avión" | Una vez cada una. Pasás por el detector y subís al avión, no volvés |
| "cada persona pueda pasar **más de una vez**, siendo **aleatoria** esa cantidad" | `for j = 1 to random()` — acotado, pero no fijo |
| "**cuando ha atendido a las 50 personas se retira**" | El servidor lleva **cuenta** y hace un `for` acotado. `while(true)` acá **está mal**: el enunciado pide que termine |
| "**todos los procesos deben terminar su ejecución**" | Está avisándote que hay trabajo extra de terminación. Ningún `while(true)` suelto |

> **El test:** *¿este proceso tiene una cantidad de trabajo conocida?* Si el enunciado te da un número (50 personas, 150 pasajeros, T piezas), hay terminación. Si dice "continuamente" y no da ningún total, es `while(true)`.

**Caso mixto** (fábrica de ventanas, P2 Ej. 9): los carpinteros "continuamente hacen marcos" → `while(true)`, sin terminación, porque el enunciado nunca dice cuántas ventanas.

---

## 2. El valor inicial del semáforo: los tres usos

Lo que más rinde memorizar. **El valor inicial te dice para qué sirve.**

| Inicial | Para qué | Cómo suena en el enunciado |
|---|---|---|
| `sem s = 1` | **Exclusión mutua** (candado) | "de a **una persona a la vez**", "un único X", "no puede haber dos al mismo tiempo" — o directamente: hay una variable compartida que **se lee y se escribe** |
| `sem s = K` | **Contador de recursos** (K > 1) | Un **número de capacidad**: "**5 instancias**", "**6 usuarios** como máximo", "**3 detectores**", "capacidad para **N** contenedores", "**7 camiones** a la vez", "**30 marcos**" |
| `sem s = 0` | **Señal / condición** | "**esperar a que**", "le **avisa**", "le **indica**", "espera a que la enfermera lo **llame**", "el profesor les **otorga**" |

### La regla que nunca falla: mirá quién hace el `V`

| El `V` está… | Entonces es… | Inicial |
|---|---|---|
| en **el mismo proceso** que el `P` | un **candado** o un **recurso que devolvés** | ≥ 1 |
| en **otro proceso** | una **señal** (te avisan) | **0** |

Si escribiste `P(x)` y `V(x)` en el mismo proceso pero lo inicializaste en 0, **está mal**: te colgás en el primer `P`. Y al revés: si un proceso hace `P(x)` y otro el `V(x)`, ese semáforo casi seguro arranca en 0.

> **En criollo:** *inicial ≥ 1 = "hay permiso disponible". Inicial 0 = "esperá que alguien te avise".*

---

## 3. ¿Un semáforo, o un arreglo de semáforos?

| Si el enunciado… | Entonces | Forma |
|---|---|---|
| Sólo dice **cuántos** hay, y son **intercambiables** | **Un** contador | `sem detector = 3;` |
| Te dice **cuál** te toca | **Un arreglo**, uno por recurso | `sem puesto[3] = ([3] 1);` |
| Alguien **despierta a un proceso específico** | **Arreglo de semáforos privados en 0** | `sem priv[N] = ([N] 0);` |

- *"Existen tres detectores de metales"* (P2 Ej. 1c) → da igual cuál te toque. **Un contador en 3.**
- *"El coordinador le indica cuándo puede usar una impresora, **y cuál debe usar**"* (P2 Ej. 6e) → hay que identificarla. **Arreglo.**
- *"En cada puesto hay una Enfermera… el pasajero espera a que la enfermera correspondiente lo **llame**"* (P2 Ej. 12) → despertar individual. **Arreglo de privados en 0.**

> **Cuidado con la notación:**
> ```
> sem disponible = 5;            // UN semáforo cuyo valor es 5
> sem instancias[5] = ([5] 1);   // CINCO semáforos, cada uno vale 1
> ```
> El primero modela "5 unidades intercambiables". El segundo, "5 recursos distintos, cada uno con su candado".

---

## 4. Semáforos cruzados (el patrón del buffer)

**La pista maestra:** hay un **lugar donde se guardan cosas**, con **capacidad limitada**, alguien que **pone** y alguien que **saca**.

Palabras que lo delatan: *depósito · sala · capacidad para K · buffer · los deja en · toman de · almacenar*

```
sem vacios = K;      // lugares libres  → lo espera el que PONE
sem llenos = 0;      // cosas guardadas → lo espera el que SACA
```

Cada bando hace `P` sobre lo que **consume** y `V` sobre lo que **produce**. Invariante: `vacios + llenos = K`.

- *"sala con capacidad para N contenedores"*, Preparador deja / Entregador toma (P2 Ej. 5) → **un par cruzado**.
- *"depósito de 30 marcos"* + *"otro depósito para 50 vidrios"* (P2 Ej. 9) → **dos pares cruzados**, independientes.

> **Regla de conteo:** *tantos pares cruzados como depósitos distintos haya.* No los mezcles en uno.

**Ojo con esta frase** de la fábrica de ventanas: *"toman un marco y un vidrio **(en ese orden)**"*. No es decorativo — te **fija el orden de los `P`** para que todos los armadores pidan igual y no haya deadlock (uno esperando marco con un vidrio en la mano y otro al revés).

---

## 5. ¿Hace falta mutex, o alcanza con los contadores?

**No mires si la estructura es compartida: contá los escritores.**

| Situación | ¿Mutex? |
|---|---|
| Una variable con **un solo proceso** que la toca | **No.** Es local, no hay con quién competir |
| **Varios procesos del mismo rol** tocando la misma variable | **Sí** |
| Un arreglo donde **cada proceso escribe su propio casillero** (`v[id]`) | **No.** Cada casillero tiene un único dueño |
| Todos suman sobre **la misma** variable (`total += x`) | **Sí** |

- **Sala de N contenedores con 1 Preparador y 1 Entregador** (P2 Ej. 5a): el arreglo es compartido y **no lleva mutex**, porque cada índice tiene un solo dueño y los semáforos `vacios`/`llenos` ya impiden que los dos toquen la misma celda. Con P Preparadores, el índice deja de tener un solo dueño → aparece el mutex.
- **Contar fallos por gravedad en `vector[0..3]`** (P2 Ej. 2c): si cada proceso se ocupa de **un** nivel, escribe sólo `vector[id]` → **cero sincronización**. Repartir bien el trabajo puede eliminar la necesidad de sincronizar.

> **Para el oral:** *una variable con un único dueño no necesita candado, por más compartida que esté declarada.*

---

## 6. ¿Proceso activo, o semáforo pasivo?

| Si el enunciado… | Es un… |
|---|---|
| Le hace **tomar decisiones**: "le **indica** qué puesto", "**atiende según** el orden de llegada", "les **otorga** un puntaje", "le **avisa** que es su turno" | **Proceso** (activo) |
| Sólo lo describe como algo que **se ocupa y se libera**: el detector, la impresora, la BD, un contenedor | **Semáforo** (pasivo) |

**El test:** ¿*decide* algo, o sólo *se usa*? Decide → proceso. Se usa → semáforo.

En la terminal de micros (P2 Ej. 12), el **Recepcionista decide** a qué puesto mandarte → proceso; la **Enfermera llama** a un pasajero → proceso; pero el **puesto** es sólo un lugar → semáforo.

### La pista negativa, que es la más importante

- *"**Sólo se deben usar los procesos que representan a las Personas**"* (P2 Ej. 6a)
- *"Implemente una solución que **no use procesos adicionales** (sólo camiones)"* (P2 Ej. 10b)

…te está **prohibiendo el coordinador**. La coordinación tiene que salir de los procesos que compiten, pasándose el permiso unos a otros. **Ahí aparece Passing the Baton**: es literalmente "administrar un recurso sin un árbitro externo".

Cuando el enunciado **sí** te da el coordinador (*"un proceso extra que actúe como coordinador"*, P2 Ej. 10a), el problema se vuelve mucho más fácil: el coordinador lleva la cola a mano y despierta con `V(priv[i])`.

> Varios ejercicios piden **las dos versiones del mismo problema** (con y sin coordinador). Es a propósito: Passing the Baton es "el coordinador, pero repartido entre los que compiten".

---

## 7. El orden: tres sabores bien distintos

| Si dice… | Es… | Mecanismo |
|---|---|---|
| "**sin importar el orden**" | Nada especial | Un `mutex = 1` y listo |
| "**según el orden de llegada**" / "el primero que llegó" | **FIFO dinámica** | **Cola explícita + arreglo de semáforos privados en 0.** No alcanza con un mutex |
| "orden dado por el **identificador**", "X no puede hasta que **termine X−1**" | **Orden estático** | **Posta**: `sem turno[N] = ([1] 1, [N-1] 0)`; cada uno `P(turno[i])` … `V(turno[i+1])` |

La distinción que importa: **el orden por identificador se sabe de antemano** (lo cableás), mientras que **el orden de llegada hay que registrarlo en tiempo de ejecución**.

> **La razón de fondo:** la práctica aclara que **no se puede suponer que los semáforos sean FIFO**. Si diez procesos están bloqueados en `P(mutex)`, no está definido cuál despierta. Si el enunciado pide orden, **el orden lo construís vos** — el semáforo no te lo regala.

Ese es el motivo por el que existen los ejercicios de Passing the Baton.

---

## 8. Barrera

| Si dice… | Es una barrera |
|---|---|
| "**Una vez que todos** los alumnos eligieron su tarea, **comienzan** a realizarla" (P2 Ej. 7) | ✔ |
| "La fábrica **empieza a producir una vez que todos** los empleados llegaron" (P2 Ej. 8) | ✔ |

La forma: *"nadie avanza a la etapa 2 hasta que **todos** terminaron la etapa 1"*.

Con semáforos: contador compartido + su mutex + `sem barrera = 0`; cada uno incrementa, y **el último** (el que ve el contador en N) libera a los demás con N−1 `V`.

**Pista adicional del Ej. 7:** *"espera el puntaje **del grupo** (depende de todos los que comparten el mismo enunciado)"* → no es **una** barrera, son **10 barreras** (una por grupo). El "todos" es relativo al grupo, no al total.

---

## 9. Bolsa de tareas vs. reparto fijo

| Si dice… | Entonces |
|---|---|
| "P procesos recorren un arreglo de M elementos" | **Reparto por bloques** (cada uno su rango, calculado con el `id`). Sin sincronización durante el recorrido |
| "**mientras haya piezas por fabricar, tomarán una** y la realizarán" | **Bolsa de tareas**: contador compartido + mutex |
| "**cada empleado puede tardar distinto tiempo**" | Confirma bolsa de tareas. Esa frase está puesta para **descartar** el reparto fijo |

Si todos tardan lo mismo, repartís de antemano. Si tardan distinto, el reparto fijo deja a los rápidos parados esperando a los lentos.

**Detalle del Ej. 8:** *"(a) asumiendo que T > E"* y *"(b) cualquier valor de T y E"*. El (b) te obliga a manejar **T < E** — empleados que llegan a la bolsa y ya no queda nada. Ese caso borde es lo único que separa los dos ítems.

---

## 10. "Se debe conocer cuál/cuántos…" (los resultados)

| Si dice… | Entonces |
|---|---|
| "los resultados deben quedar en un **vector global**" | Acumulador **local** por proceso + volcado al vector **al final**, no en cada iteración |
| "cada proceso se ocupa de **un** nivel/tipo determinado" | Cada uno escribe **su propio casillero** → **sin mutex** |
| "se debe conocer **cuál** empleado fabricó **más** piezas" | Contador local por empleado + comparación final |
| "imprimir en pantalla, **no importa el orden**" | La pantalla es recurso compartido → `mutex = 1`. Sin él la salida sale **entreverada** (renglones partidos), no sólo desordenada |

*Desordenado* ≠ *entreverado*: el enunciado te perdona el desorden, pero nunca que se mezclen los caracteres de dos líneas.

---

# La chuleta de una carilla

| Frase del enunciado | Mecanismo |
|---|---|
| "de a uno a la vez" | `sem mutex = 1` |
| número de capacidad (5 instancias, 6 usuarios, 3 detectores, N contenedores) | `sem s = K` (contador) |
| "esperar a que", "le avisa", "le indica", "lo llama" | `sem s = 0` (señal) |
| depósito/sala/buffer con capacidad K, uno pone y otro saca | `sem vacios = K` + `sem llenos = 0` **cruzados** |
| dos depósitos | **dos pares** cruzados, independientes |
| "toman A y B **en ese orden**" | orden fijo de los `P` → evita deadlock |
| "continuamente" | `while (true)` |
| "cada una un trabajo", "las 50 y se retira", "todos deben terminar" | **sin** `while(true)`; loop acotado + terminación |
| "sin importar el orden" | mutex y nada más |
| "según el orden de llegada" | cola + `sem priv[N] = ([N] 0)` → Passing the Baton o coordinador |
| "orden por identificador", "X después de X−1" | posta: `sem turno[N] = ([1] 1, [N-1] 0)` |
| "una vez que **todos**… comienzan" | barrera (contador + mutex + `sem barrera = 0`) |
| "mientras haya… toma una", "tardan distinto" | bolsa de tareas |
| "le indica", "atiende según", "otorga" | **proceso** coordinador |
| "sólo los procesos X", "sin procesos adicionales" | **prohibido** el coordinador → Passing the Baton |
| "le indica **cuál** debe usar" | **arreglo** de semáforos, no un contador |
| "cada uno se ocupa de un tipo determinado" | `vector[id]` → **sin** sincronización |
| operación larga (usar, imprimir, entregar, hisopar, descargar) | **afuera** de la sección crítica, siempre |

Y las dos reglas transversales que valen para cualquier ejercicio:

1. **`P(contador)` antes que `P(mutex)`.** Nunca te bloqueás esperando un recurso con el candado en la mano.
2. **La operación larga va afuera del candado.** Adentro, K recursos rinden como 1 y N procesos rinden como 1.
