# Práctica 3 (Monitores) — Ejercicio 5

> **Enunciado.** En un corralón de materiales se debe atender a **N clientes de acuerdo con el orden de llegada**. Cuando un cliente es llamado para ser atendido, **entrega una lista** con los productos que comprará, y **espera a que alguno de los empleados le entregue el comprobante** de la compra realizada.
> **a)** Resuelva considerando que el corralón tiene un **único empleado**.
> **b)** Resuelva considerando que el corralón tiene **E empleados (E > 1)**. Los empleados **no deben terminar** su ejecución.
> **c)** Modifique la solución (b) considerando que los empleados **deben terminar** su ejecución cuando se hayan atendido todos los clientes.

---

## 0. Primero: quién es proceso y quién es monitor

> **Los que HACEN cosas son procesos. Lo que COMPARTEN es el monitor.**

| | Qué es | Por qué |
|---|---|---|
| **Cliente** | `Process Cliente[1..N]` | Llega, entrega su lista, se va. **Hace** cosas |
| **Empleado** | `Process Empleado[1..E]` | Atiende, procesa listas, entrega comprobantes. **Hace** cosas |
| **Corralón** | `monitor Corralon` | Es el **mostrador**: el lugar donde se encuentran. No hace nada por su cuenta |

```
Process Cliente[1..N]                    Process Empleado[1..E]
        │                                         │
        │        llamadas a procedures            │
        └────────►   monitor Corralon   ◄─────────┘
```

**El monitor es pasivo:** no hace nada hasta que un proceso lo llama. Todo el movimiento lo ponen los procesos.

Viniendo de semáforos: **el monitor es donde ahora vive lo que antes eran las variables compartidas + los semáforos.** Los procesos son los mismos; lo que cambia es que ya no tocan el estado directamente sino llamando a procedures.

---

## 1. ¿Cuántos procedures? La regla de los cortes

Es la pregunta que más cuesta en monitores, y tiene una regla:

> **Cada vez que un proceso tiene que hacer algo AFUERA del monitor en el medio de su secuencia, hay que cortar en dos llamadas.**

Porque no podés quedarte adentro del monitor mientras el otro trabaja: bloquearías a todos.

### La coreografía

```
CLIENTE                                          EMPLEADO
───────                                          ────────
 ┌─ se anota y espera turno ─┐
 │                            │  ◄── lo llama ──  ┌─ espera que haya alguien
 │  (lo despiertan)           │                   │  saca al primero
 └─ (entrega la lista) ───────┘  ── la lista ──►  └─ y se la lleva

        ⟡ SALE del monitor ⟡                            ⟡ SALE del monitor ⟡
        (espera afuera)                            procesar(L) → comprobante

 ┌─ espera el comprobante ────┐  ◄─ el comp. ───  ┌─ entrega el comprobante
 └─ lo recibe y se va ────────┘                   └─ vuelve a buscar otro cliente
```

**Los dos salen del monitor en el mismo punto**, cuando el empleado se va a procesar. Ese es el único corte ⇒ **cada actor hace exactamente dos llamadas, y el monitor tiene cuatro procedures.**

| Quién | Llamada 1 | Llamada 2 |
|---|---|---|
| **Cliente** | `hacerFila(id, L)` | `recibirComprobante(id, out C)` |
| **Empleado** | `llamarCliente(out id, out L)` | `entregarComprobante(id, C)` |

**No se pueden juntar más:** si el cliente hiciera una sola llamada, se quedaría **adentro** del monitor mientras el empleado procesa, y nadie más podría entrar.
**No se pueden separar más:** "esperar turno" y "entregar la lista" son dos cosas seguidas que el cliente hace **sin salir** del monitor.

---

## 2. La decisión de diseño: ¿cuándo entrega la lista?

El enunciado dice *"cuando un cliente **es llamado**, entrega una lista"*. Hay dos lecturas:

| | **A — deja la lista al anotarse** | **B — literal: primero lo llaman, después entrega** |
|---|---|---|
| El `push` lleva | `{id, lista}` | sólo `{id}` |
| Procedures | **4** | 5 o 6 |
| Variables condición | 3 | 4 |

**Se elige A**, y se justifica así:

> El cliente **ya tiene la lista antes de llegar** —no la arma mientras espera— y **nadie la lee hasta que el empleado lo atiende**. Dejarla al anotarse o después de ser llamado produce **el mismo comportamiento observable**. Lo que el enunciado exige de verdad es el **orden de atención** y que el cliente **espere su comprobante**, y eso se cumple igual.

Conviene escribir esa línea en la entrega: muestra que fue una decisión y no un descuido.

---

## 3. Las variables condición: el criterio

**No es *"¿la condición es general?"*. Es *"¿tengo que ELEGIR a quién despierto?"*.**

| Espera | ¿A quién hay que despertar? | Forma |
|---|---|---|
| El **empleado** espera que haya clientes | A **cualquier** empleado libre | **una sola `cond`** |
| El **cliente** espera que lo llamen | A **ese** cliente (el primero de la fila) | **`cond[N]`** |
| El **cliente** espera su comprobante | A **ese** cliente | **`cond[N]`** |

Y hay una correspondencia que no es casual:

| | ¿`cond` arreglo o sola? | ¿Pasa su `id` como parámetro? |
|---|---|---|
| **Cliente** | Arreglo `[N]` | **Sí** |
| **Empleado** | Una sola | **No** |

> **Es la misma pregunta las dos veces: *¿el monitor tiene que distinguirlo?*** Un parámetro `id` sólo hace falta si hay que despertarlo puntualmente o indexar algo suyo. Ninguna decisión del monitor depende de **cuál** empleado llama — el enunciado hasta lo dice: *"espera a que **alguno** de los empleados le entregue el comprobante"*.

---

## 4. Los tres tipos de `wait`

Este ejercicio los tiene a los tres, así que sirve de catálogo.

| Situación | Forma | Por qué |
|---|---|---|
| *"esperá a que se cumpla una condición"* | **`while`** | Después del `wait` modificás estado, y alguien pudo colarse en el hueco |
| *"te paso el turno, ya te dejé todo hecho"* | **`if`** | Passing the Condition: nadie te lo pudo robar |
| *"nos esperamos mutuamente y tengo garantizado un `signal` dirigido a mí"* | **`wait` pelado** | Rendezvous: no hay nada que chequear |

Aplicados acá:

```
while (empty(colaDeEspera)) wait(esperaCliente);   // el EMPLEADO: hace pop después ⇒ while
wait(esperaEmpleado[id]);                           // el CLIENTE: pelado (rendezvous)
if (not listo[id]) wait(compListo[id]);             // el CLIENTE: if + bandera (ver §5)
```

**El `while` del empleado es el que hace que el ítem (b) funcione.** Con `if`: dos empleados dormidos, llega un cliente, uno despierta, pero **otro empleado que recién entra se lo lleva** en el hueco; el despertado hace `pop` sobre una cola vacía.

---

## 5. ⚠️ La trampa: la señal perdida

> **`signal` sobre una `cond` donde no hay nadie esperando NO HACE NADA: se pierde.** (A diferencia de `V`, que queda guardado en el valor del semáforo.)

Y el cliente **sale del monitor** entre sus dos llamadas:

| # | Cliente | Empleado | |
|---|---|---|---|
| 1 | vuelve de `hacerFila` | | está **afuera** del monitor |
| 2 | todavía no llamó a `recibirComprobante` | sale y **procesa la lista** | |
| 3 | | termina rápido: `entregarComprobante(id, C)` | |
| 4 | | `signal(compListo[id])` → **no hay nadie ahí** | 💥 señal perdida |
| 5 | recién ahora hace `wait(compListo[id])` | | **duerme para siempre** |

### Por qué en `hacerFila` esto NO pasa

```
push(colaDeEspera, id, L);
signal(esperaCliente);
wait(esperaEmpleado[id]);      ← el push y el wait ocurren SIN soltar el monitor
```

Como es **signal and continue**, el `signal` **no suelta el monitor**: el cliente sigue adentro hasta el `wait`. Ningún empleado puede entrar y señalizar antes de que esté en la cola de la `cond`.

> **La regla:** un `wait` pelado sólo es seguro si **te anotás y te dormís sin soltar el monitor en el medio**. Si salís y volvés a entrar, la señal te puede pasar por al lado.

### El arreglo: la cond no recuerda, el booleano sí

