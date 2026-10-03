# Práctica 2 — Ejercicio 6

> **Enunciado.** Existen **N personas** que deben imprimir **un trabajo cada una**. Resolver cada ítem usando semáforos:
> **a)** Una **única impresora** compartida, usada **de a una persona a la vez, sin importar el orden**. Existe una función `Imprimir(documento)` llamada por la persona que simula el uso de la impresora. **Sólo se deben usar los procesos que representan a las Personas.**
> **b)** Modifique (a) para que se respete el **orden de llegada**.
> **c)** Modifique (a) para respetar estrictamente el orden dado por el **identificador** del proceso (la persona X no puede usar la impresora hasta que no haya terminado de usarla la persona X−1).
> **d)** Modifique (b) para el caso en que además hay un proceso **Coordinador** que le indica a cada persona que es su turno de usar la impresora.
> **e)** Modifique (d) para el caso en que sean **5 impresoras**. El coordinador le indica a la persona cuándo puede usar una impresora, **y cuál** debe usar.

---

# a) Una impresora, sin importar el orden

## Dos detalles del enunciado antes de escribir nada

**(1) No hay `while(true)`.** Dice *"N personas que deben imprimir **un trabajo cada una**"*. Cada persona imprime **una sola vez** y termina. Un `while(true)` acá cambia el problema y se penaliza.

**(2) `Imprimir()` va ADENTRO del candado.** Hay una regla general que dice *"la operación larga va afuera de la sección crítica"* — y acá **no aplica**:

| Caso | ¿Dónde va la operación larga? |
|---|---|
| Sacás un recurso de una estructura y después lo usás | El candado protege **la estructura**; el uso va **afuera** |
| La operación larga **es** el uso del recurso protegido | El candado **es** el recurso; la operación va **adentro** |

`Imprimir()` es literalmente "usar la impresora", y la impresora es lo que hay que serializar. Afuera del `mutex`, dos personas imprimirían a la vez y la hoja saldría con los dos documentos entreverados.

## Solución (a)

```
// PRECONDICIONES: N > 0

sem mutex = 1;                    // la impresora: libre (1) u ocupada (0)

Process Persona[id: 1..N]
{ P(mutex);                       // espero a que esté libre
  Imprimir(documento);            // la uso — ADENTRO: la impresora ES el recurso
  V(mutex);                       // la libero
}                                 // sin while(true): imprimo una vez y me voy
```

Un semáforo inicializado en **1** = exclusión mutua. El más simple de los tres usos.

## ¿Y el orden en que imprimen?

**Indefinido, y el enunciado lo permite** (*"sin importar el orden"*). Si hay diez personas bloqueadas en `P(mutex)`, **no está definido cuál despierta**: la práctica aclara que no se puede suponer que los semáforos sean FIFO. Podría imprimir la última que llegó.

No es un defecto acá. Pero es **exactamente el agujero que abre el ítem (b)**.

---

# b) Respetando el orden de llegada

## 1. Por qué el (a) no se puede "arreglar"

> **El punto de partida:** un semáforo te dice *cuántos permisos hay*, pero **no lleva registro de quién llegó primero**. Cuando hacés `V(mutex)`, se despierta a **alguno** de los bloqueados — no sabés a cuál y no lo podés influir.
>
> **Si el enunciado te pide un orden, el orden lo construís vos.** El semáforo no te lo va a dar.

Y el enunciado del (a) —que el (b) hereda— dice *"sólo se deben usar los procesos que representan a las Personas"*: **no podés poner un Coordinador**. La organización tiene que salir de las propias personas.

## 2. Qué necesitamos

| Necesidad | Herramienta |
|---|---|
| **Anotar** quién llegó y en qué orden | Una **cola** (FIFO) con los `id` |
| **Despertar a una persona específica** (no a "alguna") | Un **semáforo privado por persona**: `sem priv[N] = ([N] 0)` |
| Que la cola y el estado no se corrompan | Un **mutex**: `sem e = 1` |

La pieza que no se ocurre sola es la del medio:

> **Un semáforo compartido despierta a "alguno". Un semáforo por persona despierta a "ese".**
> Si `priv[3] = 0` y sólo la Persona 3 hace `P(priv[3])`, entonces `V(priv[3])` la despierta a ella y a nadie más. Es un **timbre con nombre**.

Un semáforo privado siempre arranca en **0**: es una señal, y su `V` lo hace **otro** proceso.

## 3. La idea: el traspaso directo

**La analogía.** Un baño con una sola llave.

- **Versión (a):** salís, **colgás la llave en la pared** y gritás "¡libre!". Se abalanzan todos y entra el más rápido — que puede ser el que acaba de llegar.
- **Versión (b):** salís y **le ponés la llave en la mano** al primero de la fila. La llave **nunca vuelve a la pared**.

Eso es **Passing the Baton**: el recurso **no se libera, se entrega**.

> **Consecuencia clave:** la impresora **nunca vuelve al estado "libre"** mientras haya gente esperando. `libre` se queda en `false` durante todo el traspaso. Si se pusiera en `true` aunque sea un instante, alguien recién llegado podría meterse en el medio y se rompería el orden.

## 4. Solución (b)

```
// PRECONDICIONES: N > 0

sem e = 1;                      // protege el estado (libre y cola). NO es la impresora
sem priv[N] = ([N] 0);          // el "timbre con nombre" de cada persona
bool libre = true;              // ¿la impresora está sin dueño?
cola espera;                    // ids, en orden de llegada

Process Persona[id: 1..N]
{
  // ── LLEGADA ───────────────────────────────────────────────
  P(e);
  if (libre) {
      libre = false;            // me la quedo yo
      V(e);                     // suelto el candado del estado
  } else {
      push(espera, id);         // me anoto en la fila
      V(e);                     // suelto el candado ANTES de dormirme (clave)
      P(priv[id]);              // me duermo hasta que me toque a MÍ
  }

  // ── USO ───────────────────────────────────────────────────
  Imprimir(documento);          // AFUERA de e: e protege el estado, no la impresora

  // ── SALIDA ────────────────────────────────────────────────
  P(e);
  if (empty(espera)) {
      libre = true;             // no hay nadie: recién ahora la dejo libre
  } else {
      pop(espera, siguiente);
      V(priv[siguiente]);       // TRASPASO: se la paso en la mano
                                // libre SIGUE en false — nunca vuelve a la pared
  }
  V(e);
}
```

