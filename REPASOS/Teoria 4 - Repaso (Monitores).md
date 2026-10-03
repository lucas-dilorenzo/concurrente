https://github.com/playsistemico/modo-sdk-pagos-ios/pull/1860# Teoría 4 / Clase 4 — Monitores

> Repaso por conceptos de **Teoría 4** + **Clase 4**. Los dos PDFs son casi iguales; las diferencias están marcadas al final.

---

## 0. El problema que viene a resolver

Con **semáforos**, un recurso compartido se protege así:

```
sem mutex = 1;
Process A { ... P(mutex); toca el recurso; V(mutex); ... }
Process B { ... P(mutex); toca el recurso; V(mutex); ... }
Process C { ... P(mutex); toca el recurso; V(mutex); ... }
```

Los problemas, según la teoría:

| Problema del semáforo | Qué significa |
|---|---|
| Variables compartidas **globales** a todos los procesos | Cualquiera puede tocar el recurso directamente |
| Los `P`/`V` están **dispersos en el código** | La lógica de un recurso está repartida en N procesos distintos |
| Al **agregar un proceso** hay que re-verificar todo | Si el proceso nuevo se olvida un `V`, se rompe todo el sistema |
| **Exclusión mutua y sincronización se programan igual** | Aunque son conceptos distintos, los dos se escriben con `P`/`V`, y eso confunde |

> **La idea del monitor:** en vez de que cada proceso se acuerde de pedir permiso, **encerrar el recurso** y que la única forma de tocarlo sea llamando a sus procedures. El que se tiene que acordar de sincronizar es el recurso, no sus usuarios.

Es **abstracción de datos aplicada a concurrencia**: como un TAD, pero compartido por procesos que ejecutan concurrentemente.

---

## 1. Anatomía

```
monitor NombreMonitor {
    declaraciones de variables permanentes;     // el ESTADO del recurso
    código de inicialización;

    procedure op1 (parámetros formales)
      { cuerpo de op1 }
    ...
    procedure opn (parámetros formales)
      { cuerpo de opn }
}
```

- **Interfaz** = los nombres de los procedures. Es lo único visible desde afuera.
- **Cuerpo** = las variables permanentes (el estado) + la implementación.
- Se invoca así: `NombreMonitor.operacion(argumentos)`
- Un procedure **sólo** puede acceder a: variables permanentes, sus variables locales, y los parámetros que le pasaron. **Nada más.**

Dos consecuencias que la teoría subraya:

- **El que llama puede ignorar cómo está implementado.** Y el que programa el monitor puede ignorar **quién** y **en qué orden** lo llama — de hecho *"el programador de un monitor no puede conocer a priori el orden de llamado"*.
- **Procesos activos, monitores pasivos.** El monitor no hace nada por su cuenta: sólo reacciona cuando alguien lo llama. Dos procesos interactúan **invocando procedures del mismo monitor**.

### Ejemplo mínimo

5 empleados que producen, y un coordinador que cada tanto quiere ver el total:

```
monitor TOTAL {
    int cant = 0;

    procedure incrementar ()
      { cant = cant + 1; }

    procedure verificar (R: out int)
      { R = cant; }
}

process empleado[id: 0..4] { while (true) { ... TOTAL.incrementar(); ... } }
process coordinador      { while (true) { ... TOTAL.verificar(c);  ... } }
```

**No hay ni un `P` ni un `V` en ningún lado, y `cant++` es seguro.** Eso es lo que compra el monitor.

---

## 2. Las dos mitades (la distinción central de la unidad)

| | Cómo se consigue | ¿Hay que escribirla? |
|---|---|---|
| **Exclusión mutua** | **IMPLÍCITA**: dos procedures del mismo monitor nunca ejecutan concurrentemente | **No.** Viene gratis |
| **Sincronización por condición** | **EXPLÍCITA**: con variables condición (`cond`) | **Sí.** Es todo tu trabajo |

> **En criollo:** *el monitor te regala el candado y te cobra la espera.*

Con semáforos las dos cosas se escribían igual (`P`/`V`) y por eso se confundían. Acá quedan separadas: **la primera desaparece del código, la segunda es lo único que escribís.**

Todo lo que sigue es sobre la segunda mitad.

---

## 3. Variables condición: `wait`, `signal`, `signal_all`

```
cond cv;
```

Una variable condición **no guarda un valor**. Lo que tiene asociado es **una cola de procesos demorados**, que no es visible directamente al programador.

| Operación | Qué hace |
|---|---|
| `wait(cv)` | El proceso **se demora al final de la cola** de `cv` **y deja el acceso exclusivo al monitor** |
| `signal(cv)` | Despierta al proceso que está **al frente de la cola** (si hay alguno) y lo saca de ella. El despertado **recién podrá ejecutar cuando readquiera el acceso exclusivo al monitor** |
| `signal_all(cv)` | Despierta a **todos** los demorados en `cv`; la cola queda vacía |