**No son alternativas: hacen trabajos distintos.**

| | Qué hace | Por qué no alcanza sola |
|---|---|---|
| `bool listo[N]` | **Recuerda** que ya está. Se puede **consultar** | Con sólo esto habría que hacer `while (not listo) skip` → **busy waiting**, prohibido |
| `cond compListo[N]` | **Bloquea y despierta** | No se puede consultar y **no tiene memoria** |

Es el mismo reparto que con semáforos en el pozo de recursos: **el semáforo da la espera, la estructura da la información.**

---

## 6. Solución (a)

```
// PRECONDICIONES: N > 0, E >= 1
//                 cada cliente trae su lista desde el inicio

monitor Corralon {
   cola colaDeEspera;              // pares {id, lista}, en orden de llegada
   cond esperaCliente;             // UNA sola: cualquier empleado libre sirve
   cond esperaEmpleado[N];         // arreglo: hay que despertar a ESE cliente

   comprobante compDe[N];          // el DATO
   bool listo[N] = ([N] false);    // la MEMORIA: ¿ya está?
   cond compListo[N];              // la ESPERA

   // ── CLIENTE ──
   procedure hacerFila (int id; lista L)
   { push(colaDeEspera, id, L);          // me anoto con mi lista
     signal(esperaCliente);              // le aviso al empleado que llegué
     wait(esperaEmpleado[id]);           // pelado: tengo un signal garantizado para mí
   }

   // ── EMPLEADO ──
   procedure llamarCliente (out int id; out lista L)
   { while (empty(colaDeEspera))         // while, no if: otro empleado puede ganarme
        wait(esperaCliente);
     pop(colaDeEspera, id, L);           // el primero de la fila, con su lista
     signal(esperaEmpleado[id]);         // lo llamo
   }

   // ── EMPLEADO ──
   procedure entregarComprobante (int id; comprobante C)
   { compDe[id] = C;                     // guardo el dato
     listo[id] = true;                   // lo REGISTRO ← evita perder la señal
     signal(compListo[id]);
   }

   // ── CLIENTE ──
   procedure recibirComprobante (int id; out comprobante C)
   { if (not listo[id])                  // ¿ya está? entonces no espero nada
        wait(compListo[id]);
     C = compDe[id];
   }
}


Process Cliente [id: 1..N]
{ lista L;  comprobante C;                     // trae su lista desde el inicio
  Corralon.hacerFila(id, L);                   // se anota y espera que lo llamen
  Corralon.recibirComprobante(id, C);          // espera su comprobante
}                                              // se retira. Sin while(true)


Process Empleado [id: 1..E]                    // en el (a), uno solo
{ int idCliente;                               // ← el id DEL CLIENTE, distinto del mío
  lista L;  comprobante C;

  while (true) {                               // el loop va en el PROCESO, no en el monitor
     Corralon.llamarCliente(idCliente, L);
     C = procesar(L);                          // AFUERA del monitor (incluye su delay)
     Corralon.entregarComprobante(idCliente, C);
  }
}
```

### Dos cosas sobre los procesos

**El `while(true)` va en el proceso, nunca adentro del monitor.** Un procedure de monitor **tiene que terminar**: mientras un proceso está adentro, nadie más puede entrar, así que un `while(true)` adentro deja el monitor tomado para siempre. Y además el loop contiene `procesar(L)`, que tiene que ir afuera. *El monitor es pasivo: la repetición es un comportamiento del empleado, no del mostrador.*

**El cliente no hace nada entre sus dos llamadas.** Toda su espera ocurre **adentro** del monitor, en los `wait`. Poner un `delay` ahí sería una pausa fija sin relación con cuándo esté listo el comprobante — `delay` modela *hacer algo que lleva tiempo*, no *esperar a otro proceso*.

---

## 7. Solución (b): es la misma

> *"E empleados (E > 1). Los empleados no deben terminar su ejecución."*

**No hay que cambiar nada.** Y eso no es suerte: quedó preparado al escribir el (a).