### Por qué cada decisión

| Decisión | Por qué |
|---|---|
| `V(e)` **antes** de `P(priv[id])` | Si te dormís **con el candado en la mano**, nadie puede anotarse ni salir. **Deadlock instantáneo.** Es el error más grave del ejercicio |
| `Imprimir()` **afuera** de `e` | `e` protege `libre` y `espera` (dos asignaciones), **no la impresora**. La exclusión sobre la impresora la da el protocolo de traspaso. Adentro, la fila se pararía mientras alguien imprime |
| `libre` queda en `false` al traspasar | Es **toda la idea**. En `true`, un recién llegado entra por la rama `if (libre)` y se cuela |
| `priv[]` arranca en **0** | Es una señal: el `P` lo hace quien espera, el `V` lo hace **otro** |
| `e` arranca en **1** | Es un candado: `P` y `V` en el **mismo** proceso |

Conviven los **dos usos** del semáforo: `e` en 1 (candado) y `priv[]` en 0 (señal). Distinguirlos es lo que hace legible el código.

## 5. La duda clásica: ¿por qué el despertado no vuelve a hacer `P(e)`?

Después de `P(priv[id])` la persona va **derecho a `Imprimir()`**. No reentra.

> **Respuesta corta: porque si volviera a competir, se perdería todo el trabajo de haber ordenado la fila.**

Y la respuesta larga es concreta: veamos qué pasaría **exactamente** si reentrara.

### Si reentrara, se cuelga

```
P(priv[id]);
P(e);                    ← supongamos que agregamos esto
if (libre) { ... }       ← libre vale FALSE
else { push(espera,id); V(e); P(priv[id]); }   ← se re-anota y se duerme
```

`libre` está en `false` —porque el que salió **se la pasó en la mano**, no la liberó— así que caería por el `else`, **se anotaría de nuevo al final de la fila y se dormiría para siempre**. Nadie la va a volver a despertar: su turno ya se usó.

### Y si lo "arreglás" poniendo `libre = true`, rompés el orden

| # | Qué pasa |
|---|---|
| 1 | La Persona 1 termina, pone `libre = true` y hace `V(priv[2])` |
| 2 | La Persona 2 está **despierta pero todavía no la ejecutó el scheduler** |
| 3 | Llega la Persona 9 (recién llegada), hace `P(e)`, ve `libre == true` **y se lleva la impresora** |
| 4 | La Persona 2 por fin corre, ve `libre == false`, se re-anota al final de la fila |

La Persona 2 esperaba desde el principio y la 9 acaba de llegar. **El orden de llegada se rompió** — lo único que pedía el ítem.

### La conclusión

> **El proceso despertado no reentra porque no tiene nada que negociar: ya ganó.** El que salió no le avisó "la impresora quedó libre, andá a ver" — le avisó **"la impresora es tuya"**. Volver a hacer `P(e)` sería ponerse de nuevo en la fila habiendo llegado a la caja.

Esa es la diferencia entre **liberar** un recurso y **entregarlo**.

## 6. La traza

Tres personas. P1 llega primera, P2 segunda, P3 tercera.
Estado inicial: `e=1`, `priv[1..3]=0`, `libre=true`, `espera=[]`.

| # | Quién | Qué ejecuta | `e` | `libre` | `espera` | Estado |
|---|---|---|---|---|---|---|
| 1 | **P1** | `P(e)`; ve `libre`; `libre=false`; `V(e)` | 1 | **false** | `[]` | P1 → a imprimir |
| 2 | **P2** | `P(e)`; ocupado; `push(2)`; `V(e)`; `P(priv[2])` | 1 | false | `[2]` | **P2 dormida** |
| 3 | **P3** | `P(e)`; ocupado; `push(3)`; `V(e)`; `P(priv[3])` | 1 | false | `[2,3]` | **P3 dormida** |
| 4 | **P1** | termina; `P(e)`; cola no vacía; `pop`→2; `V(priv[2])`; `V(e)` | 1 | **false** ← *no cambia* | `[3]` | **P2 despierta**; P1 termina |
| 5 | **P2** | (no hace `P(e)`) → `Imprimir()` | 1 | false | `[3]` | P2 → a imprimir |
| 6 | **P2** | termina; `P(e)`; `pop`→3; `V(priv[3])`; `V(e)` | 1 | **false** | `[]` | **P3 despierta**; P2 termina |
| 7 | **P3** | `Imprimir()` | 1 | false | `[]` | P3 → a imprimir |
| 8 | **P3** | termina; `P(e)`; cola **vacía** → `libre=true`; `V(e)` | 1 | **true** | `[]` | P3 termina |

**Orden de impresión: P1, P2, P3** = orden de llegada. ✔

Las dos cosas para mirar:

- **`libre` se queda en `false` del paso 1 al 8.** Nunca parpadea: la impresora pasa de mano en mano sin volver a estar disponible.
- **`libre` vuelve a `true` en un solo lugar:** cuando el que sale encuentra la **cola vacía**. Ahí no hay a quién entregársela, así que se cuelga en la pared.

## 7. Los dos idiomas de Passing the Baton

Si comparás este código con el de la teoría (Clase 3) vas a ver una diferencia. Los dos resuelven lo mismo, pero **qué significa el `V(priv[i])`** cambia:

| | **Idioma "permiso"** (el de arriba) | **Idioma "bastón"** (el de la teoría) |
|---|---|---|
| Qué transmite `V(priv[i])` | El **permiso de usar el recurso** | El **bastón de la sección crítica** |
| El que sale hace `V(e)` | **Sí**, siempre | **No** en la rama del traspaso |
| El que despierta, ¿está en la SC? | **No** — va directo a usar el recurso | **Sí** — despierta *adentro* de la SC |
| ¿Tiene que actualizar el estado? | No, ya se lo dejaron hecho | **Sí**, y después pasa el bastón él |

