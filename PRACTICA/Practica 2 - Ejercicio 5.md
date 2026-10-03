# Práctica 2 — Ejercicio 5

> **Enunciado.** En una empresa de logística de paquetes existe una **sala de contenedores** donde se preparan las entregas. Cada contenedor puede almacenar **un paquete** y la sala cuenta con capacidad para **N contenedores**. Resuelva considerando las siguientes situaciones:
> **a)** La empresa cuenta con 2 empleados: un empleado **Preparador** que se ocupa de preparar los paquetes y dejarlos en los contenedores; un empleado **Entregador** que se ocupa de tomar los paquetes de los contenedores y realizar las entregas. Tanto el Preparador como el Entregador trabajan de a un paquete por vez.
> **b)** Modifique la solución a) para el caso en que haya **P empleados Preparadores**.
> **c)** Modifique la solución a) para el caso en que haya **E empleados Entregadores**.
> **d)** Modifique la solución a) para el caso en que haya **P Preparadores y E Entregadores**.

---

## 0. Lo que el enunciado está diciendo

Empecemos por la versión **secuencial**, con un solo empleado que hace todo:

```
Process Empleado
{ while (true) {
    p = Preparar();      // trabajo largo → delay(tp)
    Entregar(p);         // trabajo largo → delay(te)
  }
}
```

Fijate en algo: **acá la sala de contenedores no hace falta para nada.** El empleado prepara el paquete, lo tiene en la mano, y lo entrega.

Los contenedores aparecen recién cuando **son dos personas distintas**. Y esa es toda la idea del ejercicio:

> **La sala de contenedores es un *buffer*: existe para desacoplar dos ritmos.**
> El Preparador puede tardar 2 minutos por paquete y el Entregador 10. Si tuvieran que encontrarse mano a mano para cada paquete, el rápido se quedaría esperando al lento todo el tiempo. Con N contenedores en el medio, el Preparador puede adelantarse hasta N paquetes antes de tener que frenar.

Eso también te dice de dónde sale la **capacidad N**: no es un capricho del enunciado, es **cuánto se puede adelantar el rápido antes de bloquearse**.

---

## 1. De la secuencial a la concurrente: las dos esperas

Partimos el `while` de arriba en dos procesos, uno con cada mitad:

```
Preparador                    Entregador
while(true) {                 while(true) {
  p = Preparar();               p = sala[?];
  sala[?] = p;                  Entregar(p);
}                             }
```

Y ahí aparecen los dos agujeros, que son **las dos esperas**:

| # | ¿Quién espera? | ¿Por qué? |
|---|---|---|
| 1 | El **Preparador** | La sala está **llena**: los N contenedores tienen paquete. No tiene dónde dejar el suyo |
| 2 | El **Entregador** | La sala está **vacía**: no hay ningún paquete todavía. No tiene qué entregar |

### Dos esperas ⇒ dos semáforos

La regla que manda en todo el ejercicio:

> **Un semáforo sólo sabe esperar una cosa.** No existe "esperá hasta que se cumpla tal condición": lo único que podés hacer es `P` sobre *un* semáforo. Entonces **cada espera distinta necesita su propio semáforo.**

Y como la práctica además prohíbe mirar el valor de un semáforo, tampoco podés preguntarle "¿cuántos contenedores quedan?". Cada semáforo tiene que **ser** exactamente la espera que te interesa.

```
sem vacios = N;      // cuántos contenedores LIBRES hay  → lo espera el Preparador
sem llenos = 0;      // cuántos contenedores CON PAQUETE → lo espera el Entregador
```

Arrancan así porque **la sala empieza vacía**: N contenedores libres, 0 paquetes. Son **complementarios**; en reposo vale siempre:

```
vacios + llenos = N
```

Y trabajan **cruzados**, que es lo que puede marear al principio:

| | espera en… | avisa en… |
|---|---|---|
| **Preparador** | `P(vacios)` — "necesito lugar" | `V(llenos)` — "dejé un paquete" |
| **Entregador** | `P(llenos)` — "necesito paquete" | `V(vacios)` — "liberé un contenedor" |

> **En criollo:** *cada uno espera lo que consume y avisa lo que produce.* El Preparador **consume lugar** y **produce paquetes**; el Entregador **consume paquetes** y **produce lugar**.

### El índice: qué contenedor usar

