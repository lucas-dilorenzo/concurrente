# Clase 3 — Semáforos

> Cubre las **dos partes** de la presentación usada en clase (= Teoría 3 sin *filósofos* ni *varios recursos compartidos*).
> **Qué pregunta responde:** Teoría 2 terminó con *"necesidad de herramientas para diseñar protocolos de sincronización"*. El semáforo es esa herramienta.

---

## 1. Qué es un semáforo

Descripto por **Dijkstra, 1968**.

> **Semáforo:** instancia de un tipo de datos abstracto con **sólo dos operaciones atómicas**, `P` y `V`. Internamente su valor es un **entero no negativo**.
>
> - **`V`** → señala la ocurrencia de un evento (**incrementa**)
> - **`P`** → demora un proceso hasta que ocurra un evento (**decrementa**)

```
Semáforo general (counting):     P(s): < await (s > 0) s = s-1 >
                                 V(s): < s = s+1 >

Semáforo binario:                P(b): < await (b > 0) b = b-1 >
                                 V(b): < await (b < 1) b = b+1 >
```

**El `V` del binario tiene guarda** — es lo que lo mantiene en {0,1}. Si ya está en 1, el `V` se demora. Se pasa por alto fácil y es diferencia de examen.

**Declaración — hay que inicializar sí o sí:**

```
sem s;                    // NO
sem mutex = 1;            // sí
sem fork[5] = ([5] 1);    // arreglo de semáforos
```

### La advertencia que hay que tener presente todo el tiempo

> Si la demora de las operaciones `P` se implementa **sobre una cola**, las operaciones son fair.
> **EN LA MATERIA NO SE PUEDE SUPONER ESTE TIPO DE IMPLEMENTACIÓN.** *(está en mayúsculas en la filmina)*

O sea: **el semáforo no garantiza orden**. Si hay tres procesos esperando en un `P` y llega un `V`, no sabés cuál despierta. **Si el problema pide orden, lo construís vos** — y esa es la razón de ser de *Passing the Baton*.

---

## 2. De dónde sale: la derivación desde Teoría 2

El semáforo no es un invento nuevo, es un **renombre** de algo que ya escribiste:

```
bool lock = false;                  bool free = true;              int free = 1;
await (not lock) lock = true;   →   await (free) free = false;  →  < await (free > 0) free = free-1 >
lock = false;                       free = true;                   < free = free + 1 >
```

Cambio de variable, después booleano como entero. Y esas dos últimas líneas **son exactamente `P` y `V`**:

```
sem free = 1;

process SC[i = 1 to n]
{ while (true)
   { P(free);  sección crítica;  V(free);  sección no crítica; }
}
```

Comparalo con Bakery o Tie-Breaker: *"Es más simple que las soluciones busy waiting."*

**¿Y si inicializo `free = 0`?** Nadie entra nunca: el primer `P` espera un `V` que sólo puede venir de alguien que ya esté adentro. **Deadlock total.**

---

## 3. La idea que ordena todo el resto

**Hay sólo tres formas de usar un semáforo, y el valor inicial te dice cuál es.**

| Init | Rol | Cómo se usa | Ejemplo |
|---|---|---|---|
| **`= 1`** | **Mutex** | `P` … `V` **por el mismo proceso**, rodeando la SC | `sem mutex = 1` |
| **`= 0`** | **Señalización** | Un proceso hace `V`, **otro** hace `P` | `sem llega1 = 0` |
| **`= n`** | **Contador de recursos** | Cuenta unidades libres | `sem vacio = n` |

> **Regla de lectura:** **mutex** → `P` y `V` los hace el **mismo** proceso. **Señalización** → los hacen procesos **distintos**.
> Si ves un `P` y un `V` del mismo semáforo en procesos diferentes, no es un candado: es un mensaje.

Todos los problemas de la clase son uno de esos tres, o una combinación.

---

## 4. Patrón — Señalización de eventos

**Idea:** un semáforo por cada flag de sincronización. Prender el flag es `V`; esperarlo y limpiarlo es `P`.

> Semáforo de señalización → **generalmente inicializado en 0**.

**Barrera para dos procesos:**

```
sem llega1 = 0, llega2 = 0;

process Worker1                  process Worker2
{ ...                            { ...
  V(llega1);  P(llega2);           V(llega2);  P(llega1);
  ...                              ...
}                                }
```