**En el idioma "bastón"**, el que sale **omite el `V(e)` a propósito**: el candado queda cerrado, pero su *propiedad* se transfirió. El despertado hereda la SC, y por eso tampoco hace `P(e)` — **ya lo tiene**.

**En el idioma "permiso"**, el candado se libera normalmente en ambas ramas y por `priv[i]` viaja sólo el derecho a usar la impresora. Tampoco hace `P(e)`, pero **porque no lo necesita**.

> **Esto es lo que cierra la duda:** la pregunta *"¿por qué no vuelve a hacer `P(e)`?"* tiene **dos respuestas distintas según el idioma**, y por eso confunde tanto si los mezclás.
>
> - Idioma bastón: **porque ya lo tiene** (el otro no lo soltó).
> - Idioma permiso: **porque no lo necesita** (no tiene nada que hacer adentro).
>
> Lo único que comparten —y es lo que importa— es la razón de fondo: **el despertado no vuelve a competir.** Si compitiera, el orden que construiste con la cola se evaporaría.

**Para el parcial: usá el idioma "permiso".** Es más fácil de escribir sin equivocarse (todo `P` tiene su `V` visible), más fácil de justificar, y resuelve todos los ejercicios de esta práctica. El "bastón" rinde de verdad en lectores/escritores, donde el despertado **sí** tiene contadores que actualizar.

## 8. Los errores que se penalizan

**(1) Dormirse con el candado.**
```
P(e);
push(espera, id);
P(priv[id]);        ← MAL: nunca hizo V(e)
```
**Deadlock total.** El error más caro del ejercicio.

**(2) `Imprimir()` adentro de `e`.** Funciona, pero convierte `e` en el candado de la impresora — y entonces la cola no sirve para nada, porque las personas vuelven a competir por `P(e)`.

**(3) Poner `libre = true` al traspasar.** Rompe el orden de llegada (traza del punto 5).

**(4) Un solo semáforo compartido en vez de `priv[N]`.** `V(esperar)` despierta a **alguno**: volvés al problema del ítem (a). Necesitás el timbre **con nombre**.

## 9. Chequeos de sanidad

- **Con N = 1:** entra por `if (libre)`, imprime, encuentra la cola vacía, pone `libre = true`. Nunca se usó `priv[]`. Colapsa en el ítem (a). ✔
- **`libre == false` ⟺ hay exactamente una persona con la impresora asignada** (imprimiendo o recién despertada). Es el invariante para escribir en el papel.
- **Cada `priv[i]` recibe a lo sumo un `V`**, y sólo si la persona `i` se anotó.
- **Todo el que entra a la cola, sale:** el que termina siempre revisa la cola antes de irse, así que la cadena de traspasos no se corta.

---

# c) Orden estricto por identificador

## 1. La frase entre paréntesis es la especificación

*"la persona X no puede usar la impresora hasta que no haya terminado de usarla la persona X−1"*:

```
persona 1 → persona 2 → persona 3 → … → persona N
```

Una cadena fija, conocida **antes de que el programa arranque**:

| | ¿Cuándo se conoce el orden? | ¿Hay que registrarlo? |
|---|---|---|
| **Orden de llegada** (ítem b) | En ejecución — nadie sabe quién llega primero | **Sí**: cola + anotarse |
| **Orden por identificador** (ítem c) | **De antemano.** Está en los números | **No**: ya está escrito |

> En el (b) hubo que **construir** la fila. Acá **la fila ya existe** — es la numeración de los procesos.

Por eso (c) es **más corto que (a)**, aunque suene más difícil. Mucha gente lo resuelve con la maquinaria del (b), que acá sobra.

## 2. La idea: la posta

**La analogía.** Una carrera de postas. Hay **un solo testimonio** y el recorrido está fijado: el corredor 1 corre su tramo y se lo entrega al 2; el 2 al 3. Nadie compite por el testimonio ni elige a quién dárselo: **la ruta está dibujada de antemano.**

- **El testimonio** = el permiso de usar la impresora.
- **Cada corredor tiene su propia mano** → un semáforo por persona, `sem turno[N]`.
- **Sólo el primero arranca con él** → `turno[1] = 1`, el resto en **0**.
- **Cada uno se lo pasa al siguiente** → `V(turno[id+1])`.

Es **el mismo traspaso directo del ítem (b)**, pero sin cola: **la cola es la lista de identificadores.**

## 3. Solución (c)

```
// PRECONDICIONES: N > 0
//                 los procesos se identifican 1..N

sem turno[N] = ([1] 1, [N-1] 0);     // turno[1]=1, el resto en 0

Process Persona[id: 1..N]
{ P(turno[id]);                      // espero MI turno (el testimonio)
  Imprimir(documento);               // uso la impresora
  if (id < N)                        // si no soy el último…
     V(turno[id+1]);                 // …se lo paso al siguiente
}                                    // sin while(true)
```

Cinco líneas: **cero mutex, cero cola, cero variables de estado.**

La declaración `sem turno[N] = ([1] 1, [N-1] 0);` se lee *"el primer casillero vale 1, los N−1 restantes valen 0"*.

## 4. Las tres preguntas que te van a hacer

**¿Por qué no hace falta mutex?**

> Existe **un único permiso** en todo el sistema: el `1` con el que arranca `turno[1]`. Cada persona lo **consume** con su `P` y **crea uno nuevo** para la siguiente con su `V`. Nunca hay dos en circulación.

La exclusión mutua **es una consecuencia de la cadena**, no algo que se pide con un candado. Agregar un `sem mutex = 1` no rompe nada pero es redundante y señala que no entendiste el mecanismo.

**¿Por qué no hace falta cola?** Una cola sirve para recordar un orden que no sabías de antemano. Acá lo sabés: es `1, 2, …, N`.

