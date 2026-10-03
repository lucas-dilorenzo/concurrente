# Teoría 2 — Locks y Barreras

> **Qué pregunta responde.** Teoría 1 te dio `<await B; S>` y te dijo: *"alto poder expresivo, pero el costo de implementación de la forma general es alto"*. Y ahí lo dejó. **Teoría 2 es la respuesta a "¿y cómo se implementa?"** — usando nada más que variables compartidas comunes.

Cambia el rol de los `< >`: en los ejercicios 1 a 5 de la Práctica 1 eran una **primitiva dada**; acá pasan a ser **lo que hay que construir**. Por eso los ejercicios 6 y 7 se sienten distintos: te sacan la herramienta y te piden fabricarla.

Todo se hace con una sola técnica: **busy waiting** (*spin*) — chequear repetidamente una condición hasta que sea verdadera.

| Busy waiting | |
|---|---|
| **Ventaja** | Se implementa con instrucciones de cualquier procesador |
| **Desventaja** | Ineficiente en multiprogramación: un procesador que hace spin podría estar haciendo algo útil |
| **Aceptable si** | Cada proceso corre en su propio procesador |

Esa desventaja es la que al final justifica que existan los semáforos.

---

# PARTE A — El problema de la Sección Crítica

## A.1 La plantilla

```
process SC[i = 1 to n]
{ while (true)
   { protocolo de entrada;      <-- esto hay que escribir
     sección crítica;
     protocolo de salida;       <-- y esto
     sección no crítica;
   }
}
```

> *"Las soluciones a este problema pueden usarse para implementar sentencias `await` arbitrarias."*

No es un problema más: es **el** problema. Si lo resolvés, tenés `<>` y `<await B; S>` gratis para todo lo demás.

## A.2 Las 4 propiedades

| # | Propiedad | Qué exige | Tipo |
|---|---|---|---|
| 1 | **Exclusión mutua** | A lo sumo un proceso está en su SC | safety |
| 2 | **Ausencia de deadlock** (livelock) | Si 2 o más intentan entrar, **al menos uno** tendrá éxito | safety |
| 3 | **Ausencia de demora innecesaria** | Si un proceso quiere entrar y los otros están en su SNC o terminaron, **no se le impide** entrar | safety |
| 4 | **Eventual entrada** | Un proceso que intenta entrar **eventualmente** lo hará | **liveness** |

Que la 4 sea de liveness explica por qué en todo el apunte aparece la coletilla *"se garantiza con scheduling fuertemente fair"* / *"alcanza con débilmente fair"*: **las propiedades de liveness siempre dependen de la política de scheduling.**

> **En criollo:**
> **1** = no entran dos.
> **2** = no se traban todos.
> **3** = no me hagas esperar si no hay nadie adentro.
> **4** = no me dejes esperando para siempre.

### Los dos pares que se confunden

**2 vs. 4 — la distinción más importante del apunte.**

| | Habla de | Falla cuando |
|---|---|---|
| **Ausencia de deadlock** | el **conjunto** | nadie progresa |
| **Eventual entrada** | el **individuo** | uno puntual nunca entra → **inanición** |

Una solución puede cumplir la 2 y violar la 4: el sistema avanza alegremente mientras un proceso se muere de hambre. Eso es exactamente lo que le pasa a los **spin locks**, y es la razón de ser de Tie-Breaker, Ticket y Bakery — existen **sólo** para conseguir la propiedad 4.

**La 3 es la que se pasa por alto.** Prohíbe demorar a alguien *sin motivo*. El caso clásico que la viola es la **alternancia estricta**: si el protocolo obliga a turnarse, cuando SC1 se va a su sección **no** crítica —y la SC queda libre— SC2 igual tiene que esperarlo. Nadie adentro, y sin embargo hay alguien bloqueado.

---

## A.3 La escalera de implementaciones

Cada peldaño existe porque el anterior tiene un defecto puntual.

### Peldaño 0 — Intento ingenuo (roto)

```
bool in1 = false, in2 = false;      # MUTEX deseado: ¬(in1 ∧ in2)

process SC1 { while(true) { in1 = true;  SC; in1 = false;  SNC; } }
process SC2 { while(true) { in2 = true;  SC; in2 = false;  SNC; } }
```

No cumple nada: **nadie mira la bandera del otro**. Levantar la mano no es pedir permiso.

### Peldaño 1 — Grano grueso, 2 procesos

Mirar la del otro y levantar la propia, en **una sola acción atómica**:

```
process SC1 { while(true) { await (not in2) in1 = true;  SC; in1 = false;  SNC; } }
process SC2 { while(true) { await (not in1) in2 = true;  SC; in2 = false;  SNC; } }
```

Que sea *una* acción es todo el punto: si mirás y después levantás, los dos pueden mirar antes de que alguno levante.

**La verificación de las 4 propiedades — así se escribe una demostración en el parcial:**

| Propiedad | Argumento |
|---|---|
| Exclusión mutua | Por construcción: el invariante `¬(in1 ∧ in2)` se mantiene |
| Ausencia de deadlock | Si hubiera deadlock, ambos estarían bloqueados en su protocolo de entrada → `in1` e `in2` serían true a la vez. **Imposible**: en ese punto ambas son false (lo son inicialmente, y cada uno la vuelve a false al salir) |
| Ausencia de demora innecesaria | Si SC1 está fuera de su SC o terminó, `in1` es false → la guarda de SC2 es true → puede entrar |
| Eventual entrada | Si SC1 no puede entrar, SC2 está en su SC (`in2` true). Un proceso en su SC eventualmente sale → `in2` pasa a false → la guarda se hace true. **Requiere scheduling fuertemente fair** |

El patrón: las tres primeras se demuestran **por contradicción o por invariante**; la cuarta siempre termina en *"depende del scheduling"*.

### Peldaño 2 — Cambio de variables: de 2 a n procesos

`in1`/`in2` no escalan. La técnica se llama **cambio de variables**:

> `lock = in1 ∨ in2` — en vez de *"¿está adentro el otro?"*, preguntar **"¿está tomado?"**

```
bool lock = false;

process SC[i = 1..n]
{ while (true)
   { await (not lock) lock = true;    # protocolo de entrada
     sección crítica;
     lock = false;                    # protocolo de salida
     sección no crítica;
   }
}
```

**El problema:** ese `await (not lock) lock = true` **sigue siendo grano grueso** — son dos accesos a `lock` que deben ser indivisibles. Estamos usando atomicidad para implementar atomicidad. **Razonamiento circular** → hay que pedírsela al hardware.

### Peldaño 3 — Spin Lock con Test-and-Set

```
bool TS(bool ok)
{ < bool inicial = ok;
    ok = true;
    return inicial; >
}
```

Hace dos cosas: **siempre pone `true`** y **devuelve lo que había antes**.

```
bool lock = false;

process SC[i = 1..n]
{ while (true)
   { while (TS(lock)) skip;      # protocolo de entrada
     sección crítica;
     lock = false;               # protocolo de salida
     sección no crítica;
   }
}
```

**Cómo se lee `while (TS(lock)) skip`** — confunde porque se testea el valor **viejo**:

| `lock` antes | `TS` devuelve | `lock` después | Qué pasa |
|---|---|---|---|
| `true` (ocupado) | `true` | `true` | El `while` sigue → **spin**. Escribió true sobre true: no cambió nada |
| `false` (libre) | `false` | `true` | El `while` corta → **entro**, y ya quedó tomado |

> **En criollo:** TS es *"manoteá el picaporte y decime si ya había otra mano"*. Si había, seguís esperando. Si no, ya lo agarraste.

Lo elegante: **tomar el lock y averiguar si estaba libre son la misma operación**. No hay ventana entre consultar y tomar.

**Propiedades:** cumple 1, 2 y 3. La 4 necesita fuertemente fair — aunque en la práctica **débilmente fair es aceptable**, porque rara vez todos intentan entrar al mismo tiempo.

### Peldaño 4 — Test-and-Test-and-Set

**Defecto de TS:** escribe siempre, aunque el valor no cambie. En un multiprocesador con caches, cada escritura invalida la de todos → contención de memoria.

**Arreglo:** mirar primero (barato, se lee de cache) y hacer TS sólo cuando *parece* libre.

```
while (lock) skip;              # espero mirando, sin escribir
while (TS(lock))
   while (lock) skip;
```

La contención **se reduce pero no desaparece**: cuando `lock` pasa a false, todos intentan el TS a la vez.

---

## A.4 Los algoritmos fair

**El problema que resuelven:** los spin locks **no controlan el orden** de los procesos demorados. Al liberarse el lock todos compiten de cero — el que hace más rato que espera no tiene ninguna ventaja. Los tres que siguen existen para conseguir la propiedad 4 con scheduling apenas **débilmente fair**.

### Tie-Breaker (Peterson) — 2 procesos

**Idea:** una variable por proceso para decir *"empecé a pedir"*, más `ultimo` para romper empates. **Se demora al último en llegar.**