Las dos partes de `wait` que hay que tener grabadas:

1. **Se duerme**, siempre.
2. **Suelta el monitor.** Si no lo soltara, nadie más podría entrar y nadie podría despertarlo nunca. Es lo mismo que `V(e)` antes de `P(priv[id])` en Passing the Baton con semáforos: **dormirse sin soltar el candado es deadlock**. Acá el soltar viene incluido en el `wait`.

### `wait`/`signal` NO son `P`/`V` — la tabla que hay que saber

| `WAIT` | `P` |
|---|---|
| El proceso **siempre** se duerme | El proceso **sólo** se duerme si el semáforo está en 0 |

| `SIGNAL` | `V` |
|---|---|
| Si hay procesos dormidos, despierta al primero. **Si no hay ninguno, no tiene ningún efecto posterior** | Incrementa el semáforo, para que un proceso dormido **o uno que haga `P` después** continúe |
| Despierta al **primero de la cola** (la cola de `cv` es FIFO) | **No sigue ningún orden** al despertar |

> **Las dos diferencias que más cuestan:**
>
> 1. **La variable condición no tiene memoria.** Un `signal` sin nadie esperando **se pierde para siempre**. Un `V` sin nadie esperando queda guardado en el valor del semáforo y sirve para el próximo `P`. Por eso un monitor **siempre** necesita variables de estado propias (`cant`, `nr`, `libre`, …): el estado lo llevás vos, la `cond` sólo sirve para dormir y despertar.
> 2. **`wait` siempre duerme.** No existe "hago wait y si la condición ya se cumple sigo". Por eso el `wait` **va precedido de un `if` o un `while` que consulte tus variables de estado.**

---

## 4. Signal and Continue vs. Signal and Wait

Cuando un proceso hace `signal(cv)`, hay dos procesos candidatos a seguir ejecutando dentro del monitor: **el que señalizó** y **el que se despertó**. Sólo puede haber uno. ¿Cuál sigue?

| Disciplina | Quién sigue en el monitor | Qué le pasa al otro |
|---|---|---|
| **Signal and Continue** ← **la que usa la materia** | **El que hizo el `signal`** continúa usando el monitor | El despertado **pasa a competir** para reentrar al monitor, y cuando entra sigue en la instrucción que le sigue al `wait` |
| **Signal and Wait** | **El despertado** pasa a ejecutar a partir de la instrucción que le sigue al `wait` | El que señalizó pasa a competir para reentrar |

**En esta materia es siempre Signal and Continue.** Y de ahí sale la regla más importante de toda la unidad:

---

## 5. ⚠️ La regla del `while` (el error clásico)

Es **la** regla de la unidad. Vale desarmarla despacio.

### Hay DOS colas, no una

```
                    ┌─────────────────────────────────┐
  Proceso nuevo ──► │  COLA DE ENTRADA AL MONITOR     │ ──► ejecuta adentro
  (nunca durmió)    │  (los que quieren entrar)       │
                    └─────────────────────────────────┘
                                   ▲
                                   │ signal(cv) te manda ACÁ
                                   │
                    ┌──────────────┴──────────────────┐
                    │  COLA DE LA VARIABLE CONDICIÓN  │
                    │  (los que hicieron wait(cv))    │
                    └─────────────────────────────────┘
```

Lo que **sí** está garantizado: la cola de la variable condición **es FIFO**, y `signal(cv)` despierta al **primero**. Entre los dormidos, no hay sorteo.

Lo que **no** está garantizado:

> Que te despierten **no te mete en el monitor**. Te manda a la **cola de entrada**, a competir con procesos que **nunca durmieron**.
>
> **El peligro no es que otro dormido te gane: es que te gane alguien que recién llegó.** Eso se llama *barging*.

Con **Signal and Wait** el hueco no existe (el despertado entra de una), y por eso allá el `while` no haría falta. Con **Signal and Continue** —la disciplina de la materia— sí.

### La analogía

Un consultorio con **un solo médico** (el monitor: uno adentro por vez).

- **Sala de espera general** = la cola de entrada al monitor. Ahí está el que acaba de llegar de la calle.
- **Salita del fondo** = la cola de la variable condición. El médico te manda ahí a esperar un resultado.

Estás en la salita del fondo. Llega tu resultado y la enfermera te llama (`signal`). Sos el primero de la salita: eso está garantizado.

**Pero no entrás directo al consultorio.** Salís a la **sala de espera general**, y mientras estabas adentro **entró alguien de la calle** que está antes que vos. Ese pasa primero y **se lleva el último turno del día**.

Cuando por fin entrás: con `if` no volvés a mirar el tablero y pedís un turno que ya no existe. Con `while` mirás, ves que no queda, y te volvés a sentar.

### La regla

```
while (no puedo avanzar) wait(cv);        ← SIEMPRE while, no if
```

### Traza 1: el semáforo simulado (el ejemplo de la teoría)