**¿Por qué N semáforos y no uno solo?** Con un único `sem impresora = 1`, el `V` despierta a *alguno* — y **no se puede suponer que los semáforos sean FIFO**. Podría despertar a la persona 7 cuando le tocaba a la 2. Con `turno[N]`, `V(turno[id+1])` despierta **a esa y sólo a esa**.

## 5. La traza: llegan al revés y funciona igual

Tres personas, llegando en orden invertido: primero la 3, después la 2, la 1 al final. Inicial: `turno = [1,0,0]`.

| # | Quién | Qué ejecuta | `turno` | Durmiendo |
|---|---|---|---|---|
| 1 | **P3** | `P(turno[3])` → vale 0, **se bloquea** | `[1,0,0]` | P3 |
| 2 | **P2** | `P(turno[2])` → vale 0, **se bloquea** | `[1,0,0]` | P3, P2 |
| 3 | **P1** | `P(turno[1])` → vale 1, **pasa** | `[0,0,0]` | P3, P2 |
| 4 | **P1** | `Imprimir()` | `[0,0,0]` | P3, P2 |
| 5 | **P1** | `id < N` → `V(turno[2])`. Termina | `[0,1,0]` → P2 despierta | P3 |
| 6 | **P2** | `Imprimir()`; `V(turno[3])`. Termina | `[0,0,1]` → P3 despierta | — |
| 7 | **P3** | `Imprimir()`; `id == N` → **no avisa a nadie**. Termina | `[0,0,0]` | — |

**Orden de impresión: 1, 2, 3.** Aunque hayan llegado 3, 2, 1. Eso es "estrictamente": **el orden de llegada es irrelevante.**

## 6. El costo: la impresora puede quedar parada

En los pasos 1 y 2 la persona 3 llegó primera, la impresora estaba **libre**, y aun así esperó. Nadie imprimió hasta que apareció la persona 1.

> **Es un defecto del enunciado, no de tu solución.** En (b) la impresora nunca está ociosa habiendo gente esperando; acá sí. **El orden estricto se paga con rendimiento.**

Conviene escribirlo en la justificación: muestra que distinguís "correcto" de "eficiente".

### Justificación (esto es lo que suma puntos)

> El orden exigido es estático y conocido antes de la ejecución, así que no hace falta registrarlo: la secuencia de identificadores **es** la fila. Se usa un arreglo `turno[N]` de semáforos privados, con el primero en 1 y el resto en 0, de modo que existe un único permiso circulando. Cada persona lo consume con `P(turno[id])` y genera el de la siguiente con `V(turno[id+1])`, lo que da exclusión mutua sin mutex: nunca hay dos permisos simultáneos. Se requiere un semáforo por persona y no uno compartido porque hay que despertar a un proceso determinado y no puede suponerse que los semáforos sean FIFO. La última persona no señaliza a nadie. La contrapartida del orden estricto es que la impresora puede quedar ociosa esperando al proceso que le toca, aunque haya otros listos.

## 7. Los errores que se penalizan

**(1) Busy waiting.** La tentación más grande:
```
int turno = 1;
Process Persona[id: 1..N]
{ while (turno != id) skip;             // ← MAL: busy waiting
  Imprimir(documento);
  turno++;                              // ← y además no es atómico
}
```
Las **consideraciones de la práctica lo prohíben explícitamente**.

**(2) El `V` del último.** Sin el `if (id < N)`, con `id == N` accedés a `turno[N+1]`, que no existe. Y si lo hacés circular con `V(turno[(id mod N) + 1])`, le dejás un permiso a la persona 1, **que ya imprimió y terminó**: queda un semáforo sucio. Como cada persona imprime una sola vez, la cadena **termina**, no da la vuelta.

**(3) Traer la maquinaria del ítem (b).** Funciona, pero es tres veces más código para un orden que ya estaba escrito.

**(4) Un solo semáforo compartido.** Despierta a *alguno*: no hay forma de dirigir el turno.

## 8. Cuidado: hay un enunciado casi igual que NO se resuelve así

| Enunciado | Qué pide | Solución |
|---|---|---|
| *"la persona X no puede usar la impresora hasta que no haya terminado la persona X−1"* | Cadena **fija**: 1, 2, …, N pase lo que pase | **Posta** (esta) |
| *"cuando está libre la impresora, **de los procesos que han solicitado su uso**, la debe usar el que tenga **menor identificador**"* | **Prioridad dinámica**: entre los que ya pidieron, gana el de id más chico | **Cola + selección del mínimo** (maquinaria del ítem b, cambiando el `pop` FIFO por "sacar el menor") |

La frase que los separa es **"de los procesos que han solicitado su uso"**. Si está, el orden depende de **quién pidió**, y eso sólo se sabe en ejecución.

> **La pista:** *si el orden se puede dibujar antes de ejecutar el programa, es una posta. Si depende de quién llegó o quién pidió, hace falta una estructura que lo registre.*

## 9. Chequeos de sanidad

- **Con N = 1:** `turno[1]=1`, la única persona pasa, imprime, `id == N` así que no avisa. Termina. ✔
- **Contá permisos:** 1 en la inicialización + N−1 de los `V` = **N permisos**, uno por persona. Si te da N, la cadena está bien cerrada.
- **Todos terminan:** la persona `i` sólo depende de la `i−1`, y la 1 no depende de nadie. Sin ciclo ⇒ **sin deadlock**.
- **Nadie imprime dos veces:** cada `turno[i]` recibe exactamente un `V` y sólo la persona `i` hace `P` sobre él.

---

# d) Con un proceso Coordinador

Ojo: modifica **(b)**, así que el orden a respetar sigue siendo el **de llegada**. Lo único que cambia es **quién organiza la fila**.

## 1. Qué cambia de mano

| Tarea | En (b) la hacía… | En (d) la hace… |
|---|---|---|
| Anotarse al llegar | La persona | La persona (igual) |
| **Elegir quién sigue** | La persona **que termina** | **El Coordinador** |
| **Despertar al elegido** | La persona que termina | **El Coordinador** |
| Recordar si la impresora está ocupada | Una variable `libre` compartida | **El Coordinador** |