Los semáforos te dicen **que hay** un contenedor libre, pero no **cuál**. Falta el índice, y cada bando lo hace avanzar en círculo:

```
pos_prep : 0, 1, 2, ..., N-1, 0, 1, ...     ← lo mueven los Preparadores
pos_entr : 0, 1, 2, ..., N-1, 0, 1, ...     ← lo mueven los Entregadores
```

La sala funciona como una **cinta circular**: se carga adelante y se descarga atrás, siempre en el mismo orden. **Esa suposición de orden es la clave de los ítems (b) y (c)** — todo lo que la rompa, rompe la solución.

---

# a) Un Preparador, un Entregador

## La pregunta que vale el ítem

> **Con un Preparador y un Entregador, ¿hace falta algún `mutex`?**

Es la pregunta natural: `sala` es una estructura compartida, y uno escribe y el otro lee. El reflejo dice "poné un candado".

**La respuesta es NO.** Y el motivo tiene dos patas:

**(i) Los índices no son compartidos.** `pos_prep` lo lee y lo escribe **únicamente** el Preparador; `pos_entr`, únicamente el Entregador. Son variables **locales**. Una variable con un solo dueño no tiene condición de carrera: no hay con quién competir.

**(ii) Nunca tocan la misma celda al mismo tiempo — y de eso ya se encargan los semáforos.** Mirá la cadena para un contenedor cualquiera `i`:

```
El Entregador sólo llega a leer sala[i]  ⟸ pasó un P(llenos)
                                         ⟸ alguien hizo V(llenos)
                                         ⟸ el Preparador YA terminó de escribir sala[i]
```

y en el otro sentido:

```
El Preparador sólo llega a escribir sala[i] ⟸ pasó un P(vacios)
                                            ⟸ alguien hizo V(vacios)
                                            ⟸ el Entregador YA terminó de leer sala[i]
```

Cada contenedor **alterna dueño**: vacío → lo escribe el Preparador → lleno → lo lee el Entregador → vacío. En ningún instante hay dos manos sobre la misma celda. Los semáforos `vacios`/`llenos` no están sólo contando: están **dando la exclusión mutua gratis**, celda por celda.

> **Por eso el ejercicio tiene cuatro ítems.** El (a) es el único caso donde te zafás sin mutex, justamente porque hay **uno de cada**. Apenas haya P Preparadores, los índices dejan de tener un solo dueño. Los ítems (b), (c) y (d) son exactamente ese descubrimiento.

## Solución (a)

```
// PRECONDICIONES: N > 0
//                 la sala arranca vacía (ningún contenedor tiene paquete)

paquete sala[0..N-1];        // los N contenedores
sem vacios = N;              // contenedores libres   (espera el Preparador)
sem llenos = 0;              // contenedores ocupados (espera el Entregador)

Process Preparador
{ paquete p;
  int pos = 0;                          // LOCAL: mi próximo contenedor a llenar
  while (true) {
    p = Preparar();                     // delay(tp) — AFUERA: no ocupo la sala
    P(vacios);                          // espero que haya un contenedor libre
    sala[pos] = p;                      // dejo el paquete
    pos = (pos + 1) mod N;              // avanzo en círculo
    V(llenos);                          // aviso: hay un paquete más
  }
}

Process Entregador
{ paquete p;
  int pos = 0;                          // LOCAL: mi próximo contenedor a vaciar
  while (true) {
    P(llenos);                          // espero que haya un paquete
    p = sala[pos];                      // lo saco
    pos = (pos + 1) mod N;              // avanzo en círculo
    V(vacios);                          // aviso: quedó un contenedor libre
    Entregar(p);                        // delay(te) — AFUERA: ya liberé el contenedor
  }
}
```

### Por qué cada pieza está donde está

| Línea | Por qué |
|---|---|
| `Preparar()` **antes** del `P(vacios)` | El paquete se arma *en la mano*, no dentro del contenedor. Si hicieras `P(vacios)` primero, tendrías un contenedor **reservado y vacío** durante todo el armado: desperdiciás capacidad |
| `P(vacios)` antes de escribir | Es la única forma de esperar. Sin eso, con la sala llena pisarías un paquete todavía no entregado |
| `V(llenos)` **después** de escribir | Si avisás antes, el Entregador puede leer un contenedor que todavía no tiene el paquete |
| `V(vacios)` **antes** de `Entregar()` | El contenedor queda libre apenas sacaste el paquete. Dejarlo "ocupado" durante la entrega inmoviliza un contenedor por nada |
| `p` y `pos` **locales** | Un solo dueño cada una ⇒ sin carrera y sin mutex |