| Pieza | Por qué aguanta E empleados |
|---|---|
| `while (empty(colaDeEspera))` y no `if` | Dos empleados dormidos, llega un cliente, uno despierta pero **otro se lo lleva en el hueco**. El `while` lo manda a dormir de nuevo |
| `esperaCliente` es **una sola** `cond` | Cualquier empleado libre sirve: despertar a "alguno" es justo lo que se quiere |
| Todo indexado **por cliente** | Dos empleados atienden a clientes distintos ⇒ tocan casilleros distintos |
| El `pop` adentro del monitor | La exclusión mutua garantiza que dos empleados nunca saquen el mismo cliente |
| Los empleados no pasan su `id` | Ninguna decisión del monitor depende de cuál empleado llama |

### Justificación del (b)

> La solución del ítem (a) contempla E empleados sin modificaciones. La espera del empleado usa `while` y no `if`: al despertar vuelve a verificar que la cola no esté vacía, de modo que si otro empleado se adelantó y tomó al cliente, se vuelve a demorar en lugar de hacer `pop` sobre una cola vacía. `esperaCliente` es una única variable condición porque cualquier empleado libre puede atender al próximo cliente y no hace falta elegir cuál despertar. Toda la información está indexada por cliente —el semáforo privado de llamada, el comprobante y su marca de listo—, así que dos empleados que atienden clientes distintos nunca escriben sobre los mismos casilleros. La exclusión mutua implícita del monitor garantiza que dos empleados no extraigan el mismo cliente de la cola. Los empleados no necesitan comunicar su identificador porque ninguna decisión del monitor depende de cuál de ellos invoca los procedures.

---

## 8. Solución (c): con terminación

> *"los empleados deben terminar su ejecución cuando se hayan atendido todos los clientes"*

### La trampa del ítem

Cuando se atiende al último cliente puede haber **empleados dormidos** en `wait(esperaCliente)`, y nadie va a llegar nunca más. **Si no los despertás, duermen para siempre** y el programa no termina — aunque todos los clientes hayan sido atendidos correctamente. Es un error **silencioso**: los resultados son correctos y el programa simplemente no corta.

> **El mismo estado final —empleados dormidos para siempre— es CORRECTO en el (b) y es EL ERROR en el (c).** El código no cambia porque aparezca un bug: cambia porque **cambió el requisito**.

### Quién sabe que fue el último

**`id == N` NO es "el último".** Los clientes llegan en orden arbitrario: el cliente N puede llegar primero. Si avisara él, los empleados se irían con 49 clientes sin atender.

> Y hay un argumento más fuerte: **todo este ejercicio existe porque el orden de llegada no es el de los identificadores.** Si lo fuera, no haría falta ninguna cola — alcanzaría con una posta.

El único que puede saberlo es el **monitor**, y de una sola forma: **contando**. Es exactamente lo mismo que en una barrera, donde el último no es *"el que se llama N"* sino *"el N-ésimo en llegar"*.

### Dónde va el contador: al sacar, no al entregar

Si contaras las entregas de comprobante, con **N = 1, E = 2**:

| # | Empleado A | Empleado B | |
|---|---|---|---|
| 1 | saca al único cliente | | contador todavía en 0 |
| 2 | sale a procesar (largo) | cola vacía y contador < N → **espera** | |
| 3 | entrega el comprobante, contador = 1 | | |
| 4 | | **nadie le hace `signal(esperaCliente)`** | 💀 duerme para siempre |

Contando los `pop`, en cambio, **el que toma al último cliente está adentro del monitor en ese preciso momento** y puede despertar a los demás ahí mismo.

### Solución

Sólo cambian `llamarCliente` y el `Process Empleado`. Todo lo demás queda igual que en el (b).