> **La idea:** el Coordinador no elimina la fila — **la centraliza**. Antes la administraban entre todos; ahora la administra uno solo.

## 2. Los mensajes que viajan

| Mensaje | De quién a quién | Semáforo |
|---|---|---|
| "llegué, anotame" | Persona → Coordinador | `sem hayPedido = 0` |
| "es tu turno" | Coordinador → **una persona específica** | `sem priv[N] = ([N] 0)` |
| "terminé, quedó libre" | Persona → Coordinador | `sem libre = 0` |
| (proteger la cola de N escritores) | — | `sem mutexCola = 1` |

Los tres primeros en **0** porque son **señales** (`P` en uno, `V` en otro). El cuarto en **1** porque es un **candado** (`P` y `V` en el mismo proceso).

**Por qué `priv[N]` y no un semáforo único:** el Coordinador debe despertar a **la primera de la fila**. Con un semáforo compartido, `V` despierta a *alguno*, y **no se puede suponer que los semáforos sean FIFO**.

> **Punto clave:** el Coordinador te ahorra el traspaso entre personas, pero **no** te ahorra ni la cola ni los semáforos privados. Sigue habiendo que recordar el orden y despertar a alguien puntual.

## 3. Solución (d)

```
// PRECONDICIONES: N > 0;  cada persona imprime exactamente una vez

sem mutexCola = 1;              // protege la cola (N personas escriben en ella)
sem hayPedido = 0;              // Persona → Coordinador: "llegué"
sem libre     = 0;              // Persona → Coordinador: "terminé"
sem priv[N]   = ([N] 0);        // Coordinador → Persona id: "es tu turno"
cola espera;                    // ids, en orden de llegada

Process Persona[id: 1..N]
{ P(mutexCola);
    push(espera, id);           // me anoto
  V(mutexCola);
  V(hayPedido);                 // aviso al coordinador que hay uno más

  P(priv[id]);                  // espero a que ME llame

  Imprimir(documento);          // uso la impresora

  V(libre);                     // aviso al coordinador que terminé
}                               // sin while(true)

Process Coordinador
{ int siguiente;
  for i = 1 to N {              // sé cuántos son: atiendo N y me retiro
     P(hayPedido);              // espero que haya al menos alguien anotado
     P(mutexCola);
       pop(espera, siguiente);  // el primero de la fila
     V(mutexCola);
     V(priv[siguiente]);        // "es tu turno"
     P(libre);                  // y espero a que termine
  }
}
```

## 4. Lo que se simplificó: el estado desapareció

Respecto del ítem (b), **ya no está** `bool libre`. En (b) hacía falta esa variable **más** la regla delicadísima de no ponerla en `true` durante el traspaso. Acá no existe:

> **El estado de la impresora es el punto del programa donde está parado el Coordinador.**
>
> Entre el `V(priv[siguiente])` y el `P(libre)`, el Coordinador está **bloqueado**. Y si está bloqueado, **no puede repartir otro turno**. La exclusión mutua no la da ninguna variable ni ningún candado: la da que **hay un solo repartidor y está esperando**.

Esa línea —`P(libre)`— es la que hace todo el trabajo.

Y de yapa: **el "barging" se volvió imposible.** En (b) había que cuidar que un recién llegado no se colara. Acá **el único que reparte permisos es el Coordinador**, y reparte de a uno.

## 5. Un detalle lindo: la fila es un productor/consumidor

- **N productores** (las personas) que hacen `push` y avisan con `V(hayPedido)`.
- **Un consumidor** (el Coordinador) que espera con `P(hayPedido)` y hace `pop`.
- `hayPedido` es el semáforo de "cuántos elementos hay guardados".

¿Por qué **no hay** un semáforo del otro lado contando lugares libres? Porque **la cola nunca se puede desbordar**: cada persona hace `push` exactamente una vez, así que nunca hay más de N anotados. Un semáforo que nunca bloquea es un semáforo de más.

El `mutexCola` sí hace falta: hay **N** procesos haciendo `push`.

## 6. La traza

Tres personas que llegan en orden **2, 3, 1** — desordenado respecto del identificador a propósito.
Inicial: `hayPedido=0`, `libre=0`, `priv=[0,0,0]`, `espera=[]`.

| # | Quién | Qué ejecuta | `hayPedido` | `espera` | Estado |
|---|---|---|---|---|---|
| 1 | **P2** | `push(2)`; `V(hayPedido)`; `P(priv[2])` | 1 | `[2]` | P2 dormida |
| 2 | **P3** | `push(3)`; `V(hayPedido)`; `P(priv[3])` | 2 | `[2,3]` | P3 dormida |
| 3 | **Coord** | `P(hayPedido)`; `pop`→**2**; `V(priv[2])`; `P(libre)` | 1 | `[3]` | **P2 despierta**; Coord bloqueado |
| 4 | **P1** | `push(1)`; `V(hayPedido)`; `P(priv[1])` | 2 | `[3,1]` | P1 dormida |
| 5 | **P2** | `Imprimir()`; `V(libre)` | 2 | `[3,1]` | **Coord despierta**; P2 termina |
| 6 | **Coord** | `P(hayPedido)`; `pop`→**3**; `V(priv[3])`; `P(libre)` | 1 | `[1]` | P3 despierta |
| 7 | **P3** | `Imprimir()`; `V(libre)` | 1 | `[1]` | P3 termina |
| 8 | **Coord** | `P(hayPedido)`; `pop`→**1**; `V(priv[1])`; `P(libre)` | 0 | `[]` | P1 despierta |
| 9 | **P1** | `Imprimir()`; `V(libre)` | 0 | `[]` | P1 termina |
| 10 | **Coord** | sale del `for` | 0 | `[]` | Coord termina |

**Orden de impresión: 2, 3, 1** = orden de llegada. ✔ (Y **no** es el orden de los identificadores — eso era el ítem (c).)

En el **paso 4 llega P1 mientras P2 está imprimiendo** y no pasa nada malo: se anota y espera. El Coordinador ni se entera hasta la iteración siguiente, y cuando se entera la fila ya tenía a P3 adelante. **El orden se preserva solo.**