### Justificación (esto es lo que suma puntos)

> La sala de N contenedores es un buffer que desacopla los ritmos del Preparador y del Entregador. Cada uno tiene una espera distinta —lugar libre y paquete disponible— y como un semáforo sólo permite esperar sobre sí mismo, se usan dos: `vacios` inicializado en N y `llenos` en 0, que cumplen `vacios + llenos = N`. Cada proceso hace `P` sobre el recurso que consume y `V` sobre el que produce. No hace falta exclusión mutua: los índices `pos` son locales (un único dueño) y los propios semáforos garantizan que el Preparador y el Entregador nunca accedan al mismo contenedor a la vez, porque cada acceso está precedido por el `V` del otro sobre esa misma celda. Las operaciones largas (`Preparar` y `Entregar`) quedan fuera del tramo en que se ocupa un contenedor, para no inmovilizar capacidad de la sala.

### Los errores que se penalizan

**(1) Un solo semáforo.** Intentar resolverlo con `sem sala = N` únicamente. No alcanza: ese semáforo sabe frenar al Preparador cuando la sala está llena, pero **no tiene con qué frenar al Entregador** cuando está vacía — y el Entregador entonces lee basura de `sala[pos]`. Dos esperas distintas, dos semáforos.

**(2) Invertir un `P` con su `V`.**
```
V(llenos);          ← MAL: aviso antes de tener el paquete puesto
sala[pos] = p;
```
El Entregador puede colarse en el medio y llevarse un contenedor vacío. **Se avisa después de hacer, nunca antes.**

**(3) `Entregar()` antes del `V(vacios)`.**
```
p = sala[pos];
Entregar(p);        ← el contenedor sigue contando como ocupado
V(vacios);
```
No rompe nada —el resultado es correcto— pero con entregas largas el Preparador se bloquea mucho antes de lo necesario. Es el mismo defecto que poner la operación larga adentro de una sección crítica: **retenés un recurso mientras hacés algo que no lo necesita.**

### Chequeos de sanidad

- **Contá:** en todo momento, contenedores libres + contenedores con paquete = **N**. Y `vacios + llenos = N`.
- **Con `N = 1`** la sala degenera: `vacios=1`, `llenos=0`, y los dos se alternan estrictamente — preparar, entregar, preparar, entregar. Es justo el caso "mano a mano" sin desacople, y la solución lo hace bien.
- **Todo `P` tiene su `V`, pero en el OTRO proceso** (no en el mismo, como en un mutex). Si los ves de a pares dentro del mismo proceso, algo está mal modelado.
- **Nadie se cuelga para siempre:** si el Preparador está bloqueado en `P(vacios)` es porque hay N paquetes esperando, o sea que el Entregador tiene trabajo y va a hacer `V(vacios)`. Y viceversa.

---

# b) P Preparadores, un Entregador

## 1. Por qué el índice ya no puede ser local

Con un solo Preparador, `pos` era local: un dueño, sin carrera. Ahora son P.

Primera tentación: **que cada Preparador tenga su propio `pos = 0`.** Probalo con P=2:

| Preparador | `pos` | Escribe en |
|---|---|---|
| A | 0 | `sala[0]` |
| B | 0 | `sala[0]` ← **encima del de A** |

Los dos pasaron `P(vacios)` (había N libres, no hay motivo para frenarlos) y los dos escribieron el mismo contenedor. **Un paquete perdido**, y encima se hicieron dos `V(llenos)` por un solo paquete real.

> El índice tiene que ser **uno solo y compartido**, porque representa algo que es de la sala, no del empleado: *cuál es el próximo contenedor a llenar.*

### La alternativa que no sirve (y por qué)

Se podría pensar en **partir la sala**: los contenedores `0..N/P-1` para el Preparador 0, los siguientes para el 1, etc. Así cada uno tendría su índice local de vuelta.