```
monitor Corralon {
   ...todo lo del (b)...
   int clientesTomados = 0;              // ← NUEVO: cuántos ya salieron de la cola

   // ── CLIENTE ── sin cambios: no avisa nada, no lleva parámetros extra
   procedure hacerFila (int id; lista L)
   { push(colaDeEspera, id, L);
     signal(esperaCliente);
     wait(esperaEmpleado[id]);
   }

   // ── EMPLEADO ── el único que cambia
   procedure llamarCliente (out int id; out lista L; out bool hay)
   { while (empty(colaDeEspera) and (clientesTomados < N))   // AND, y while
        wait(esperaCliente);

     if (clientesTomados == N)            // ya no queda nada por tomar
        { hay = false;  return; }

     pop(colaDeEspera, id, L);
     clientesTomados++;
     if (clientesTomados == N)
        signalall(esperaCliente);         // ← despierto a TODOS los dormidos
                                          //   para que vean que se acabó y salgan
     signal(esperaEmpleado[id]);
     hay = true;
   }

   // entregarComprobante y recibirComprobante: SIN CAMBIOS
}

Process Empleado [id: 1..E]
{ int idCliente;  lista L;  comprobante C;  bool hay;

  hay = true;
  while (hay) {
     Corralon.llamarCliente(idCliente, L, hay);
     if (hay) {
        C = procesar(L);
        Corralon.entregarComprobante(idCliente, C);
     }
  }
}                                         // TERMINA: eso es lo que pide el (c)
```

### Los dos motivos para dejar de esperar — por eso va `and`

```
while (empty(colaDeEspera) and (clientesTomados < N))  wait(esperaCliente);
```

Se sigue esperando sólo si **se dan las dos cosas**: no hay nadie en la cola **y** todavía falta alguien por atender. Con `or`, un empleado se dormiría teniendo un cliente esperando adelante.

### Por qué `signalall` y no `signal`

El que toma al último cliente puede tener **varios** empleados dormidos detrás, y **a todos** se les acabó el trabajo al mismo tiempo. `signal` despertaría a uno solo, y ese uno —al salir— **ya no vuelve a entrar a despertar a nadie**.

> **`signalall` cuando el cambio de estado habilita a *todos* los que esperan.** Acá no es que uno pueda avanzar: es que a todos se les terminó.

### La traza que lo valida

**N = 1, E = 2**, los dos empleados dormidos:

| # | Qué pasa |
|---|---|
| 1 | A y B esperan: cola vacía y `clientesTomados = 0 < 1` |
| 2 | Llega el cliente, `signal(esperaCliente)` → despierta a A |
| 3 | A re-verifica: la cola **no** está vacía → sale del `while`, hace `pop`, `clientesTomados = 1` |
| 4 | `1 == N` → **`signalall(esperaCliente)`** → despierta a B |
| 5 | B re-verifica: cola vacía **y** `1 < 1` es falso → sale del `while` |
| 6 | B entra al `if (clientesTomados == N)` → `hay = false` → **termina** ✔ |
| 7 | A procesa, entrega, vuelve al loop, ve que no queda → **termina** ✔ |

### La regla general (sirve para cualquier ejercicio con terminación)

Cada vez que un enunciado dice *"debe retirarse / deben terminar / todos los procesos deben terminar"*, preguntate:

> **¿Cada proceso sabe de antemano cuántas vueltas le tocan?**

| | Mecanismo |
|---|---|
| **Sí, el reparto es fijo** | Un **`for` acotado** y listo |
| **No, el reparto es dinámico** | **Contador compartido + despertar a los dormidos + bandera de salida** |

| Ejercicio | ¿Sabe cuántas vueltas? | Solución |
|---|---|---|
| Vacunatorio (P2-11): *"cuando atendió a las 50 se retira"* | **Sí**: 50/5 = 10 tandas | `for tanda = 1 to 10` |
| Cerealera (P2-10a): *"se retira cuando todos descargaron"* | **Sí**: T+M camiones | dos `for` de T+M |
| Terminal de micros (P2-12): el recepcionista | **Sí**: 150 pasajeros | `for i = 1 to 150` |
| **Corralón (c)**: E empleados | **No** — depende de quién sea más rápido | **contador + `signalall` + `out hay`** |

**La diferencia la hace el reparto dinámico:** si un empleado atiende 30 clientes y otro 20, ninguno puede escribir un `for` con un número. **Sólo el recurso compartido sabe cuándo se acabó.**

Y las tres piezas son siempre las mismas:

1. **Alguien tiene que contar** → un contador en el monitor.
2. **Hay que despertar a los que ya están dormidos** → `signalall`.
3. **Hay que avisarle al que despierta que se vaya** → un `out bool`.

**Olvidarse de la 2 es el error más común, y es silencioso:** el programa da resultados correctos y simplemente no termina.

### Justificación del (c)