## 7. Los errores que se penalizan

**(1) El Coordinador sin `P(libre)`.** Es **el** error del ítem:
```
for i = 1 to N {
   P(hayPedido);
   P(mutexCola); pop(espera, siguiente); V(mutexCola);
   V(priv[siguiente]);
}                              // ← falta esperar a que termine
```
Repartiría los N turnos **de una** y las N personas imprimirían **al mismo tiempo**. Se pierde la exclusión mutua por completo. Y lo peor es que el código *parece* razonable: no hay ningún semáforo mal inicializado, simplemente falta la espera.

> **En criollo:** *el coordinador no reparte turnos, presta la impresora — y no puede prestarla de nuevo hasta que se la devuelvan.*

**(2) La persona durmiéndose con el candado:**
```
P(mutexCola);
push(espera, id);
P(priv[id]);          ← MAL: nunca hizo V(mutexCola)
```
**Deadlock total**: nadie se puede anotar y el Coordinador se cuelga en `P(mutexCola)`.

**(3) `while (true)` en el Coordinador.** Hay un total conocido: atiende **N** y se retira. Con `while(true)` queda colgado en `P(hayPedido)` y el programa no termina.

**(4) Arrastrar el `bool libre` del ítem (b).** No rompe nada, pero es estado muerto: el Coordinador ya no lo consulta. Señala que copiaste (b) sin ver qué se simplificaba.

**(5) Un solo semáforo compartido en vez de `priv[N]`.** `V` despierta a *alguno*: el orden que costó registrar se pierde en el último paso.

## 8. Chequeos de sanidad

- **Con N = 1:** la persona se anota y avisa; el Coordinador da una vuelta, la despierta, espera; ella imprime y avisa; los dos terminan. ✔
- **Contá:** `hayPedido` recibe N `V` y N `P`; `libre`, N y N. Cada `priv[i]` recibe exactamente **un** `V` y **un** `P`. Todo cierra en N.
- **Exclusión mutua:** entre `V(priv[siguiente])` y `P(libre)` el Coordinador está bloqueado ⇒ **nunca hay dos permisos vivos** ⇒ nunca dos personas imprimiendo.
- **Todos terminan:** el Coordinador da N vueltas y cada una despierta a una persona distinta (la cola no repite ids).

## 9. Justificación (esto es lo que suma puntos)

> El orden exigido es el de llegada, que sólo se conoce en ejecución, así que se registra en una cola FIFO compartida, protegida por `mutexCola` porque las N personas escriben en ella. Cada persona se anota, avisa con `V(hayPedido)` y se duerme en su semáforo privado `priv[id]`, inicializado en 0: hace falta uno por persona porque el Coordinador debe despertar a un proceso determinado y no puede suponerse que los semáforos sean FIFO. El Coordinador toma el primero de la cola, le concede el turno con `V(priv[siguiente])` y queda bloqueado en `P(libre)` hasta que esa persona termine. Esa espera es la que provee la exclusión mutua sobre la impresora: al haber un único repartidor bloqueado, no puede existir más de un permiso concedido a la vez, y por eso desaparece la variable `libre` que requería la solución sin coordinador. El Coordinador itera exactamente N veces y se retira, ya que cada persona imprime una sola vez.

---

# e) Cinco impresoras — el Coordinador dice *cuándo* y *cuál*

## 1. La novedad no es el 5: es el "cuál"

Pasar de 1 a 5 impresoras sería trivial: un contador en 5. Lo que cambia el problema es *"…**y cuál** debe usar"*:

> **Un semáforo transmite un *instante*, no un *dato*.**
> `V(priv[id])` dice "ya podés". No tiene forma de decir "ya podés, **con la impresora 3**". Los semáforos no llevan carga.

Las dos cosas que el Coordinador comunica viajan por **canales distintos**:

| Qué comunica | Por dónde viaja |
|---|---|
| **cuándo** (el instante) | El semáforo `priv[id]` |
| **cuál** (el dato) | Una variable compartida: `int asignada[N]` |

Y el orden importa: **el Coordinador escribe el dato ANTES de dar la señal.**

```
asignada[sig] = imp;      // primero el dato
V(priv[sig]);             // después el aviso
```

Es la regla de siempre: **se avisa después de hacer, nunca antes.**

## 2. El Coordinador ya no puede preguntar "¿está libre?"

En (d) alcanzaba con `P(libre)`: *"esperá a que alguien termine"*, daba igual quién. Ahora necesita saber **cuáles** están libres, y choca con otra prohibición:

> **No se puede consultar el valor de un semáforo.** Así que `sem impresora[5] = ([5] 1)` no le sirve: no puede preguntarles cuál está en 1.

La solución es **contador + estructura**:

| Pregunta | Herramienta |
|---|---|
| **¿Hay alguna libre?** (esperar) | `sem hayLibre = 5` — un contador |
| **¿Cuál?** (identidad) | `cola libres`, que arranca con `{1,2,3,4,5}` |

El semáforo da la **espera**; la cola da la **identidad**. El contador no es redundante: **es el único mecanismo de bloqueo que existe**, porque no podés hacer "esperá hasta que la cola no esté vacía".

## 3. ¿Quién devuelve la impresora?

| | El Coordinador recibe la devolución | **La persona devuelve al pozo** |
|---|---|---|
| Cómo | La persona le avisa *cuál* liberó; él la reencola | La persona hace `push(libres, imp)` ella misma |
| Qué hace falta | **Otro** canal de datos (cola de devoluciones + su mutex) | Nada nuevo: reusa `libres` y `mutexLibres` |

**Conviene la segunda:** como las impresoras son **intercambiables desde el punto de vista del pozo**, al Coordinador no le importa *cuál* volvió, sino que **haya** una. Dejando que la persona la devuelva sola, **el Coordinador nunca necesita enterarse de cuál se liberó.** Te ahorrás un canal entero.

## 4. Solución (e)