No funciona, y el motivo está del otro lado: el **Entregador recorre la sala en orden circular** `0, 1, 2, …`. Si cada Preparador llena su propia región a su ritmo, el orden en que se llenan los contenedores deja de coincidir con el orden en que el Entregador los visita, y el Entregador se planta en un contenedor vacío mientras hay paquetes esperando en otra región. **El consumidor necesita una secuencia única y ordenada.**

## 2. El mutex, y el orden de los dos `P`

Índice compartido que se lee y se escribe ⇒ sección crítica ⇒ candado:

```
int pos_prep = 0;      // COMPARTIDO entre los P Preparadores
sem mutexP = 1;        // lo protege
```

```
P(vacios);      ← PRIMERO el contador
P(mutexP);      ← DESPUÉS el candado
```

> **La regla: contador primero, candado después.** Nunca te bloqueás esperando un recurso **con el candado en la mano**.
>
> Para ser preciso: en el (b) el orden invertido todavía **no** es deadlock (el único Entregador no necesita `mutexP` para liberar contenedores, así que tarde o temprano destraba). Pero deja a los otros P−1 Preparadores parados en `P(mutexP)` sin ninguna razón — y en el ítem (d), si compartís un único mutex entre ambos bandos, el mismo orden invertido **sí** es deadlock. Aplicá la regla siempre y no tenés que analizar caso por caso.

## 3. La trampa: ¿la escritura va adentro o afuera del mutex?

Dos candidatas.

**Versión A — el paquete se deja adentro del mutex:**
```
P(vacios);
P(mutexP);
  sala[pos_prep] = p;                    // escribo
  pos_prep = (pos_prep + 1) mod N;       // y avanzo
V(mutexP);
V(llenos);
```

**Versión B — "optimizada": adentro sólo el índice.**
```
P(vacios);
P(mutexP);  mi_pos = pos_prep;  pos_prep = (pos_prep + 1) mod N;  V(mutexP);
sala[mi_pos] = p;                        // ← escritura AFUERA
V(llenos);
```

La B se ve mejor: sección crítica cortita y escritura en paralelo. **Y está mal.**

### La traza que la rompe

Sala con N=3, dos Preparadores (A y B), el Entregador con `pos_entr = 0`.

| # | Quién | Qué hace | Estado |
|---|---|---|---|
| 1 | A | toma el mutex, se queda con `mi_pos = 0`, avanza el índice a 1, suelta | `sala[0]` **todavía vacío** |
| 2 | B | toma el mutex, se queda con `mi_pos = 1`, avanza a 2, suelta | |
| 3 | B | escribe `sala[1] = paqB`, hace `V(llenos)` | `llenos = 1` |
| 4 | — | **A todavía no escribió** (se lo comió el scheduler) | `sala[0]` vacío |
| 5 | Entregador | `P(llenos)` ✓ pasa. Lee `sala[pos_entr]` = **`sala[0]`** | **basura** |

El Entregador entrega **un paquete que no existe**, avanza a `pos_entr = 1`, y cuando A finalmente escriba `sala[0]`, **ese paquete no lo levanta nadie hasta la vuelta siguiente** (o nunca, si los contadores ya se desfasaron).

### Por qué la versión A sí funciona

El Entregador se apoya en una suposición: **los contenedores se llenan en orden `0, 1, 2, …`**. Con la versión A, el mutex hace que "reservar el contenedor k" y "escribir el contenedor k" sean **el mismo evento indivisible**. Entonces:

- La k-ésima escritura va siempre al contenedor k, y ocurre antes que la (k+1)-ésima.
- Cada `V(llenos)` ocurre después de su propia escritura.
- Si el Entregador logró pasar **j** veces por `P(llenos)`, es porque hubo **j** escrituras completas, y como van en orden, son exactamente los contenedores `0..j-1`. El que va a leer está garantizado lleno. ✓

> **En criollo:** el aviso `V(llenos)` no sólo tiene que ir *después* de escribir — tiene que respetar el **orden**. Lo único que garantiza las dos cosas a la vez es dejar la escritura adentro del candado.

### ¿Y no se pierde paralelismo?

Un poco, sí — pero lo que queda adentro del mutex es **una asignación**, no trabajo real. `Preparar()` (que es lo caro) sigue **afuera**, y es ahí donde los P Preparadores trabajan de verdad en paralelo. El mutex sólo serializa el instante de "apoyar el paquete en el contenedor".

## Solución (b)

