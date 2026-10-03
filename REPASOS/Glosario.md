# Glosario — Programación Concurrente

> Referencia rápida. Teorías 1, 2 y 3. Una línea por término.

---

## Notación

| Símbolo | Qué es |
|---|---|
| `< S >` | Acción atómica **incondicional** — sólo exclusión mutua |
| `< await (B); S >` | Acción atómica **condicional** — demora hasta `B`, después ejecuta `S` indivisible |
| `co S1 // S2 oc` | Ejecuta concurrentemente y **espera** a que todas terminen (*fork-join*) |
| `process P` | Ejecuta concurrentemente **en background**, no bloquea al que lo crea |
| `process P[i: 1..n]` | **n procesos con el mismo código**, distinguidos sólo por el valor de `i` |
| `if B1 → S1 □ B2 → S2 fi` | Guardas. Elección **no determinística**. Si ninguna es true, no hace nada |
| `do B1 → S1 □ ... od` | Igual pero itera **hasta que todas las guardas sean falsas** |
| `skip` | Termina de inmediato sin efecto. Cuerpo típico de un spin: `do (not B) → skip od` |

---

## Conceptos base (Teoría 1)

| Término | Qué es |
|---|---|
| **Concurrencia** | Concepto de **software**: varios procesos, su comunicación y su sincronización. No depende del hardware |
| **Paralelismo** | Ejecución concurrente en **múltiples procesadores**, para reducir el tiempo |
| **Proceso** | Un único flujo de control secuencial, con espacio de direcciones y recursos propios |
| **Hilo / thread** | Proceso liviano: PC y pila propios, pero **comparte** espacio de direcciones y recursos |
| **No determinismo** | La misma entrada puede producir distinta salida, según el interleaving |
| **Acción atómica** | Transformación de estado **indivisible**: sus estados intermedios son invisibles para los demás |
| **Grano fino** | Acción atómica implementada **por hardware** (una instrucción máquina) |
| **Grano grueso** | Secuencia de acciones de grano fino que **aparenta** ser indivisible |
| **Interleaving** | El intercalado de las acciones atómicas de los distintos procesos |
| **Historia** (*trace*) | Una ejecución concreta con un interleaving particular. Hay muchísimas y **no todas son válidas** |
| **Referencia crítica** | Referencia a una variable **que otro proceso modifica** |
| **ASV** (*a lo sumo una vez*) | **A lo sumo una variable compartida, referenciada a lo sumo una vez** → la asignación parece atómica y no hace falta protegerla |
| **Interferencia** | Un proceso toma una acción que **invalida las suposiciones** de otro |
| **Sección crítica (SC)** | Fragmento que accede a un recurso compartido y no puede ejecutarlo más de un proceso a la vez |
| **SNC** | Sección **no** crítica: todo lo demás del ciclo del proceso |
| **Exclusión mutua** | Sincronización que asegura **un solo proceso a la vez** en la SC |
| **Sincronización por condición** | **Bloquear** un proceso hasta que se cumpla una condición. Restringe las historias posibles |
| **Deadlock** | Dos o más procesos esperan indefinidamente que el otro libere un recurso |
| **Inanición** | Un proceso puntual **nunca** consigue el recurso, aunque el sistema en conjunto progrese |
| **Invariante** | Predicado verdadero en **todo** estado observable del programa. Es la herramienta para demostrar safety |

### Las 4 condiciones de deadlock

Recursos reusables serialmente · adquisición incremental · no-preemption · espera cíclica.
*(Si falla alguna, no puede haber deadlock. La que más se usa: si nadie espera **con un recurso en la mano**, falla la adquisición incremental.)*

### Propiedades y fairness

| Término | Qué es |
|---|---|
| **Safety** | "Nada malo ocurre". Se demuestra por **invariante o contradicción** |
| **Liveness** | "Eventualmente algo bueno ocurre". **Siempre depende del scheduling** |
| **Fairness incondicional** | Toda acción atómica **incondicional** elegible eventualmente se ejecuta |
| **Fairness débil** | Incondicional **+** toda acción condicional se ejecuta si su guarda se hace true **y permanece** true |
| **Fairness fuerte** | Incondicional **+** se ejecuta si su guarda se hace true **con infinita frecuencia** |
| | *Round-robin es práctica y débilmente fair, pero **no** fuertemente fair* |

---

## Sección crítica (Teoría 2)

### Las 4 propiedades a cumplir

| # | Propiedad | En criollo | Tipo |
|---|---|---|---|
| 1 | Exclusión mutua | no entran dos | safety |
| 2 | Ausencia de deadlock | no se traban todos | safety |
| 3 | Ausencia de demora innecesaria | no me hagas esperar si no hay nadie adentro | safety |
| 4 | Eventual entrada | no me dejes esperando para siempre | **liveness** |

*La 2 habla del **conjunto**; la 4, del **individuo**. La diferencia entre ambas es la **inanición**.*

### Mecanismos

| Término | Qué es |
|---|---|
| **Busy waiting** (*spin*) | Chequear repetidamente una condición hasta que sea verdadera. Funciona en cualquier procesador, pero desperdicia CPU en multiprogramación |
| **Lock** | Booleano compartido que representa "el recurso está tomado" |
| **Cambio de variables** | Técnica para pasar de 2 a n procesos: reemplazar `in1`/`in2` por `lock = in1 ∨ in2` |
| **TS** (*Test-and-Set*) | `TS(ok)`: pone `ok` en **true** y devuelve el **valor anterior**. Instrucción de hardware |
| **Spin lock** | `while (TS(lock)) skip;` — la solución de grano fino más simple. Cumple 1, 2 y 3 |
| **Test-and-Test-and-Set** | Mirar antes de hacer TS, para no escribir de gusto. Reduce **contención de memoria** |
| **FA** (*Fetch-and-Add*) | `FA(var, incr)`: suma `incr` a `var` y devuelve el **valor anterior**. Instrucción de hardware |
| **Contención de memoria** | Degradación por muchos procesos escribiendo la misma posición (invalida caches) |