```
// PRECONDICIONES: N > 0;  cada persona imprime exactamente una vez
//                 la cola 'libres' contiene {1,2,3,4,5} antes de crear los procesos

sem mutexCola   = 1;            // protege la cola de personas
sem hayPedido   = 0;            // Persona → Coordinador: "llegué"
sem hayLibre    = 5;            // cuántas impresoras libres hay  (¡ya no es 0!)
sem mutexLibres = 1;            // protege el pozo de impresoras
sem priv[N]     = ([N] 0);      // Coordinador → Persona id: "es tu turno"

cola espera;                    // ids de personas, en orden de llegada
cola libres;                    // ids de impresoras libres: {1..5}
int asignada[N];                // CUÁL impresora le tocó a cada persona

Process Persona[id: 1..N]
{ int imp;                                    // LOCAL
  P(mutexCola);
    push(espera, id);                         // me anoto
  V(mutexCola);
  V(hayPedido);                               // aviso que hay uno más

  P(priv[id]);                                // espero a que ME llamen
  imp = asignada[id];                         // leo CUÁL me tocó

  Imprimir(documento, imp);                   // uso ESA impresora

  P(mutexLibres);
    push(libres, imp);                        // la devuelvo al pozo
  V(mutexLibres);
  V(hayLibre);                                // aviso que hay una libre más
}

Process Coordinador
{ int sig, imp;
  for i = 1 to N {                            // atiendo a los N y me retiro
     P(hayPedido);                            // espero que haya alguien anotado
     P(hayLibre);                             // espero que haya una impresora libre

     P(mutexLibres);  pop(libres, imp);  V(mutexLibres);     // cuál doy
     P(mutexCola);    pop(espera, sig);  V(mutexCola);       // a quién

     asignada[sig] = imp;                     // el DATO primero
     V(priv[sig]);                            // la SEÑAL después
  }
}
```

## 5. Las tres decisiones que hay que saber defender

### (a) `hayLibre` arranca en 5, no en 0

Es el cambio que habilita el paralelismo real. En (d) el Coordinador quedaba bloqueado después de cada asignación y repartía de a uno. Acá reparte **cinco turnos seguidos** antes de tener que esperar.

> **`hayLibre` no es una señal, es un contador de recursos** — por eso arranca en 5. Es el mismo tipo de semáforo que usarías para "3 detectores" o "6 usuarios como máximo".

### (b) `asignada[N]` es un arreglo, y no lleva mutex

**¿Por qué un arreglo y no una sola variable?** Porque hasta **cinco asignaciones pueden estar en vuelo a la vez**. Con una sola variable, el Coordinador escribiría "impresora 2 para la persona 7" y, antes de que la 7 la leyera, la pisaría con "impresora 3 para la persona 4": dos personas leerían el mismo número y se pelearían por la misma impresora.

**¿Y no hay que protegerlo?** **No.** Contá los que tocan cada casillero:

> `asignada[id]` tiene **un solo escritor** (el Coordinador) y **un solo lector** (la persona `id`). Y nunca se tocan a la vez: el Coordinador escribe **antes** del `V(priv[id])` y la persona lee **después** del `P(priv[id])`, así que el propio semáforo ordena las dos operaciones. Un candado ahí no protegería nada.

### (c) `P(hayPedido)` antes que `P(hayLibre)`

Los dos órdenes son **correctos** (el Coordinador no retiene ningún candado mientras se bloquea, así que no hay deadlock en ninguno). Pero el de arriba se lee mejor y evita sacar una impresora del pozo cuando no hay nadie que la quiera: **primero verificás que haya cliente, después reservás el recurso.**

Lo que **sí** sería un error grave es tomar un mutex y después bloquearse:
```
P(mutexLibres);
P(hayLibre);        ← MAL: te dormís con el candado del pozo en la mano
```
Nadie podría devolver una impresora, porque devolver también necesita `mutexLibres`. **Deadlock.** Vale la regla de siempre: **contador primero, candado después.**

## 6. La traza

**5 impresoras, 6 personas** que llegan en orden 1…6. Lo interesante es el sexto.
Inicial: `hayPedido=0`, `hayLibre=5`, `libres={1,2,3,4,5}`, `espera=[]`.

| # | Quién | Qué pasa | `hayLibre` | `libres` | `espera` |
|---|---|---|---|---|---|
| 1 | **P1…P5** | se anotan y avisan | 5 | `{1,2,3,4,5}` | `[1,2,3,4,5]` |
| 2 | **Coord** | vueltas 1–5: asigna **1→P1, 2→P2, 3→P3, 4→P4, 5→P5** | **0** | `{}` | `[]` |
| 3 | — | las cinco imprimen **en paralelo** | 0 | `{}` | `[]` |
| 4 | **P6** | llega, se anota, avisa; `P(priv[6])` → **duerme** | 0 | `{}` | `[6]` |
| 5 | **Coord** | vuelta 6: `P(hayPedido)` ✓ … `P(hayLibre)` → **se bloquea** | 0 | `{}` | `[6]` |
| 6 | **P3** | termina; `push(libres,3)`; `V(hayLibre)` | 1 | `{3}` | `[6]` |
| 7 | **Coord** | despierta; `pop`→**3**; `pop`→**6**; `asignada[6]=3`; `V(priv[6])` | 0 | `{}` | `[]` |
| 8 | **P6** | despierta, lee `asignada[6] = 3`, imprime en la **impresora 3** | 0 | `{}` | `[]` |
| 9 | **Coord** | terminó las N vueltas → **se retira** | | | |

Las dos cosas para mirar:

- **En el paso 3 hay cinco personas imprimiendo a la vez.** Eso es lo que (e) agrega respecto de (d).
- **En el paso 5 el Coordinador se bloquea en `P(hayLibre)`, no en `P(hayPedido)`.** Hay un pedido esperando; lo que falta es una impresora. Los dos `P` frenan por motivos distintos: por eso hacen falta los dos.

### Una precisión sobre "el orden de llegada"

Con cinco impresoras en paralelo, *"quién imprime primero"* deja de estar bien definido. Lo que la solución garantiza —y es lo que el enunciado pide— es que **el Coordinador concede los turnos en orden de llegada**. El paso 7 lo muestra: la impresora liberada se la lleva P6, que venía esperando, y no alguien que llegara después.