Cada uno **avisa que llegó** y después **espera al otro**. Es el principio de flags de Teoría 2, pero el semáforo ya trae el "esperar y limpiar" en una operación atómica. Con esta barrera de a dos se arma una **butterfly** para n.

**¿Qué pasa si primero hacen `P` y después `V`?** → **Deadlock.** Los dos esperan una señal que el otro todavía no mandó, y como están bloqueados en el `P`, nunca llegan a su `V`.

> **En criollo:** primero avisás que llegaste, después esperás. Al revés, se cuelgan los dos.

---

## 5. Patrón — Semáforo Binario Dividido (SBS)

> **Split Binary Semaphore:** los semáforos binarios `b1, ..., bn` forman un SBS si es invariante global
> **`0 ≤ b1 + ... + bn ≤ 1`**

Se lee como **un único semáforo binario partido en n pedazos**: entre todos, a lo sumo hay un permiso circulando.

**Por qué importa:** si cada camino de ejecución **empieza con un `P` sobre uno** y **termina con un `V` sobre otro**, las sentencias del medio se ejecutan **con exclusión mutua** — gratis, sin declarar ningún mutex.

**Buffer unitario, M productores y N consumidores.** Depositar y retirar deben alternarse:

```
typeT buf;   sem vacio = 1, lleno = 0;

process Productor[i = 1..M]              process Consumidor[j = 1..N]
{ while(true)                            { while(true)
   { producir mensaje datos                 { P(lleno); resultado = buf; V(vacio);
     P(vacio); buf = datos; V(lleno);         consumir mensaje resultado
   } }                                    } }
```

`vacio + lleno = 1` **siempre** → son un SBS. Y eso compra algo grande: **funciona con M y N sin ningún mutex adicional**, porque el invariante ya garantiza que a lo sumo uno esté entre el `P` y el `V`.

---

## 6. Patrón — Contadores de recursos: buffer limitado

```
typeT buf[n];   int ocupado = 0, libre = 0;
sem vacio = n, lleno = 0;

process Productor                                    process Consumidor
{ while(true)                                        { while(true)
   { producir mensaje datos                             { P(lleno);
     P(vacio);                                            resultado = buf[ocupado];
     buf[libre] = datos;                                  ocupado = (ocupado+1) mod n;
     libre = (libre+1) mod n;                             V(vacio);
     V(lleno);                                            consumir mensaje resultado
   } }                                                } }
```

- **`vacio`** cuenta lugares libres, **`lleno`** los ocupados. Invariante: `vacio + lleno = n`.
- Depositar y retirar **se pudieron asumir atómicas** porque hay **un solo** productor y **un solo** consumidor.

### Es el Ejercicio 3 de la Práctica 1

| Con `await` (Práctica 1, Ej. 3) | Con semáforos (Clase 3) |
|---|---|
| `<await (cant < N); ...; cant++>` | `P(vacio); ...; V(lleno)` |
| `<await (cant > 0); ...; cant-->` | `P(lleno); ...; V(vacio)` |
| Un contador `cant` | **Dos** semáforos, `vacio + lleno = n` |

El par de semáforos **es** el contador `cant`, con el bloqueo incorporado. Y ya no hay riesgo de que el contador mienta: el `V(lleno)` va **después** de escribir el buffer.

### Con varios productores/consumidores

> *"Si no se protege cada slot, podría retirarse dos veces el mismo dato o perderse datos al sobrescribirlo."*

Es **la misma traza** del ejercicio 3(b): dos productores leen el mismo `libre` y uno pisa al otro. Los índices dejan de tener dueño único.

```
sem vacio = n, lleno = 0;   sem mutexD = 1, mutexR = 1;

process Productor[i = 1..M]
{ producir mensaje datos
  P(vacio);
  P(mutexD);  buf[libre] = datos;  libre = (libre+1) mod n;  V(mutexD);
  V(lleno);
}

process Consumidor[i = 1..N]
{ P(lleno);
  P(mutexR);  resultado = buf[ocupado];  ocupado = (ocupado+1) mod n;  V(mutexR);
  V(vacio);
  consumir mensaje resultado
}
```

**Lo que hay que mirar es el ORDEN de los dos `P`:** primero `P(vacio)`, después `P(mutexD)`. Nunca al revés — si tomás el mutex antes de saber si hay lugar, te bloqueás **con el candado en la mano**.