```
bool in1 = false, in2 = false;   int ultimo = 1;

process SC1                              process SC2
{ while (true)                           { while (true)
   { ultimo = 1; in1 = true;                { ultimo = 2; in2 = true;
     await (not in2 or ultimo == 2);          await (not in1 or ultimo == 1);
     SC; in1 = false; SNC;                    SC; in2 = false; SNC;
   } }                                    } }
```

La guarda se lee: *"paso si el otro no está pidiendo, **o** si el último en llegar fue él"*.

Grano fino — **ojo que se dan vuelta las dos primeras líneas**:

```
process SC1 { while(true) { in1 = true; ultimo = 1;
                            while (in2 and ultimo == 1) skip;
                            SC; in1 = false; SNC; } }
```

**Logro:** ninguna instrucción especial, alcanza con débilmente fair.
**Costo:** generalizar a n procesos requiere `n-1` etapas, cada una un tie-breaker de a dos. Complejo y caro.

### Ticket — el número de la carnicería

```
int numero = 1, proximo = 1, turno[1:n] = ([n] 0);

process SC[i: 1..n]
{ while (true)
   { < turno[i] = numero; numero = numero + 1 >    # saco número
     < await (turno[i] == proximo) >               # espero que me llamen
     sección crítica;
     < proximo = proximo + 1 >                     # llamo al siguiente
     sección no crítica;
   }
}
```

Dos observaciones finas, que son **puro ASV**:

- **El `await` se puede hacer con busy waiting**, porque la expresión referencia **una sola** variable compartida (`proximo`).
- **`proximo = proximo + 1` NO necesita `<>`**: a lo sumo un proceso puede estar en su protocolo de salida a la vez, así que no hay con quién competir.

Lo único que sí necesita atomicidad es sacar el número → **Fetch-and-Add**:

```
FA(var, incr):  < temp = var; var = var + incr; return temp >

turno[i] = FA(numero, 1);
while (turno[i] != proximo) skip;
sección crítica;
proximo = proximo + 1;
sección no crítica;
```

**Problema potencial:** `numero` y `proximo` crecen sin límite. En la práctica se resetean a un valor chico.

### Bakery

**Por qué existe:** Ticket depende de `FA`. Sin esa instrucción hay que simular la asignación atómica con una SC — circular otra vez, y la solución puede dejar de ser fair.

**Idea:** cada proceso recorre los números de **todos los demás** y se auto-asigna uno mayor; después espera a que el suyo sea el más chico de los que esperan. **Los procesos se comparan entre ellos, no contra un contador global.**

```
int turno[1:n] = ([n] 0);

process SC[i = 1..n]
{ while (true)
   { < turno[i] = max(turno[1:n]) + 1 >
     for [j = 1 to n st j != i]  < await (turno[j] == 0 or turno[i] < turno[j]) >
     sección crítica;
     turno[i] = 0;
     sección no crítica;
   }
}
```

**No es implementable directamente:** calcular el máximo de n valores no es atómico, y el `await` referencia una variable compartida **dos veces** (viola ASV). Grano fino:

```
turno[i] = 1;                          # señala "empecé el protocolo de entrada"
turno[i] = max(turno[1:n]) + 1;
for [j = 1 to n st j != i]
   while (turno[j] != 0) and ((turno[i], i) > (turno[j], j)) skip;
sección crítica;
turno[i] = 0;
sección no crítica;
```

Dos trucos:
- **`turno[i] = 1` primero** avisa que arrancó, para que otro que llega no se auto-asigne un número más chico mientras él calcula el máximo.
- **`(turno[i], i) > (turno[j], j)`** es comparación **lexicográfica**: si dos sacaron el mismo número, desempata el id. Sin eso, dos procesos con el mismo número se esperarían mutuamente.

---

## A.5 Cuadro comparativo — Sección Crítica

| Solución | ¿Instrucción especial? | Fairness necesaria | Complejidad | Observación |
|---|---|---|---|---|
| Grano grueso con `await` | — | fuerte | trivial | Es la **especificación**, no la implementación |
| **Spin lock** (TS) | Test-and-Set | fuerte (débil aceptable) | mínima | La más usada en la práctica |
| Test-and-Test-and-Set | Test-and-Set | ídem | mínima | Reduce contención de memoria |
| **Tie-Breaker** | ninguna | **débil** | alta con n | `n-1` etapas |
| **Ticket** | Fetch-and-Add | **débil** | baja | Números ilimitados |
| **Bakery** | ninguna | **débil** | media-alta | Los procesos se comparan entre sí |