```
// ❌ MAL                                   // ✔ BIEN
monitor Semaforo {                          monitor Semaforo {
  int s = 1; cond pos;                        int s = 1; cond pos;

  procedure P ()                              procedure P ()
    { if (s == 0) wait(pos);                    { while (s == 0) wait(pos);
      s = s - 1;                                  s = s - 1;
    }                                           }

  procedure V ()                              procedure V ()
    { s = s + 1;                                { s = s + 1;
      signal(pos);                                signal(pos);
    }                                           }
}                                           }
```

| # | Quién | Qué hace | `s` | Estado |
|---|---|---|---|---|
| 1 | **A** | `P()`: `s != 0` → `s = s-1` | **0** | A tiene el semáforo |
| 2 | **B** | `P()`: `s == 0` → `wait(pos)` | 0 | **B duerme** (soltó el monitor) |
| 3 | **A** | `V()`: `s = s+1`; `signal(pos)` | **1** | **B despierta**… pero va a la cola de entrada |
| 4 | — | A sale del monitor. **B todavía no ejecutó** | 1 | |
| 5 | **D** | llega **nuevo**, entra al monitor libre. Ve `s == 1` → no espera → `s = s-1` | **0** | **D se coló** |
| 6 | **B** | por fin entra. Con `if` **no re-chequea**: `s = s - 1` | **−1** | 💥 |

`s = −1`, y **D y B creen los dos que tienen el semáforo**. Un valor negativo viola la definición de semáforo.

Con `while`, en el paso 6 B vuelve a mirar, ve `s == 0` y se duerme de nuevo. ✔

### Traza 2: el buffer limitado

El mismo error con otra cara. Con capacidad `n = 1`:

```
procedure depositar (typeT datos)
  { if (cantidad == n) wait(not_lleno);        // ← if, MAL
    buf[libre] = datos;  libre = (libre+1) mod n;  cantidad++;
    signal(not_vacio); }
```

| # | Quién | Qué pasa | `cantidad` |
|---|---|---|---|
| 1 | — | El buffer está lleno | **1** |
| 2 | **P1** | `depositar()`: `cantidad == n` → `wait(not_lleno)` | 1 |
| 3 | **Consumidor** | `retirar()`: saca el dato, `cantidad--`, `signal(not_lleno)` | **0** |
| 4 | — | **P1 despierta pero todavía no ejecuta** | 0 |
| 5 | **P3** | productor **nuevo**, entra, ve `cantidad == 0` → deposita | **1** |
| 6 | **P1** | entra. Con `if` no re-chequea: **deposita igual** | **2** 💥 |

`cantidad = 2` con capacidad 1: se pisó un dato y el contador quedó mintiendo.

> **En criollo:** *el `signal` no te promete que la condición se cumple ahora; te promete que en algún momento se cumplió.*

---

### La excepción: Passing the Condition va con `if`

No es una inconsistencia: **es otro problema**.

```
monitor Semaforo {
  int s = 1, espera = 0;  cond pos;

  procedure P ()  { if (s == 0) { espera++;  wait(pos); }    // ← if, y está BIEN
                    else s = s - 1; }

  procedure V ()  { if (espera == 0)  s = s + 1;
                    else { espera--;  signal(pos); } }       // ← NO incrementa s
}
```

La línea clave: en la rama del traspaso **`s` NO se incrementa**. El permiso no vuelve al pozo, **se entrega directo**.

| # | Quién | Qué hace | `s` | `espera` |
|---|---|---|---|---|
| 1 | **A** | `P()`: `s != 0` → `s = s-1` | **0** | 0 |
| 2 | **B** | `P()`: `s == 0` → `espera++`, `wait(pos)` | 0 | **1** |
| 3 | **A** | `V()`: `espera != 0` → `espera--`, `signal(pos)`. **`s` queda en 0** | **0** | 0 |
| 4 | **D** | llega nuevo, entra. Ve `s == 0` → `espera++`, `wait(pos)`. **Se anota, no se cuela** | 0 | **1** |
| 5 | **B** | entra. Sale del `if` y **no hace nada más** | 0 | 1 |

**D no pudo colarse** porque `s` nunca volvió a 1: el estado jamás dijo "disponible". Y en el paso 5 B **no hace `s = s-1`** —esa línea está en el `else`—: el que liberó ya hizo la contabilidad en su nombre.

> **Por qué el `if` es seguro:** la condición que te hizo esperar **no puede haber sido restaurada por otro**, porque el estado nunca pasó por "disponible". Te lo transfirieron directo: nadie te lo pudo robar.
>
> Es la misma idea que Passing the Baton con semáforos — *el recurso no se libera, se entrega*, y por eso la variable de estado no vuelve a "libre" mientras haya cola.

En la analogía del consultorio: el que sale **escribe tu nombre en el turno** antes de llamarte. El de la calle mira el tablero, ve **"ocupado"**, y se sienta.

---

### El test mecánico

No hay que razonar todo esto cada vez. Mirá tu propio código **justo después del `wait`**:

| ¿Qué hay después del `wait`? | Va con |
|---|---|
| Código tuyo que **modifica variables permanentes** (`s--`, `cantidad++`, `nr++`…) | **`while`** |
| **Nada**, o sólo leer lo que te dejaron asignado | **`if`** |

Si vos tenés que modificar el estado, es que el estado **todavía no está arreglado** — y si no está arreglado, alguien te lo pudo cambiar en el hueco.

> **No hay default seguro.** Poner `while` "por las dudas" en un Passing the Condition te deja durmiendo **para siempre**: el estado quedó en "ocupado" justamente porque te lo entregaron a vos. Hay que saber cuál es.

### Las dos formas que vas a escribir

```
// (1) CONDICIÓN BÁSICA — "esperá a que haya lugar / datos / no haya escritores"
while (condición mala)  wait(cv);
...modifico el estado...

// (2) PASSING THE CONDITION — "atender por orden de llegada"
if (no puedo)  { push(cola, id);  wait(priv[id]); }
// y el que libera NO devuelve el estado a "libre"
```

La (1) aparece en buffers, contadores de recursos y lectores/escritores. La (2), en todo lo que diga **orden de llegada**.


## 6. Las técnicas de sincronización

Este es el corazón para resolver ejercicios. Son seis, y la teoría las presenta con un ejemplo canónico cada una.

### 6.0 Antes que nada: qué NO se puede usar en la práctica

La teoría lista tres operaciones sobre variables condición y aclara que **NO SON USADAS EN LA PRÁCTICA**:

| Operación | Qué haría |
|---|---|
| `empty(cv)` | `true` si la cola de `cv` está vacía |
| `wait(cv, rank)` | El proceso se demora en la cola **ordenado ascendentemente** por `rank` |
| `minrank(cv)` | Devuelve el mínimo ranking de los demorados |

> **Esto es clave para el parcial.** Varias soluciones de la teoría las usan — y al lado está siempre **la misma solución sin ellas**. Las que hay que saber escribir son las segundas.
>
> Tu caja de herramientas real es: **`wait(cv)`, `signal(cv)`, `signal_all(cv)`** y las variables de estado que declares vos.

Y de ahí sale el sustituto universal:

> **Si no podés preguntar `empty` ni ordenar con `rank`, entonces llevás vos un contador de cuántos están esperando y una cola propia, y usás un arreglo `cond cv[N]` para despertar a uno puntual.**

---

### 6.1 Sincronización por condición básica

La más simple: una variable de estado, un `while`, un `signal`.

**Ejemplo canónico: buffer limitado (productor/consumidor).**

```
monitor Buffer_Limitado
{ typeT buf[n];
  int ocupado = 0, libre = 0, cantidad = 0;
  cond not_lleno, not_vacio;

  procedure depositar (typeT datos)
    { while (cantidad == n) wait(not_lleno);      // ← while
      buf[libre] = datos;
      libre = (libre + 1) mod n;
      cantidad++;
      signal(not_vacio);
    }

  procedure retirar (typeT &resultado)
    { while (cantidad == 0) wait(not_vacio);      // ← while
      resultado = buf[ocupado];
      ocupado = (ocupado + 1) mod n;
      cantidad--;
      signal(not_lleno);
    }
}
```

Comparalo con la versión de semáforos del mismo problema (`sem vacios = n; sem llenos = 0;`):

| Semáforos | Monitor |
|---|---|
| `sem vacios = n` y `sem llenos = 0`, cruzados | `int cantidad` (el estado) + `cond not_lleno`, `cond not_vacio` |
| El semáforo **cuenta y bloquea** a la vez | La variable **cuenta**, la `cond` **bloquea**. Trabajos separados |
| Con P productores hacía falta un `mutex` para el índice | **No hace falta**: la exclusión mutua es implícita |

> **El patrón:** *variable de estado + `while (condición mala) wait(cv)` + `signal(cv)` cuando cambiás el estado en favor del otro.*

---

### 6.2 Passing the Condition

**Cuándo:** cuando necesitás decidir **vos** quién sigue, en vez de dejar que el despertado re-chequee. Típicamente porque no tenés `empty` ni podés permitir que alguien se cuele.

**El mecanismo:** un **contador de cuántos están esperando**, que reemplaza al `empty` prohibido.

```
// Simulación de semáforos SIN usar empty()
monitor Semaforo
{ int s = 1, espera = 0;  cond pos;

  procedure P ()
    { if (s == 0) { espera++;  wait(pos); }       // ← if, no while
      else s = s - 1;
    }

  procedure V ()
    { if (espera == 0) s = s + 1;
      else { espera--;  signal(pos); }            // le PASO la condición
    }
}
```

Lo que hay que ver:

| Línea | Qué significa |
|---|---|
| `espera++` antes del `wait` | Me anoto. Este contador **es** el `empty` que no tengo |
| `wait` con **`if`** | No re-chequeo: quien me despierte ya me va a haber dejado todo hecho |
| `if (espera == 0) s = s+1` | **No hay nadie esperando** → el permiso se guarda en la variable de estado |
| `else { espera--; signal(pos); }` | **Hay alguien** → le paso el permiso directo. **Fijate que `s` NO se incrementa**: el recurso nunca vuelve a estar "disponible", pasa de mano en mano |

> **Es Passing the Baton, con otro nombre y otro mecanismo.** La misma regla de oro: **el recurso no se libera, se entrega**, y por eso la variable de estado no vuelve al valor "libre" durante el traspaso.

---

### 6.3 Variables condición privadas (`cond cv[N]`)

**Cuándo:** cuando tenés que despertar a **un proceso determinado**, no a "alguno" ni "al primero de la cola".

Es el análogo exacto del `sem priv[N] = ([N] 0)` de semáforos: **un timbre con nombre por proceso**. Y como la elección la hacés vos, va siempre junto con una **cola propia** donde llevás el orden que te interesa.

**Ejemplo canónico: alocación SJN (Shortest Job Next).**

La versión *con* `wait(cv, rank)` y `empty` (la que **no** podés usar):
```
monitor Shortest_Job_Next
{ bool libre = true;  cond turno;
  procedure request (int tiempo)
    { if (libre) libre = false;
      else wait(turno, tiempo);            // ← rank prohibido
    }
  procedure release ()
    { if (empty(turno)) libre = true;      // ← empty prohibido
      else signal(turno);
    }
}
```

La versión **con variables condición privadas** (la que **sí** va):
```
monitor Shortest_Job_Next
{ bool libre = true;
  cond turno[N];                            // un timbre por proceso
  cola espera;                              // el orden lo llevo yo

  procedure request (int id, int tiempo)
    { if (libre) libre = false;
      else { insertar_ordenado(espera, id, tiempo);   // ordeno por tiempo
             wait(turno[id]);                          // ← if, no while
           }
    }

  procedure release ()
    { if (empty(espera)) libre = true;                 // empty de MI cola, no de la cond
      else { sacar(espera, id);
             signal(turno[id]);                        // despierto a ESE
           }
    }
}
```

Tres cosas para notar:

1. **`empty(espera)` sí se puede**: es una cola tuya, no una variable condición.
2. **`libre` no vuelve a `true` cuando hay alguien esperando** → es Passing the Condition otra vez.
3. **El `wait` va con `if`**, por lo mismo.

> **La receta general, y sirve para casi cualquier ejercicio de orden:**
> ```
> cond cv[N];        // un timbre por proceso
> cola espera;       // el orden que el enunciado pida (FIFO, por prioridad, …)
> bool/int estado;   // si el recurso está libre o cuántos hay
> ```
> Cambiar de "orden de llegada" a "el trabajo más corto primero" es cambiar **una sola línea**: cómo insertás en la cola.

---

### 6.4 Broadcast signal (`signal_all`)

**Cuándo:** cuando un cambio de estado habilita a **muchos procesos a la vez**, y no a uno.

**Ejemplo canónico: lectores y escritores.** Cuando un escritor termina, **todos** los lectores demorados pueden pasar juntos (los lectores no se estorban entre sí).

```
monitor Controlador_RW
{ int nr = 0, nw = 0;                       // lectores y escritores activos
  cond ok_leer, ok_escribir;

  procedure pedido_leer ()
    { while (nw > 0) wait(ok_leer);
      nr = nr + 1;
    }
  procedure libera_leer ()
    { nr = nr - 1;
      if (nr == 0) signal(ok_escribir);     // el último lector habilita a UN escritor
    }
  procedure pedido_escribir ()
    { while (nr > 0 OR nw > 0) wait(ok_escribir);
      nw = nw + 1;
    }
  procedure libera_escribir ()
    { nw = nw - 1;
      signal(ok_escribir);                  // a UN escritor
      signal_all(ok_leer);                  // a TODOS los lectores
    }
}
```

> **El criterio:** `signal` cuando el recurso alcanza para **uno** (escribir es excluyente). `signal_all` cuando alcanza para **todos** los que esperan (leer es compartido).
>
> Y como es `signal_all` con Signal and Continue, **los `wait` van con `while`**: se despiertan todos, pero entran de a uno y cada uno tiene que re-verificar que la condición siga valiendo.

**El mismo problema con Passing the Condition** (la otra solución de la teoría) agrega `dr` y `dw` — *demorados* para leer y para escribir — y cada procedure decide explícitamente a quién le pasa el turno, actualizando `nr`/`nw` **en nombre del que despierta**:

```
procedure libera_escribir ()
{ if (dw > 0) { dw--; signal(ok_escribir); }      // le paso el turno a un escritor
  else { nw--;
         if (dr > 0) { nr = dr;  dr = 0;          // ¡les actualizo el contador a ELLOS!
                       signal_all(ok_leer); }
       }
}
```