> El requisito de terminación obliga a que el monitor sepa cuándo ya no queda trabajo, información que ningún cliente ni empleado posee individualmente: los clientes llegan en orden arbitrario —`id == N` no identifica al último— y el reparto entre los E empleados es dinámico, de modo que ninguno puede acotar sus iteraciones con un `for`. Se agrega entonces el contador `clientesTomados`, incrementado **al extraer un cliente de la cola** y no al entregar el comprobante: ese es el instante en que se sabe que no quedan clientes por tomar, y además el empleado que lo detecta está dentro del monitor y puede actuar. La condición de espera pasa a ser compuesta —se espera mientras la cola esté vacía **y** falten clientes por atender— porque ahora hay dos motivos distintos para dejar de esperar. Cuando el contador alcanza N se emite `signalall` sobre `esperaCliente`: no alcanza con `signal`, porque a todos los empleados demorados se les acabó el trabajo simultáneamente y el que despertara con `signal` no volvería a entrar al monitor a despertar a los demás. Finalmente, `llamarCliente` devuelve una bandera `hay` para que el empleado distinga entre "te asigné un cliente" y "ya no queda nada": sin ella intentaría procesar una lista inexistente. Los clientes no requieren ningún cambio respecto del ítem (b).

## 9. Los errores que se penalizan

**(1) `while(true)` adentro de un procedure del monitor.** El monitor queda tomado para siempre: deadlock total.

**(2) `procesar(L)` adentro del monitor.** Los E empleados atienden **de a uno** — el monitor serializa todo lo que pasa adentro. Contratar E empleados no sirve de nada.

**(3) `if` en vez de `while` en la espera del empleado.** Funciona en el (a) con un empleado y **se rompe en el (b)**.

**(4) `wait(compListo[id])` pelado, sin la bandera `listo[]`.** Si el empleado termina antes de que el cliente llame a `recibirComprobante`, **la señal se pierde** y el cliente duerme para siempre.

**(5) Usar el `id` del empleado donde va el del cliente.** `entregarComprobante(id, C)` dentro de `Process Empleado[id: 1..E]` le manda el comprobante al cliente equivocado. El id del cliente llega por un `out` y hay que guardarlo en otra variable.

**(6) Un candado adentro del monitor.** La exclusión mutua es implícita. `compDe[]` y `listo[]` son variables permanentes: ya están protegidas.

**(7) Un `delay` en el cliente entre sus dos llamadas.** No espera nada real: la espera la hacen los `wait`.

**(8) `esperaEmpleado` como una sola `cond`.** Hay que despertar al cliente que se sacó de la cola, no a uno cualquiera.

### Específicos del (c)

**(9) Hacer que el cliente avise que es el último con `id == N`.** Los clientes llegan en orden arbitrario: el cliente N puede llegar primero y los empleados se irían con 49 sin atender.

**(10) Contar en `entregarComprobante` en vez de en `llamarCliente`.** Deja empleados dormidos sin nadie que los despierte (traza del §8).

**(11) `or` en vez de `and` en la condición de espera.** El empleado se duerme teniendo un cliente esperando adelante.

**(12) `signal` en vez de `signalall`.** Despierta a uno y los demás quedan colgados — y ese uno, al salir, ya no vuelve a entrar a despertar a nadie. **Es el error más común del ítem, y es silencioso: el programa funciona bien y simplemente no termina.**

**(13) No devolverle al empleado una bandera de salida.** Intenta procesar una lista que no existe.

---

## 10. Chequeos de sanidad

- **Cada cliente hace exactamente dos llamadas** y cada empleado dos por vuelta.
- **Cada `esperaEmpleado[i]` recibe un solo `signal`**, y cada `compListo[i]` a lo sumo uno.
- **`listo[i]` pasa de `false` a `true` una sola vez** y nunca vuelve.
- **La cola termina vacía:** N `push` y N `pop`.
- **Con E = 1** el ítem (b) colapsa al (a): el `while` del empleado nunca da más de una vuelta porque nadie le puede ganar.
- **Con E > N**: algunos empleados nunca atienden a nadie y quedan dormidos en `wait(esperaCliente)` — correcto en el (b), que dice que **no terminan**. En el (c) eso es justamente el problema a resolver.
- **Sin deadlock:** los `wait` sueltan el monitor, así que siempre puede entrar otro proceso.