```
// PRECONDICIONES: N > 0, P > 0;  la sala arranca vacía

paquete sala[0..N-1];
sem vacios = N;
sem llenos = 0;
int pos_prep = 0;              // COMPARTIDO entre los Preparadores
sem mutexP = 1;                // protege pos_prep y la escritura

Process Preparador[id: 1..P]
{ paquete p;                                   // LOCAL
  while (true) {
    p = Preparar();                            // delay(tp) — AFUERA, en paralelo
    P(vacios);                                 // espero lugar
    P(mutexP);
      sala[pos_prep] = p;                      // dejo el paquete
      pos_prep = (pos_prep + 1) mod N;         // y avanzo el índice
    V(mutexP);
    V(llenos);                                 // aviso: hay un paquete más
  }
}

Process Entregador                             // IDÉNTICO al del ítem (a)
{ paquete p;
  int pos = 0;                                 // LOCAL: sigue habiendo uno solo
  while (true) {
    P(llenos);
    p = sala[pos];
    pos = (pos + 1) mod N;
    V(vacios);
    Entregar(p);                               // delay(te) — AFUERA
  }
}
```

**El Entregador no cambia una línea.** Eso es señal de que el modelo está bien: los semáforos `vacios`/`llenos` cuentan **contenedores**, no empleados, así que no les importa cuántos Preparadores haya.

---

# c) Un Preparador, E Entregadores

Es **el espejo exacto** del (b). Ahora el que se multiplica es el consumidor, así que el índice que se vuelve compartido es `pos_entr`:

```
int pos_entr = 0;      // COMPARTIDO entre los Entregadores
sem mutexE = 1;
```

Y **la lectura va adentro del mutex**, por el motivo simétrico: quien ahora recorre la sala en orden circular y da por sentado que los contenedores se **liberan** en orden es el Preparador.

### La traza del error espejado

Si un Entregador se llevara sólo el índice adentro del mutex y leyera afuera, con N=3:

| # | Quién | Qué hace | Estado |
|---|---|---|---|
| 1 | Entregador A | se queda con `mi_pos = 0`, avanza a 1, suelta el mutex | **no leyó `sala[0]` todavía** |
| 2 | Entregador B | se queda con `mi_pos = 1`, avanza a 2, suelta | |
| 3 | B | lee `sala[1]`, hace `V(vacios)` | `vacios` sube |
| 4 | Preparador | `P(vacios)` ✓ pasa. Su índice ya dio la vuelta y apunta a 0: escribe **`sala[0]`** | |
| 5 | A | por fin lee `sala[0]` → **el paquete nuevo**, y el viejo se perdió | |

Un paquete **pisado antes de ser entregado** y otro **entregado dos veces**.

## Solución (c)

```
// PRECONDICIONES: N > 0, E > 0;  la sala arranca vacía

paquete sala[0..N-1];
sem vacios = N;
sem llenos = 0;
int pos_entr = 0;              // COMPARTIDO entre los Entregadores
sem mutexE = 1;

Process Preparador                             // IDÉNTICO al del ítem (a)
{ paquete p;
  int pos = 0;                                 // LOCAL
  while (true) {
    p = Preparar();
    P(vacios);
    sala[pos] = p;
    pos = (pos + 1) mod N;
    V(llenos);
  }
}

Process Entregador[id: 1..E]
{ paquete p;                                   // LOCAL: cada uno se lleva el suyo
  while (true) {
    P(llenos);                                 // espero que haya paquete
    P(mutexE);
      p = sala[pos_entr];                      // lo saco
      pos_entr = (pos_entr + 1) mod N;         // y avanzo
    V(mutexE);
    V(vacios);                                 // aviso: contenedor libre
    Entregar(p);                               // delay(te) — AFUERA del mutex
  }
}
```

**Lo que hace que el ítem valga la pena:** `p` es **local** y `Entregar(p)` está **afuera** del mutex. Si la entrega estuviera adentro, los E Entregadores se turnarían de a uno y trabajarían como **uno solo** — la única razón de contratar E entregadores desaparecería.

---

# d) P Preparadores y E Entregadores

Se juntan los dos ítems anteriores. No hay ninguna interacción nueva.