Fijate `nr = dr; dr = 0;` — el que libera **hace las cuentas por los que despierta**. Esa es la firma de Passing the Condition: el despertado se encuentra el trabajo hecho y por eso no re-chequea.

---

### 6.5 Covering conditions

**Cuándo:** cuando no sabés cuál de los que esperan puede avanzar, así que **despertás a todos y que cada uno se fije**.

**Ejemplo canónico: reloj lógico.** Procesos que se duermen N ticks.

```
monitor Timer
{ int hora_actual = 0;
  cond chequear;                            // UNA sola cond "que cubre" a todos

  procedure demorar (int intervalo)
    { int hora_de_despertar;
      hora_de_despertar = hora_actual + intervalo;
      while (hora_de_despertar > hora_actual) wait(chequear);
    }

  procedure tick ()
    { hora_actual = hora_actual + 1;
      signal_all(chequear);                 // despierto a TODOS en cada tick
    }
}
```

La teoría lo marca explícitamente: **ineficiente** → *"mejor usar wait con prioridad o variables condition privadas"*. En cada tick se despiertan los N procesos, chequean, y casi todos se vuelven a dormir.

La versión con **variables condición privadas** (la que sí se puede usar en la práctica) mantiene una cola ordenada por hora de despertar y despierta sólo a los que corresponde:

```
monitor Timer
{ int hora_actual = 0;
  cond espera[N];
  colaOrdenada dormidos;

  procedure demorar (int intervalo, int id)
    { int hora_de_despertar = hora_actual + intervalo;
      Insertar(dormidos, id, hora_de_despertar);
      wait(espera[id]);
    }

  procedure tick ()
    { int aux, idAux;
      hora_actual = hora_actual + 1;
      aux = verPrimero(dormidos);
      while (aux <= hora_actual)
         { sacar(dormidos, idAux);
           signal(espera[idAux]);
           aux = verPrimero(dormidos);
         }
    }
}
```

> **Covering conditions es la solución "fuerza bruta"**: correcta, fácil de escribir, ineficiente. Si el ejercicio no pide eficiencia, sirve. Si la pide, o si querés mostrar que entendiste, van las condiciones privadas.

---

### 6.6 Rendezvous

**Cuándo:** cuando **dos procesos tienen que esperarse mutuamente**, en varias etapas. Es una barrera entre dos: ninguno sigue hasta que llegaron los dos.

**Ejemplo canónico: el peluquero dormilón.**

El enunciado, corto: una peluquería con dos puertas. El peluquero atiende de a uno; si no hay nadie, duerme. Si llega un cliente y el peluquero duerme, lo despierta y se sienta. Si el peluquero está ocupado, el cliente espera en otra silla. Terminado el corte, el peluquero abre la puerta de salida y **la cierra recién cuando el cliente se fue**.

Las tres esperas cruzadas:

1. El peluquero espera que llegue un cliente **y** el cliente espera que el peluquero esté disponible.
2. El cliente espera que el peluquero termine (le abre la puerta).
3. El peluquero espera que el cliente se haya ido, antes de cerrar.

```
monitor Peluqueria {
  int peluquero = 0, silla = 0, abierto = 0;
  cond peluquero_disponible, silla_ocupada, puerta_abierta, salio_cliente;

  procedure corte_de_pelo() {              // lo llama el CLIENTE
      while (peluquero == 0) wait(peluquero_disponible);
      peluquero = peluquero - 1;
      signal(silla_ocupada);               // "ya me senté"
      wait(puerta_abierta);                // espero a que termine
      signal(salio_cliente);               // "me fui"
   }
   procedure proximo_cliente() {           // lo llama el PELUQUERO
       peluquero = peluquero + 1;
       signal(peluquero_disponible);       // "estoy disponible"
       wait(silla_ocupada);                // espero que se siente
   }
   procedure corte_terminado() {           // lo llama el PELUQUERO
       signal(puerta_abierta);             // "terminé, salí"
       wait(salio_cliente);                // espero que se vaya
   }
}

Process Cliente[id: 0..N-1] { ... Peluqueria.corte_de_pelo(); ... }

Process Peluquero { int i;
   for i = 0..N-1 { Peluqueria.proximo_cliente();
                    // corta el pelo
                    Peluqueria.corte_terminado(); }
}
```

> **La firma del rendezvous:** los `signal` y `wait` aparecen **de a pares cruzados** — cada proceso avisa e inmediatamente espera el acuse del otro (`signal(X); wait(Y);`). Si ves esa alternancia, es un rendezvous.

---

## 7. Tabla de decisión: qué técnica usar