Acá, con mutex separados, no llega a deadlock (el consumidor usa `mutexR` y puede liberar `vacio`), pero bloquea de gusto a los otros productores. **Con un único mutex compartido sería deadlock directo.**

> **Regla general:** nunca te bloquees en un semáforo de condición mientras tenés un mutex tomado. **Primero el permiso, después el candado.**

Hacen falta **dos** mutex y no uno porque productores y consumidores tocan índices distintos (`libre` vs `ocupado`): serializarlos entre sí sería perder concurrencia sin motivo.

---

## 7. Alocación de recursos y scheduling

> **Problema:** decidir **cuándo** se le puede dar a un proceso acceso a un recurso.
> **Recurso:** cualquier objeto, elemento, componente, dato o SC por la que un proceso puede ser demorado esperando adquirirlo.

```
request(parámetros):  await (request puede ser satisfecho) tomar unidades;
release(parámetros):  retornar unidades;
```

**Una unidad, sin orden** → es el problema de la SC, se resuelve con un mutex:

```
sem mutex = 1;
Request: P(mutex);   // usa el recurso   Release: V(mutex);
```

**Por orden de llegada** → como el semáforo no garantiza orden, hay que construirlo con una cola:

```
sem mutex = 1;   cola espera;   bool libre = true;

request(id):                                      release():
   P(mutex);                                         P(mutex);
   push(espera, id);                                 libre = true;
   while ((libre == false) or (top(espera) != id))   pop(espera, ...);
      { V(mutex);  P(mutex); }   ← BUSY WAITING      V(mutex);
   libre = false;
   V(mutex);
```

Ese `V(mutex); P(mutex);` adentro del `while` **es busy waiting**: el proceso suelta el candado y lo vuelve a tomar hasta que le toque. Vinimos a los semáforos para escaparle y reapareció. **Esa es la motivación de Passing the Baton.**

---

## 8. Passing the Baton

> **Técnica general para implementar sentencias `await`.**
> Un proceso dentro de la SC **mantiene el baton** (testimonio, token): el permiso para ejecutar. Al salir, **pasa el baton** a otro proceso. Si nadie espera, **lo libera** para el próximo que llegue.

> **Semáforo privado:** `s` es privado si **exactamente un proceso** ejecuta `P` sobre `s`. Sirven para señalar procesos individuales.

### Esquema general

```
e          semáforo binario inicialmente 1 — controla la entrada a las sentencias atómicas
bj, dj     por cada guarda distinta Bj: un semáforo bj y un contador dj, ambos en 0
           bj demora a los que esperan Bj;  dj cuenta cuántos hay demorados en bj
```

`e` y los `bj` forman un **SBS**: a lo sumo uno vale 1 a la vez, y cada camino empieza con un `P` y termina con un único `V`.

```
F1:  < Si >               →    P(e);  Si;  SIGNAL;

F2:  < await (Bj) Sj >    →    P(e);
                               if (not Bj) { dj = dj + 1;  V(e);  P(bj); }
                               Sj;
                               SIGNAL;

SIGNAL:  if      (B1 and d1 > 0)  { d1 = d1 - 1;  V(b1) }
         □       ...
         □       (Bn and dn > 0)  { dn = dn - 1;  V(bn) }
         else    V(e);
         fi
```

### Lo que hay que entender de `SIGNAL`

Al salir hay **dos opciones excluyentes**:

- Hay alguien esperando cuya condición **ahora** se cumple → `V(bj)`. **Y NO hacés `V(e)`.**
- No hay nadie → `V(e)`: liberás el baton para el próximo que llegue.

> **La clave:** el proceso que despertás **no vuelve a hacer `P(e)`**. Recibe el baton **ya tomado** y sigue directo con su `Sj`.
> Por eso "pasar el testimonio": no lo devolvés a la pista, se lo das **en la mano** al que sigue. Entre el `V(bj)` y el `SIGNAL` del despertado **nadie más pudo entrar** — la exclusión mutua nunca se soltó.

### Alocación de recursos con Passing the Baton

```
bool libre = true;   cola espera;
sem baton = 1;       sem b[n] = ([n] 0);      // un semáforo privado por proceso

Process Cliente[id: 1..n]
{ int sig;
  // trabaja
  P(baton);
  if (!libre) { push(espera, id);  V(baton);  P(b[id]); }   ← duerme en SU semáforo
  libre = false;
  V(baton);

  // USA EL RECURSO

  P(baton);
  libre = true;
  if (not empty(espera)) { pop(espera, sig);  V(b[sig]); }  ← le paso el baton a sig
  else V(baton);                                            ← no hay nadie: lo suelto
}
```