> **La tensión de todo el apunte:** o usás una instrucción especial del hardware (rápido y simple, pero no fair), o la reemplazás con más variables y más lógica (fair, pero complejo). No hay almuerzo gratis.

---

# PARTE B — Barreras

> **Barrera:** un punto de demora al que deben llegar **todos** los procesos antes de que **cualquiera** pueda continuar.

> **En criollo:** la **sección crítica separa** (nunca dos adentro). La **barrera junta** (nadie pasa hasta que estén todos).

**Para qué sirve:** algoritmos iterativos. Cada worker calcula su parte de una iteración, todos se esperan, y recién ahí arranca la siguiente — porque el paso `k+1` necesita los resultados completos del paso `k`.

**Detalle que genera todos los problemas:** las barreras normalmente **se reutilizan**, están adentro de un `while (true)`.

## B.1 Contador compartido

```
int cantidad = 0;

process Worker[i = 1 to n]
{ while (true)
   { código para implementar la tarea i;
     < cantidad = cantidad + 1 >;
     < await (cantidad == n) >;
   }
}
```

Con `FA`: `FA(cantidad, 1); while (cantidad != n) skip;`

### El defecto que lo mata

> **¿Cuándo se reinicia `cantidad` en 0 para la siguiente iteración?**

No tiene solución fácil:
- Si un worker resetea apenas pasa, los que todavía no salieron del `while (cantidad != n)` **quedan colgados para siempre** — la condición que esperaban desapareció.
- Si esperás a que salgan todos, no tenés cómo saber que salieron todos — que es justamente lo que la barrera debería decirte. **Circular.**

Encima hay contención de memoria: n procesos golpeando la misma variable.

## B.2 Flags y coordinador

**Idea:** en vez de una variable que todos tocan, **repartir el contador en n variables**, una por proceso, y agregar un proceso extra para que **cada Worker espere por una sola variable: la suya**.

```
int arribo[1:n] = ([n] 0), continuar[1:n] = ([n] 0);

process Worker[i = 1 to n]
{ while (true)
   { código para implementar la tarea i;
     arribo[i] = 1;                      # aviso que llegué
     < await (continuar[i] == 1) >;      # espero MI permiso
     continuar[i] = 0;                   # limpio MI flag
   }
}

process Coordinador
{ while (true)
   { for [i = 1 to n]
        { < await (arribo[i] == 1) >;    # espero a cada uno
          arribo[i] = 0;                 # y limpio SU flag
        }
     for [i = 1 to n] continuar[i] = 1;  # los libero a todos
   }
}
```

Con busy waiting:

```
Worker:       arribo[i] = 1;
              while (continuar[i] == 0) skip;
              continuar[i] = 0;

Coordinador:  for [i = 1 to n] { while (arribo[i] == 0) skip;  arribo[i] = 0; }
              for [i = 1 to n] continuar[i] = 1;
```

### La clave para leerlo: el principio de flags (Andrews)

1. **El proceso que espera un flag es el que lo limpia.**
2. **Un flag no se prende hasta que se sabe que está limpio.**

Verificalo: el Coordinador esperó `arribo[i]` → **el Coordinador** lo limpia. El Worker esperó `continuar[i]` → **el Worker** lo limpia. Nunca hay dos procesos escribiendo el mismo flag para lo mismo.

Y eso **resuelve el problema del reset**: limpiar dejó de ser un paso suelto que nadie sabe cuándo hacer, y pasó a ser parte del protocolo de cada uno.

Granularidad: cada `await` referencia **una sola** variable compartida → busy waiting sin problema (ASV).

> **Es el material del ejercicio 7 de la Práctica 1**, que pide explícitamente basarse en esta solución: ahí el coordinador, en vez de coordinar una barrera, reparte permisos de sección crítica.

**Los dos defectos:** requiere un **proceso (y procesador) extra**, y el tiempo del coordinador es **proporcional a n** (recorre los arribos secuencialmente).

## B.3 Árboles (combining tree barrier)

**Idea:** eliminar el proceso extra combinando los roles — que **cada Worker sea también coordinador** de un pedacito. Se arma un árbol: las señales de **arribo suben** y las de **continuar bajan**.

**Ganancia:** de O(n) a **O(log n)**. Es la opción para n grande.
**Lo feo:** los procesos **juegan roles distintos** (hoja / nodo interno / raíz), así que el código deja de ser el mismo para todos.

## B.4 Barrera simétrica (Butterfly)