| Si el problema… | Técnica | Forma |
|---|---|---|
| Sólo hay que esperar una condición sobre el estado | **Condición básica** | `while (mal) wait(cv);` + `signal` al cambiar el estado |
| Hay que garantizar un **orden** o evitar que alguien se cuele | **Passing the Condition** | contador de demorados + `if (…) wait(cv);` + el que libera actualiza el estado por el otro |
| Hay que despertar a **un proceso puntual** (por id, por prioridad) | **Condición privada** | `cond cv[N]` + cola propia con el criterio de orden |
| Un cambio habilita a **muchos a la vez** | **Broadcast** | `signal_all(cv)` + `while` en los `wait` |
| No se sabe cuál puede avanzar | **Covering condition** | una `cond` para todos + `signal_all` + `while` (ineficiente) |
| **Dos procesos se esperan mutuamente** por etapas | **Rendezvous** | pares cruzados `signal(X); wait(Y);` en los dos lados |

---

## 8. Los errores que se penalizan

**(1) `if` en vez de `while` en la condición básica.** *El* error de la unidad. Con Signal and Continue, entre el `signal` y el despertar alguien puede colarse. Ejemplo de la teoría: el semáforo simulado queda en −1.

**(2) `while` en vez de `if` en Passing the Condition.** El error inverso, y también está mal: si el que libera ya te actualizó el estado y además re-chequeás, **te volvés a dormir para siempre** (el estado quedó en "ocupado" justamente porque te lo entregaron a vos).

**(3) Usar `empty(cv)`, `wait(cv,rank)` o `minrank(cv)`.** **No se usan en la práctica.** Si necesitás esa funcionalidad: contador propio + cola propia + `cond cv[N]`.

**(4) Suponer que `signal` tiene memoria.** Un `signal` sin nadie esperando **se pierde**. Toda la información persistente va en **tus variables de estado**, nunca en la variable condición.

**(5) Poner `P`/`V` o un mutex adentro de un monitor.** La exclusión mutua es **implícita**. Si escribís un candado adentro de un monitor, no entendiste el mecanismo (y podés generar deadlock: `wait` suelta el monitor, pero no soltaría tu mutex).

**(6) Que un procedure toque variables que no son permanentes, locales ni parámetros.** No está permitido: todo el estado compartido vive **adentro** del monitor.

**(7) Olvidar el `signal` al cambiar el estado.** Si `depositar` no hace `signal(not_vacio)`, el consumidor duerme para siempre aunque haya datos. **Cada vez que cambiás el estado en favor de alguien, avisale.**

---

## 9. Chuleta

| Concepto | En una línea |
|---|---|
| Exclusión mutua | **Implícita**, gratis. No la escribís |
| Sincronización | **Explícita**, con `cond`. Es todo tu trabajo |
| `wait(cv)` | Siempre duerme, **y suelta el monitor** |
| `signal(cv)` | Despierta al primero de la cola. **Sin nadie esperando, se pierde** |
| `signal_all(cv)` | Despierta a todos; la cola queda vacía |
| Disciplina | **Signal and Continue** (el que señaliza sigue) |
| Por eso… | **`while`, no `if`** — salvo en Passing the Condition |
| Prohibidas en la práctica | `empty(cv)`, `wait(cv,rank)`, `minrank(cv)` |
| El sustituto universal | contador de demorados + cola propia + `cond cv[N]` |
| Passing the Condition | **El recurso no se libera, se entrega.** El que libera actualiza el estado por el que despierta |
| Equivalencias con semáforos | `cond cv[N]` ≡ `sem priv[N] = ([N] 0)` · Passing the Condition ≡ Passing the Baton |

---

## 10. Mapa de ejemplos de la teoría

| Ejemplo | Técnica que ilustra |
|---|---|
| `TOTAL` (contador de productos) | Monitor mínimo: EM implícita, sin `cond` |
| Corralón de materiales *(sólo en Clase 4)* | Ídem: un procedure sin sincronización |
| Buffer de un dato con `while (not ok)` afuera | **Contraejemplo**: busy waiting, lo que NO hay que hacer |
| Buffer de un dato con `cond P, C` | La misma cosa bien resuelta con variables condición |
| Simulación de semáforos (`if` vs `while`) | La regla del `while` |
| Simulación de semáforos con `espera` | **Passing the Condition** (sin `empty`) |
| Alocación SJN | `wait` con prioridad → **variables condición privadas** |
| Buffer limitado | **Condición básica** (productor/consumidor) |
| Lectores y escritores | **Broadcast signal** → y la variante con Passing the Condition |
| Reloj lógico | **Covering conditions** → `wait` con prioridad → condiciones privadas |
| Peluquero dormilón | **Rendezvous** |
| Scheduling de disco *(sólo en Teoría 4)* | Aplicación completa: monitor separado vs. monitor intermedio |

---

## 11. Dato: bloquearse adentro de un monitor

Material más de teórico que de práctico, pero es justo el tipo de pregunta de oral o multiple choice.

> **La regla que resume todo: el único mecanismo que SUELTA el monitor al bloquearte es `wait`.**
>
> Todo lo demás que te bloquee estando adentro —un `P`, un `wait` en **otro** monitor, una espera de I/O— te deja **durmiendo con la puerta cerrada por dentro**. Y el que podía despertarte necesita esa puerta.

### Monitor dentro de monitor (*nested monitor problem*)