**Desapareció el `while`.** Nadie gira: el que se duerme se duerme de verdad, y el que sale elige exactamente a quién despertar.

### Dos variantes que salen casi gratis

**Shortest-Job-Next** — *sólo cambia el orden con que se inserta en la cola*:

```
insertar(espera, id, tiempo);    en vez de    push(espera, id);
sacar(espera, sig);              en vez de    pop(espera, sig);
```

Todo el resto queda **idéntico**. Ese es el gran valor de la técnica: **la política de scheduling vive enteramente en la disciplina de la cola**, separada del mecanismo de sincronización.

**Recursos de múltiples instancias** — el booleano pasa a contador:

```
int libres = x;   cola espera;   sem baton = 1, b[n] = ([n] 0);

P(baton);
if (libres == 0) { push(espera, id); V(baton); P(b[id]); }
libres--;
V(baton);
// USA EL RECURSO
P(baton);
libres++;
if (not empty(espera)) { pop(espera, sig); V(b[sig]); }
else V(baton);
```

*(La filmina declara `libres` y después escribe `libre` en el cuerpo — es la misma variable, es un typo.)*

---

## 9. Lectores y Escritores

> **Problema:** dos clases de procesos comparten una Base de Datos. El acceso de los **escritores** debe ser **exclusivo**. Los **lectores** pueden ejecutar **concurrentemente entre ellos** si no hay escritores actualizando.

- Procesos **asimétricos**, con distinta prioridad según el scheduler.
- Es **exclusión mutua selectiva**: procesos que compiten por conjuntos **superpuestos** de variables compartidas.
- Dos enfoques: como **exclusión mutua** o como **sincronización por condición**.

### Enfoque 1 — Como exclusión mutua

Intento ingenuo:

```
sem rw = 1;
Lector:    P(rw);  lee la BD;      V(rw);
Escritor:  P(rw);  escribe la BD;  V(rw);
```

Correcto, pero **no hay concurrencia entre lectores** — justo lo que el problema permitía.

**La observación que lo arregla:**

> Los lectores **como grupo** necesitan bloquear a los escritores, pero **sólo el primero** necesita hacer `P(rw)`, y **sólo el último** debe hacer `V(rw)`.

```
int nr = 0;   sem rw = 1;   sem mutexR = 1;    # mutexR protege el acceso a nr

process Lector[i = 1..M]                    process Escritor[j = 1..N]
{ P(mutexR);  nr = nr + 1;                  { P(rw);
  if (nr == 1) P(rw);  V(mutexR);             escribe la BD;
  lee la BD;                                  V(rw);
  P(mutexR);  nr = nr - 1;                  }
  if (nr == 0) V(rw);  V(mutexR);
}
```

**Observación:** el primer lector hace `P(rw)` **teniendo `mutexR` tomado** — justo lo que la regla del §6 desaconseja. Acá no es deadlock (el escritor que tiene `rw` no necesita `mutexR`), pero serializa a los lectores detrás del primero. La regla sigue valiendo; sólo hay que evaluar **quién** puede liberarte.

**El problema de fondo:** da **preferencia a los lectores** → **no es fair**. Un flujo continuo de lectores deja a los escritores esperando indefinidamente. Es inanición: la propiedad 4 de Teoría 2.

### Enfoque 2 — Como sincronización por condición

```
int nr = 0, nw = 0;

process Lector[i = 1..M]                     process Escritor[j = 1..N]
{ < await (nw == 0) nr = nr + 1; >           { < await (nr == 0 and nw == 0) nw = nw + 1; >
  lee la BD;                                   escribe la BD;
  < nr = nr - 1; >                             < nw = nw - 1; >
}                                            }
```

Como **especificación** es impecable. El problema es implementarla.

#### Por qué acá los semáforos no alcanzan

> Las guardas **se superponen**: el escritor necesita que `nw` **y** `nr` sean 0; el lector sólo que `nw` sea 0.
> **Ningún semáforo podría discriminar entre estas dos condiciones.** → Passing the Baton.

Un semáforo es un solo entero: no puede representar dos condiciones distintas sobre el mismo estado.