## 7. Qué se simplificó respecto de (d)

Desapareció el **`P(libre)` del final del loop**. En (d) el Coordinador tenía que esperar a que la persona terminara antes de seguir; acá no espera a nadie: reparte mientras haya impresoras, y cuando se le acaban `P(hayLibre)` lo frena solo.

Como consecuencia, **el Coordinador se retira apenas concedió los N turnos**, sin quedarse a ver terminar a los últimos — total, ellos devuelven la impresora solos.

## 8. Los errores que se penalizan

**(1) Reusar el `sem libre = 0` del ítem (d).** Con un semáforo binario y un `P(libre)` después de cada asignación, el Coordinador sigue repartiendo **de a uno**: cuatro impresoras quedan paradas. Funciona, pero tira el enunciado a la basura.

**(2) Contador sin cola de identidades.** `sem hayLibre = 5` sólo dice **cuántas** hay libres, no **cuál**. Y no podés preguntarle al semáforo. Sin `libres`, el Coordinador no tiene qué escribir en `asignada[sig]`.

**(3) Una sola variable `asignada` en vez de `asignada[N]`.** Con hasta cinco asignaciones en vuelo, el Coordinador pisa el valor antes de que la persona anterior lo lea.

**(4) `V(priv[sig])` antes de escribir `asignada[sig]`.** La persona puede despertar y leer basura. **El dato va primero, siempre.**

**(5) La persona que no devuelve la impresora** (falta el `push(libres, imp)` o el `V(hayLibre)`). Las impresoras se "pierden": después de cinco personas el Coordinador queda bloqueado para siempre.

**(6) Dormirse con `mutexLibres` tomado.** Deadlock: devolver una impresora necesita ese mismo candado.

## 9. Chequeos de sanidad

- **Con 1 impresora en vez de 5** (`hayLibre = 1`, `libres = {1}`) la solución **colapsa exactamente en el ítem (d)**. Es el mejor control de que (e) está bien derivado.
- **Contá:** en todo momento `|libres|` + `personas imprimiendo` = **5**, y `hayLibre` coincide con `|libres|`.
- **Cada `priv[i]`** recibe exactamente un `V` y un `P`. Cada `asignada[i]` se escribe una vez y se lee una vez.
- **Ninguna impresora en dos manos:** sale del pozo con un `pop` y sólo vuelve con el `push` de quien la usó. Mientras está afuera, el Coordinador no la puede asignar de nuevo.

## 10. Justificación (esto es lo que suma puntos)

> El Coordinador debe comunicar dos cosas distintas —el instante y la identidad de la impresora— y un semáforo sólo transmite el instante, así que el dato viaja por el arreglo compartido `asignada[N]`, escrito **antes** del `V(priv[sig])` para que la persona no lea un casillero sin inicializar. No requiere exclusión mutua porque cada casillero tiene un único escritor (el Coordinador) y un único lector (la persona), ordenados por el propio semáforo privado. Para administrar las cinco impresoras se usa el patrón contador + estructura: `hayLibre` inicializado en 5 provee la espera —no se puede esperar sobre una cola ni consultar el valor de un semáforo— y la cola `libres` provee la identidad. Se inicializa en 5 y no en 0 porque es un contador de recursos, no una señal, y eso es lo que permite que hasta cinco personas impriman en paralelo. La devolución la hace la propia persona sobre el pozo compartido, con lo cual el Coordinador nunca necesita enterarse de cuál impresora se liberó. El orden de llegada se preserva porque el Coordinador extrae de la cola de personas en FIFO; con cinco impresoras concurrentes, lo que se garantiza es el orden en que se **conceden** los turnos.

---

# El Ejercicio 6 completo, de un vistazo

| Ítem | Qué pide | Mecanismo | Lo que enseña |
|---|---|---|---|
| **a** | Una impresora, sin orden | `sem mutex = 1` | Exclusión mutua pura. Sin `while(true)`, y `Imprimir()` **adentro** del candado porque el candado **es** la impresora |
| **b** | Orden de llegada, sin procesos extra | cola + `priv[N]` + `sem e = 1` | **Passing the Baton.** El recurso no se libera, se **entrega**. `libre` nunca vuelve a `true` durante el traspaso |
| **c** | Orden por identificador | `sem turno[N] = ([1] 1, [N-1] 0)` | El orden estático **ya está escrito**: es una posta. Ni cola ni mutex |
| **d** | Orden de llegada, con Coordinador | + `hayPedido`, `libre`, Coordinador | El estado "ocupada" se vuelve **el program counter del Coordinador** bloqueado en `P(libre)` |
| **e** | 5 impresoras, dice cuál | + `hayLibre = 5`, cola `libres`, `asignada[N]` | Un semáforo transmite un **instante**, no un **dato**. Contador para esperar, cola para identificar |

> **El hilo que los une:** **(b) y (d) resuelven el mismo problema con y sin árbitro.** Cuando el enunciado prohíbe procesos extra, el traspaso lo hacen entre los que compiten (Passing the Baton); cuando te da el Coordinador, lo centralizás. Comparar esos dos ítems es la mejor forma de entender qué es realmente Passing the Baton: **el coordinador, repartido entre los procesos.**

# Los tres chequeos del corrector

1. **¿Hay `while(true)` donde no corresponde?** El enunciado dice "un trabajo cada una": los procesos Persona **terminan**, y el Coordinador atiende exactamente N y se retira.
2. **¿Los semáforos privados son un arreglo inicializado en 0?** Cada vez que hay que despertar a un proceso **determinado**, hace falta un timbre con nombre: un semáforo compartido despierta a *alguno* y rompe el orden.
3. **¿Quién garantiza la exclusión mutua, y está explicado?** En (a) el `mutex`; en (b) el protocolo de traspaso (`libre` nunca vuelve a `true`); en (c) el único permiso que circula; en (d) el Coordinador bloqueado en `P(libre)`; en (e) el pozo de impresoras. Si no podés nombrarlo, no está.