```
// PRECONDICIONES: N > 0, P > 0, E > 0;  la sala arranca vacía

paquete sala[0..N-1];
sem vacios = N;                // contenedores libres
sem llenos = 0;                // contenedores con paquete
int pos_prep = 0;   sem mutexP = 1;      // punta de carga
int pos_entr = 0;   sem mutexE = 1;      // punta de descarga

Process Preparador[id: 1..P]
{ paquete p;
  while (true) {
    p = Preparar();                            // delay(tp) — en paralelo
    P(vacios);
    P(mutexP);
      sala[pos_prep] = p;
      pos_prep = (pos_prep + 1) mod N;
    V(mutexP);
    V(llenos);
  }
}

Process Entregador[id: 1..E]
{ paquete p;
  while (true) {
    P(llenos);
    P(mutexE);
      p = sala[pos_entr];
      pos_entr = (pos_entr + 1) mod N;
    V(mutexE);
    V(vacios);
    Entregar(p);                               // delay(te) — en paralelo
  }
}
```

## Lo único que hay que justificar: dos mutex, no uno

Es la decisión que el corrector busca. Un solo `sem mutex = 1` compartido por ambos bandos también daría un resultado correcto, pero:

| | Dos mutex (`mutexP`, `mutexE`) | Un mutex compartido |
|---|---|---|
| **Paralelismo** | Un Preparador carga mientras un Entregador descarga | Cargar y descargar se **serializan**: la sala se usa de a uno |
| **Sentido** | Son **dos puntas distintas** del buffer, con datos distintos (`pos_prep` vs `pos_entr`) | Un candado para dos cosas que nunca compiten entre sí |
| **Con el orden de los `P` invertido** | Se destraba solo | **Deadlock**: un Preparador espera `vacios` reteniendo el mutex, y el único que puede liberar un contenedor necesita ese mismo mutex |

> **La frase para el oral:** *`mutexP` arbitra entre Preparadores, `mutexE` entre Entregadores; entre un Preparador y un Entregador no hace falta ningún candado, porque `vacios` y `llenos` ya garantizan que nunca tocan el mismo contenedor a la vez.*

## Deadlock: por qué no hay

Ningún proceso hace un `P` **mientras retiene** un candado: entra al mutex, hace dos asignaciones que siempre terminan, y sale. Sin *hold-and-wait*, no hay espera circular posible.

## Chequeos de sanidad

- **Con `P = 1` y `E = 1`**, los dos mutex están siempre libres (no hay con quién competir) y la solución **colapsa exactamente en la del ítem (a)**. Es el mejor control de que (d) está bien derivado.
- **Con `E = 1`** colapsa en (b); **con `P = 1`**, en (c).
- `vacios + llenos = N` se mantiene siempre.
- Todo `P` tiene su `V` — pero distinguí los dos tipos: los de `mutexP`/`mutexE` cierran **en el mismo proceso** (son candados); los de `vacios`/`llenos` cierran **en el proceso del otro bando** (son avisos). Si ves `P(llenos)` y `V(llenos)` juntos en un mismo proceso, algo está mal.

---

## Los cuatro ítems de un vistazo

| Ítem | Procesos | `pos_prep` | `pos_entr` | Mutex | Qué agrega |
|---|---|---|---|---|---|
| **a** | 1 P, 1 E | local | local | ninguno | Los dos semáforos contadores. Un dueño por índice ⇒ sin candados |
| **b** | P P, 1 E | **compartido** | local | `mutexP` | La escritura va **adentro** del mutex, para no romper el orden de llenado |
| **c** | 1 P, E E | local | **compartido** | `mutexE` | Espejo. `Entregar()` **afuera** del mutex, o los E entregan de a uno |
| **d** | P P, E E | compartido | compartido | `mutexP` **+** `mutexE` | **Dos** candados: las puntas son independientes |

---

## Los tres chequeos del corrector

1. **¿Hay dos semáforos contadores, `vacios = N` y `llenos = 0`, usados cruzados?** Con uno solo no se puede frenar a los dos bandos.
2. **¿El acceso a `sala[...]` está adentro del mutex cuando el índice es compartido?** Dejar la escritura (o la lectura) afuera rompe el orden circular del otro bando: paquete perdido o paquete pisado.
3. **¿`Preparar()` y `Entregar()` quedaron afuera de todo tramo que ocupe el contenedor o retenga un candado?** Adentro, la sala de N contenedores rinde como una sola y los P/E empleados rinden como uno.