### Los tres algoritmos fair

Existen **sólo** para conseguir la propiedad 4 con scheduling apenas **débilmente fair**.

| Algoritmo | Idea | ¿Instrucción especial? |
|---|---|---|
| **Tie-Breaker** (Peterson) | Se demora **al último en llegar**; `ultimo` rompe empates | Ninguna |
| **Ticket** | Sacás número y esperás que te llamen (`turno[i] == proximo`) | Fetch-and-Add |
| **Bakery** | Te auto-asignás `max + 1` y esperás a ser el menor. **Los procesos se comparan entre sí**, no contra un global | Ninguna |

---

## Barreras (Teoría 2)

| Término | Qué es |
|---|---|
| **Barrera** | Punto de demora al que deben llegar **todos** antes de que **cualquiera** siga. *La SC separa; la barrera junta* |
| **Contador compartido** | Contar los que llegan. **No sirve reutilizable**: no hay momento seguro para resetear el contador |
| **Flags y coordinador** | Un flag `arribo[i]` y `continuar[i]` por proceso, más un proceso Coordinador. Cada worker espera **una sola** variable: la suya |
| **Principio de flags** | (1) El que **espera** un flag es el que lo **limpia**. (2) No se prende un flag hasta saber que está limpio |
| **Combining tree** | Cada worker es también coordinador de un subárbol. Arribos suben, continuar baja. **O(log n)**, pero roles distintos |
| **Butterfly** (barrera simétrica) | log₂n etapas; en la etapa `s` cada uno sincroniza con el que está a distancia `2^(s-1)`. **O(log n) y código idéntico para todos** |

---

## Semáforos (Teoría 3 / Clase 3)

| Término | Qué es |
|---|---|
| **Semáforo** | TAD con **dos operaciones atómicas** (`P` y `V`) sobre un **entero no negativo**. Dijkstra, 1968 |
| **`P(s)`** | `< await (s > 0) s = s-1 >` — **demora** hasta que ocurra el evento |
| **`V(s)`** | `< s = s+1 >` — **señala** la ocurrencia de un evento |
| **Semáforo general** (*counting*) | Su valor puede ser cualquier entero ≥ 0 |
| **Semáforo binario** | Vive en {0,1}. Su `V` **tiene guarda**: `< await (b < 1) b = b+1 >` |
| **Mutex** | Semáforo inicializado en **1**. `P` y `V` los hace **el mismo** proceso |
| **Semáforo de señalización** | Inicializado en **0**. El `V` y el `P` los hacen procesos **distintos** |
| **Contador de recursos** | Inicializado en **n**. Cuenta unidades libres del recurso |
| **SBS** (*Split Binary Semaphore*) | Semáforos binarios con invariante `0 ≤ b1 + ... + bn ≤ 1`. Un binario "partido": si cada camino empieza con un `P` sobre uno y termina con un `V` sobre otro, el medio ejecuta **con exclusión mutua gratis** |
| **Semáforo privado** | `s` es privado si **exactamente un proceso** ejecuta `P` sobre él. Sirve para despertar procesos individuales |
| **Passing the Baton** | Técnica general para implementar `await`. El que está en la SC tiene el **baton**; al salir se lo **pasa en la mano** a un proceso que espera (`V(bj)`, **sin** `V(e)`), o lo libera (`V(e)`) si no hay nadie |
| **`SIGNAL`** | El bloque de salida de Passing the Baton: elige a quién pasarle el baton, o lo suelta |
| **Exclusión mutua selectiva** | Procesos que compiten por conjuntos **superpuestos** de variables compartidas (filósofos, lectores/escritores) |

### El valor inicial te dice el rol

| Init | Rol |
|---|---|
| `= 1` | **mutex** |
| `= 0` | **señalización** |
| `= n` | **contador de recursos** |

### Trampas de semáforos

- **`P` y `V` del mismo proceso = mutex; de procesos distintos = señalización.**
- **Inicializar siempre.** `sem s;` no existe; `sem mutex = 0` es deadlock.
- **Avisá antes de esperar** (`V` y después `P`). Al revés, deadlock.
- **Primero el permiso, después el candado:** nunca `P(mutex)` antes que `P(condición)`.
- **El semáforo NO es FIFO** — la materia lo prohíbe explícitamente. Si el problema pide orden, lo construís vos.
- **En `SIGNAL`, el que despierta NO vuelve a hacer `P(e)`:** recibe el baton ya tomado.

---

## Atajos mentales

- Un `< >` sirve para **pegar dos accesos**. Si hay uno solo, no hay nada que pegar.
- Cada `< >` **reduce la concurrencia**: usar locales siempre que se pueda.
- **La operación larga nunca adentro de la reserva** (`produce`, `consume`, `usar`, `Imprimir` van afuera).
- El `< >` protege la estructura compartida, **no lo que sacaste de ella** → esa variable va **local**.
- Si podés preguntarle el estado **a la estructura misma**, no lleves un contador aparte.
- `while (TS(lock)) skip` testea el valor **viejo**: si estaba ocupado sigo girando; si estaba libre, ya lo tomé.