#### La adaptación: baton por *tipo* de proceso

> En lugar de mantener el orden **por proceso**, se hace **por tipo de proceso**. Cada tipo espera una condición distinta → hacen falta **sólo dos** semáforos (`r` y `w`) y **dos** contadores (`dr`, `dw`).
> La SC que protege el baton (`e`) es **donde se usan las variables compartidas**: `nr`, `nw`, `dr`, `dw`.

Compará: en la alocación de recursos hacía falta `b[n]` —un privado **por proceso**—, porque cada uno tenía su lugar en la cola. Acá alcanzan **dos**, porque los procesos se agrupan en dos clases con la misma condición.

### La solución completa

```
int nr = 0, nw = 0, dr = 0, dw = 0;        sem e = 1, r = 0, w = 0;

process Lector[i = 1..M]                   process Escritor[j = 1..N]
{ while(true)                              { while(true)
   { P(e);                                    { P(e);
     if (nw > 0) { dr = dr+1;                   if (nr > 0 or nw > 0)
                   V(e); P(r); }                    { dw = dw+1; V(e); P(w); }
     nr = nr + 1;                                nw = nw + 1;
     if (dr > 0) { dr = dr-1; V(r); }            V(e);
     else V(e);
     lee la BD;                                  escribe la BD;
     P(e);                                       P(e);
     nr = nr - 1;                                nw = nw - 1;
     if (nr == 0 and dw > 0)                     if (dr > 0) { dr = dr-1; V(r); }
        { dw = dw-1; V(w); }                     elseif (dw > 0) { dw = dw-1; V(w); }
     else V(e);                                  else V(e);
   } }                                       } }
```

### Cómo leerla — los cuatro SIGNAL

| Dónde | Qué decide | Por qué |
|---|---|---|
| **Entrada del lector** | `if (dr > 0) V(r)` | **Cascada de lectores.** Apenas entro despierto a otro lector que esperaba, y ese hace lo mismo. Así entran todos en fila y leen concurrentemente |
| **Entrada del escritor** | `V(e)` a secas | No hay cascada: sólo uno puede escribir, no tiene a quién despertar |
| **Salida del lector** | `if (nr == 0 and dw > 0) V(w)` | Sólo el **último** lector en salir despierta a un escritor |
| **Salida del escritor** | `if (dr>0) V(r)` **elseif** `(dw>0) V(w)` | Chequea **lectores primero** → **da preferencia a los lectores** |

La **cascada del lector** es lo más lindo de la solución: es cómo se le da el baton a **muchos** procesos, uno pasándoselo al siguiente, sin soltarlo nunca al aire.

### *"Da preferencia a los lectores → ¿cómo puede modificarse?"*

Dos cambios, y hay que hacer **los dos**:

1. En la **salida del escritor**, invertir el orden: chequear `dw` antes que `dr`.
2. En la **entrada del lector**, endurecer la guarda a `if (nw > 0 or dw > 0)` — para que un lector nuevo no se cuele adelante de escritores que ya estaban esperando.

Sin el punto 2, un goteo de lectores nuevos sigue postergando a los escritores.

---

## 10. Lo que quedó afuera de las dos clases

Comparando Clase 3 (partes 1 y 2) contra **Teoría 3 completa**:

| Tema | Qué es |
|---|---|
| **Varios procesos compitiendo por varios recursos compartidos** | El caso general, previo a los filósofos |
| **Problema de los filósofos** | Exclusión mutua selectiva con `sem fork[5] = ([5] 1)`. *Ese arreglo de semáforos que apareció suelto en el slide de declaraciones era el anticipo de este problema* |

---

## 11. Checklist de trampas

- **`P` y `V` del mismo proceso = mutex. De procesos distintos = señalización.** Si te confundís el rol, te confundís el valor inicial.
- **Inicializar siempre.** `sem s;` no existe. Y `sem mutex = 0` es deadlock garantizado.
- **Avisá antes de esperar.** En barreras, `V` y después `P`. Al revés se cuelgan los dos.
- **Primero el permiso, después el candado.** Nunca `P(mutex)` antes que `P(condición)`.
- **No supongas que el semáforo es FIFO.** Si el problema pide orden, lo construís vos.
- **En `SIGNAL`, el que despierta NO vuelve a hacer `P(e)`.** Recibe el baton tomado.
- **El `V` del semáforo binario tiene guarda.** El del general no.