**Objetivo:** que **todos ejecuten exactamente el mismo código**, sin roles. Se construye a partir de barreras simples de a dos:

```
W[i]::  < await (arribo[i] == 0) >;      W[j]::  < await (arribo[j] == 0) >;
        arribo[i] = 1;                           arribo[j] = 1;
        < await (arribo[j] == 1) >;              < await (arribo[i] == 1) >;
        arribo[j] = 0;                           arribo[i] = 0;
```

Leído con el principio de flags: espero que **mi** flag esté limpio (regla 2), lo prendo, espero el de **mi compañero**, y limpio **el de él** — porque fui yo quien lo esperó (regla 1).

**Cómo se combinan.** Si `n` es potencia de 2: **log₂n etapas**, y en la etapa `s` cada worker sincroniza con el que está a **distancia 2^(s-1)**.

```
Workers    1    2    3    4    5    6    7    8
Etapa 1    └─┘  └─┘  └─┘  └─┘        (distancia 1)
Etapa 2    └────┘    └────┘          (distancia 2)
Etapa 3    └──────────┘              (distancia 4)
```

Después de log₂n etapas cada worker sincronizó **transitivamente** con todos. Ese patrón de cruces le da el nombre de *butterfly*.

```
int E = log(N);
int arribo[1:N] = ([N] 0);

process P[i = 1..N]
{ int j;
  while (true)
   { // código anterior a la barrera
     for (etapa = 1; etapa <= E; etapa++)
        { j = (i-1) XOR (1 << (etapa-1));    // con quién me toca sincronizar
          while (arribo[i] == 1) skip;
          arribo[i] = 1;
          while (arribo[j] == 0) skip;
          arribo[j] = 0;
        }
     // código posterior a la barrera
   }
}
```

El `XOR` con una potencia de 2 da el compañero de cada etapa: dar vuelta el bit `s-1` del índice lleva exactamente al proceso a distancia `2^(s-1)`.

## B.5 Cuadro comparativo — Barreras

| Implementación | Costo | ¿Proceso extra? | ¿Código simétrico? | Problema |
|---|---|---|---|---|
| **Contador compartido** | O(1) espacio | No | Sí | **No se puede resetear**; contención |
| **Flags y coordinador** | O(n) tiempo | **Sí** | No | Proceso y procesador extra |
| **Combining tree** | **O(log n)** | No | **No** — roles distintos | Código más complejo |
| **Butterfly** | **O(log n)** | No | **Sí** | Requiere n potencia de 2 |

---

# El cierre: por qué existe Teoría 3

Los defectos del busy waiting, textual del último slide:

- Los protocolos son **complejos** y **no separan claramente** las variables de sincronización de las que computan resultados. (Mirá el Bakery: `turno[]` es medio contador, medio flag, medio prioridad.)
- Es **difícil diseñarlos y probar que son correctos**; la verificación se complica al crecer el número de procesos.
- Es **ineficiente en multiprogramación**: un procesador haciendo spin podría estar haciendo algo productivo.

> *"Necesidad de herramientas para diseñar protocolos de sincronización."*

El **semáforo** de Teoría 3 es esa herramienta: da exclusión mutua y sincronización por condición **sin busy waiting** —el proceso se bloquea de verdad— y con las variables de sincronización separadas de las de cómputo.

Puentes con lo que ya hiciste:
- El **semáforo general en 5** es el pool del **Ejercicio 4**.
- El **semáforo binario** es el `libre` del **Ejercicio 5(a)**.
- El **algoritmo Ticket** es la forma de la solución del **Ejercicio 5(b)** (orden de llegada).

---

# Frases para el oral

- El problema de la SC es **el** problema: resolverlo da `<>` y `<await B; S>` para todo lo demás.
- Las 3 primeras propiedades son **safety** y se demuestran por invariante o contradicción; la **eventual entrada** es **liveness** y siempre depende del **scheduling**.
- **Ausencia de deadlock** habla del conjunto; **eventual entrada**, del individuo. La diferencia entre las dos es la **inanición**.
- `while (TS(lock)) skip` testea el valor **viejo**: si estaba ocupado sigo girando, si estaba libre ya lo tomé.
- **Cambio de variables** (`lock = in1 ∨ in2`) es la técnica para pasar de 2 a n procesos.
- Los algoritmos **fair** existen sólo para conseguir la propiedad 4 con scheduling **débilmente** fair.
- **Principio de flags:** el que espera un flag es el que lo limpia; y no se prende hasta saber que está limpio.
- El contador compartido no sirve como barrera reutilizable porque **no hay un momento seguro para resetearlo**.