La práctica lo **contempla**: *"la única forma de comunicar datos **entre monitores** o entre un proceso y un monitor es por medio de invocaciones al procedimiento del monitor"*. Se puede — pero hay una mina.

**`wait` suelta el monitor en el que estás, y sólo ése.** Si estás adentro de A, llamás a un procedure de B y hacés `wait` adentro de B, **B se libera pero A queda tomado con vos adentro**.

```
monitor A {
   procedure f () { ...
                    B.esperar();      ← me duermo adentro de B
                    ... }             ← A sigue TOMADO mientras duermo
}
monitor B {
   cond c;
   procedure esperar () { wait(c); }
   procedure avisar ()  { signal(c); }
}
```

| # | Quién | Qué pasa |
|---|---|---|
| 1 | **P1** | `A.f()` → entra a A. **A queda tomado** |
| 2 | **P1** | `B.esperar()` → `wait(c)` → duerme. **Suelta B, NO suelta A** |
| 3 | **P2** | Quiere `A.g()`, que en algún momento haría `B.avisar()` |
| 4 | **P2** | Se bloquea **en la puerta de A**, tomada por P1 |
| 5 | — | Nadie llama nunca a `B.avisar()`. **P1 duerme para siempre** |

**El que podía despertarte necesita pasar por el monitor que vos tenés tomado mientras dormís.**

**Los otros tres problemas:**

- **Espera circular.** Un proceso toma A→B y otro B→A: se trancan. Mismo deadlock que pedir dos recursos en órdenes distintos. **La solución es la misma: todos piden los monitores en el mismo orden.**
- **Pérdida de concurrencia.** Mientras trabajás adentro de B seguís reteniendo A, y nadie puede usar A para nada.
- **Se rompe el razonamiento por invariante.** El invariante de A vale al entrar y al salir de cada procedure; si en el medio te vas a otro monitor, quedás con el invariante **a medio armar** y expuesto — peor si B termina llamando de vuelta a A (los monitores en general **no son reentrantes**).

> **Regla práctica:** nunca te bloquees mientras tenés otro monitor tomado. Si tenés que llamar a otro monitor desde adentro de uno, que la llamada sea **corta y que no pueda hacer `wait`**. Mejor todavía: hacé la segunda llamada **desde el proceso**, no desde adentro del primer monitor. Sin anidar.

### ¿Semáforos adentro de un monitor?

Técnicamente se escribe. Conceptualmente es un error y prácticamente es peligroso.

| | ¿Suelta el monitor al bloquearse? |
|---|---|
| `wait(cv)` | **Sí.** Está diseñado para eso |
| `P(s)` | **No.** Es ajeno al monitor, no sabe que existe |

```
monitor M {
   sem s = 0;
   procedure esperar () { P(s); }        ← me duermo SIN soltar el monitor
   procedure avisar ()  { V(s); }
}
```

P1 llama `esperar()` y se bloquea en `P(s)` **con el monitor tomado**. P2 quiere llamar a `avisar()` para hacer el `V(s)` y **no puede entrar**. Deadlock — el mismo del monitor anidado, con otro disfraz.

**Y aunque no se bloqueara, sobra:** un `sem mutex = 1` adentro de un monitor es redundante por definición, porque dos procedures del mismo monitor nunca ejecutan a la vez. Ponerlo le dice al corrector que no entendiste el mecanismo.

**Dato histórico:** los monitores **se implementan con semáforos** en la capa de abajo (uno para la exclusión mutua de entrada, otros para las colas de las variables condición). El semáforo está **debajo** del monitor, no adentro.

> **La frase para el oral:** *adentro de un monitor sólo te bloqueás con `wait`, y sólo sobre una variable condición de ese mismo monitor.*

---

## 12. Diferencias entre los dos PDFs

Los dos archivos son casi idénticos. Las diferencias reales:

| | Teoría 4 | Clase 4 (filminas usadas en clase) |
|---|---|---|
| Ejemplo del **corralón de materiales** (servidor de presupuestos) | — | ✔ |
| **Scheduling de disco** (slides 37–44) | ✔ | **no aparece** |
| Links a los audios MP4 | ✔ | — |

El bloque de **scheduling de disco** es el único tema grande que está en la teoría y **no se dio en clase**. Es un ejemplo de aplicación (políticas SST / SCAN / CSCAN, y el diseño de "monitor separado" vs "monitor intermedio"), y usa `empty` y `minrank`, que no se usan en la práctica. Lo que conviene llevarse, si sólo llevás una idea:

> **Monitor separado:** el usuario hace `pedir(cil)` → accede al disco → `liberar()`. Problema: el protocolo de 3 pasos es visible, y si un proceso no lo respeta, el scheduling falla.
> **Monitor intermedio:** el usuario hace **un solo llamado** (`usar_disco(...)`) y el monitor habla con el driver. La existencia del scheduling se vuelve **transparente** y no hay protocolo que el usuario pueda romper.
