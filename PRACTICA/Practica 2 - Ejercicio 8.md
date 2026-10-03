# Práctica 2 — Ejercicio 8

> **Enunciado.** Una fábrica de piezas metálicas debe producir **T piezas por día**. Para eso cuenta con **E empleados** que se ocupan de producir las piezas **de a una por vez**. La fábrica **empieza a producir una vez que todos los empleados llegaron**. **Mientras haya piezas por fabricar, los empleados tomarán una y la realizarán.** Cada empleado **puede tardar distinto tiempo** en fabricar una pieza. Al finalizar el día, se debe conocer **cuál es el empleado que más piezas fabricó**.
>
> **a)** Implemente una solución asumiendo que **T > E**.
> **b)** Implemente una solución que contemple **cualquier valor de T y E**.

---

## 0. Un solo tipo de proceso

Tentación: modelar un proceso `Fábrica`. **No va.**

Aplicá el test de siempre: *¿decide algo, o sólo se usa?* La fábrica no elige, no reparte, no avisa. Los que llegan, toman piezas y las fabrican son **los empleados**. *"La fábrica empieza a producir"* es una forma de hablar: los que empiezan a producir son ellos, todos juntos.

> **Acá hay un único proceso: `Empleado[id: 1..E]`.**

## Las dos frases que definen el ejercicio

| Frase del enunciado | Qué significa |
|---|---|
| *"empieza a producir **una vez que todos los empleados llegaron**"* | **Barrera** |
| *"**mientras haya** piezas por fabricar, los empleados **tomarán una**"* | **Bolsa de tareas** |
| *"cada empleado puede **tardar distinto tiempo**"* | Confirma la bolsa y **descarta el reparto fijo** |

### Por qué esa tercera frase no es color

La alternativa obvia sería repartir de antemano:

```
// Lo que NO hay que hacer:
Process Empleado[id: 1..E]
{ for k = 1 to T/E  fabricarPieza();      // cada uno su cuota
}
```

Si todos tardaran lo mismo esto sería **óptimo y sin nada de sincronización**. Pero como tardan distinto, el rápido termina su cuota y **se queda parado** mientras el lento sigue: la fábrica produce al ritmo del más lento.

Por eso el trabajo se sirve **de a uno, a medida que se liberan**. Eso es la bolsa de tareas.

---

## 1. No toda sincronización es una espera

Veníamos buscando "esperas", y acá aparecen pocas. Conviene contar bien:

| Qué | ¿Es una espera? |
|---|---|
| Esperar a que lleguen los E empleados | **Sí** → barrera |
| Tomar una pieza de la bolsa | **No.** Nadie se bloquea: o hay y te la llevás, o no hay y te vas |
| Saber quién fabricó más | **No.** Es una decisión que toma uno solo al final |

> **Una sola espera y dos secciones críticas.** Si vas buscando semáforos-señal en la bolsa, te perdés.

---

## 2. ⚠️ El semáforo NO sirve como bolsa

Es el error más natural del ejercicio y conviene entender bien por qué falla:

```
sem piezas = T;
Process Empleado[id: 1..E]
{ while (true) { P(piezas);  fabricarPieza();  mias++; } }
```

Dos problemas, y el segundo es fatal:

1. **No podés consultar el valor de un semáforo** (la práctica lo prohíbe), así que no tenés forma de preguntar *"¿queda alguna?"*.
2. **`P` te bloquea cuando está vacío en lugar de avisarte que está vacío.** Cuando alguien toma la última pieza, los demás vuelven al `while`, hacen `P(piezas)` y **quedan demorados para siempre**.

> Y ojo: eso pasa **para cualquier T y E**, no sólo cuando T < E. Con el semáforo **el programa no termina nunca**.

**Lo que hace falta** es algo que permita *preguntar si queda trabajo* y *llevarse una unidad* sin que nadie se meta en el medio:

```
int piezas = T;           // un ENTERO sí se puede consultar
sem mutexBolsa = 1;       // para que preguntar y llevarse sean un solo acto
```

---

## 3. El truco que resuelve el loop

La trampa al escribirlo es esta: *"si la consulta tiene que estar adentro del candado, y la consulta es la condición del `while`, entonces el candado tiene que envolver todo el loop"*.

**No.** Y la salida es la misma que se usa en la barrera:

> **La consulta va adentro del candado; lo que sale afuera no es la variable compartida, es tu copia local de la respuesta.** El candado se toma y se suelta **en cada vuelta**.

```
// MAL                                  // BIEN
P(mutexBolsa);                          hayParaMi = true;
while (piezas > 0) {                    while (hayParaMi) {
   ...                                     P(mutexBolsa);
}                                             hayParaMi = (piezas > 0);
V(mutexBolsa);                                if (hayParaMi) piezas--;
                                           V(mutexBolsa);
// la condición del while lee la            if (hayParaMi) { ... }
// variable compartida SIN proteger      }
```

Es exactamente el mismo movimiento que `ultimo = (contador == E)` dentro del candado de la barrera.

---

## 4. Solución (a)

```
// PRECONDICIONES: T > E, E > 0
//                 todas las variables compartidas inicializadas antes de crear los procesos

int contador = 0;                        // empleados que llegaron
sem mutexBarrera = 1;                    // candado → arranca en 1
sem barrera = 0;                         // señal   → arranca en 0

int piezas = T;                          // LA BOLSA: lo que queda por fabricar
sem mutexBolsa = 1;                      // candado de la bolsa

int piezasFabricadas[1..E] = ([E] 0);    // cada empleado su casillero

int terminaron = 0;                      // empleados que terminaron de fabricar
sem mutexFin = 1;                        // candado de ese contador


Process Empleado [id: 1..E]
{ boolean ultimo, hayParaMi;             // LOCALES
  int mejor, i;                          // LOCALES

  // ── BARRERA: la fábrica arranca cuando llegaron todos ──
  P(mutexBarrera);
    contador++;
    ultimo = (contador == E);            // la decisión se toma ADENTRO
  V(mutexBarrera);
  if (not ultimo) P(barrera);            // los E-1 primeros esperan
  V(barrera);                            // cascada

  // ── BOLSA DE TAREAS ──
  hayParaMi = true;
  while (hayParaMi) {
     P(mutexBolsa);
       hayParaMi = (piezas > 0);         // consulta y resta: UN SOLO acto
       if (hayParaMi) piezas--;
     V(mutexBolsa);
     if (hayParaMi) {
        fabricarPieza();                 // AFUERA del candado: es lo caro
        piezasFabricadas[id]++;          // mi casillero: único dueño ⇒ sin candado
     }
  }

  // ── ¿SOY EL ÚLTIMO EN TERMINAR? ──
  P(mutexFin);
    terminaron++;
    ultimo = (terminaron == E);
  V(mutexFin);

  // ── INFORME: lo hace uno solo ──
  if (ultimo) {
     mejor = 1;
     for i = 2 to E
        if (piezasFabricadas[i] > piezasFabricadas[mejor]) mejor = i;
     Informe(mejor, piezasFabricadas[mejor]);
  }
}
```

### Por qué cada pieza está donde está

| Decisión | Por qué |
|---|---|
| `fabricarPieza()` **afuera** del candado | Es la operación larga. Adentro, los E empleados trabajan **de a uno** y la bolsa no sirve de nada |
| Consulta y resta **juntas** adentro | Si van por separado, dos empleados pueden ver la misma última pieza y llevársela los dos |
| Sólo se resta **si había** | `piezas` nunca se va a negativo |
| `piezasFabricadas[id]` **sin candado** | Un único dueño por casillero. Compartido, eso sí, para que el que informe pueda leerlos todos |
| El informe lo hace **el último en terminar** | Cuando él llega, por definición todos terminaron: los casilleros ya están finales |
| **No hay barrera** antes del informe | No hace falta: el contador `terminaron` + el `if` ya garantizan el momento correcto, y lo hace **uno solo** |

### Justificación (esto es lo que suma puntos)

> Se modela un único tipo de proceso, `Empleado`: la fábrica no toma decisiones, sólo es el ámbito. Se usa una **barrera** porque nadie empieza a producir hasta que llegaron los E: cada empleado incrementa un contador protegido y decide **dentro** de la sección crítica si es el último, guardándolo en una variable local para no leer sin protección una variable con E escritores. El trabajo se reparte como **bolsa de tareas** y no en bloques fijos, porque los empleados tardan distinto y un reparto de antemano dejaría ociosos a los rápidos. La bolsa es un **entero con candado** y no un semáforo: no se puede consultar el valor de un semáforo, y `P` bloquearía al empleado en vez de informarle que no queda trabajo, con lo cual el programa no terminaría. La consulta *"¿queda alguna?"* y el decremento ocurren en la misma sección crítica —si fueran separados, dos empleados podrían llevarse la misma pieza— y el resultado sale en una variable local, de modo que el candado se libera en cada vuelta y `fabricarPieza()`, que es la operación costosa, queda fuera: así los E trabajan realmente en paralelo. Cada empleado acumula en su propio casillero de `piezasFabricadas`, que por tener un único escritor no requiere exclusión mutua. El informe lo emite **el último en terminar el ciclo** —no el que se llevó la última pieza, que todavía puede estar fabricándola mientras otros siguen trabajando—, momento en el cual todos los contadores están definitivos.

---

## 5. Solución (b): es la misma

> *"Implemente una solución que contemple cualquier valor de T y E."*

**No hay que cambiar nada**, y saber justificarlo es todo el ítem.

Trazá **T = 3, E = 5**:

| Momento | `piezas` | Qué pasa |
|---|---|---|
| Los 5 pasan la barrera | 3 | |
| Tres empleados toman una c/u | 0 | `hayParaMi = true` → fabrican |
| Empleado 4 consulta | 0 | `hayParaMi = false` → **no resta**, suelta el candado, sale del ciclo |
| Empleado 5 consulta | 0 | ídem |

Los empleados 4 y 5 **no se bloquean en ningún lado**: retienen el candado durante una comparación, salen con 0 piezas y se suman al contador de terminados. Con **T = 0**, salen los E de la misma forma.

> **El "guardarail" ya está:** la línea `hayParaMi = (piezas > 0)` **es** la protección. Agregar otra verificación sería preguntar dos veces lo mismo.

### Por qué existe el ítem (b)

Para detectar soluciones que **no** tienen esa consulta. Con `sem piezas = T` y `P(piezas)`, los E−T empleados sobrantes quedan demorados **para siempre**. El ítem separa a quien puso la consulta adentro del candado de quien dio por sentado que siempre habría trabajo.

### Justificación del (b)

> La solución del ítem (a) contempla cualquier valor de T y E sin modificaciones. La consulta *"¿queda alguna pieza?"* y el *"me llevo una"* ocurren dentro de la misma sección crítica, y sólo se decrementa si había: por eso `piezas` nunca se vuelve negativo y ningún empleado se bloquea esperando trabajo que no existe. Con T < E, los E−T empleados que no alcanzan pieza salen del ciclo en su primera consulta con 0 piezas fabricadas; con T = 0, salen los E. El informe sigue siendo correcto porque lo emite el último en terminar el ciclo, y ese momento está definido para cualquier T. Una solución que usara un semáforo inicializado en T como bolsa fallaría acá: `P` bloquea cuando está vacío en lugar de informar que lo está, y los empleados sobrantes quedarían demorados para siempre.

> **Lección transversal:** cuando un ítem dice *"modifique para que contemple X"*, **no siempre hay código que cambiar**. A veces la respuesta correcta es *"ya está contemplado, y acá está por qué"*. Buscar un error que no existe hace perder tiempo y, peor, suele terminar en agregar cosas que rompen lo que funcionaba.

---

## 6. Los errores que se penalizan

**(1) Semáforo como bolsa.** `sem piezas = T` + `P(piezas)`. El programa no termina **nunca**, para ningún T y E.

**(2) El candado afuera del `while`.**
```
P(mutexBolsa);
while (piezas > 0) { ... }
V(mutexBolsa);
```
Si lo soltás adentro de una rama, en las vueltas siguientes tocás `piezas` **sin protección**; si no lo soltás, retenés el candado mientras fabricás y los E trabajan de a uno.

**(3) La condición del `while` leyendo la variable compartida.** `while (piezas > 0)` consulta sin candado. Va la copia local.

**(4) `fabricarPieza()` adentro de la sección crítica.** No rompe el resultado, pero **anula el paralelismo**: E empleados rinden como 1. Es el error más caro conceptualmente.

**(5) Decrementar sin consultar.** `piezas--` incondicional → se va a negativo.

**(6) Confundir "me llevé la última pieza" con "soy el último en terminar".** Quien toma la última pieza **recién empieza** a fabricarla, y otros pueden seguir trabajando en las suyas. Si informa ahí, lee casilleros que todavía se están llenando.

**(7) Reparto fijo `for k = 1 to T/E`.** Lo descarta explícitamente la frase *"cada empleado puede tardar distinto tiempo"*.

**(8) El informe sin `if`.** Si el bloque está suelto en el cuerpo del proceso, lo ejecutan los E → E informes. (Y una barrera antes **no lo arregla**: una barrera hace pasar a todos.)

**(9) Variables con nombre repetido.** Declarar `int id` adentro de `Empleado[id: 1..E]` pisa el identificador del proceso y todos escriben el mismo casillero. Lo mismo con reusar la variable del `for` externo en uno anidado.

**(10) El máximo mal calculado.** Comparar cada uno contra el propio contador, o contra un `max` que nunca se actualiza. Va: el mejor arranca en 1 y cada candidato se compara **contra el mejor que llevás**.

---

## 7. Chequeos de sanidad

- **Contá las piezas:** la suma de `piezasFabricadas[1..E]` tiene que dar **exactamente T**. Ni una más ni una menos.
- **`piezas` nunca es negativo**, y termina en 0.
- **Con E = 1:** el único empleado pasa la barrera solo, fabrica las T piezas de a una e informa. Colapsa al caso secuencial. ✔
- **Con T < E:** los E−T sobrantes salen en su primera consulta con 0. Sin bloqueos.
- **Exactamente un empleado ve `terminaron == E`** ⇒ exactamente un informe.
- **Sin deadlock:** ningún `P` se ejecuta reteniendo otro candado. Los tres candados (`mutexBarrera`, `mutexBolsa`, `mutexFin`) se toman y se sueltan sin bloquearse en el medio.

---

## 8. Los tres chequeos del corrector

1. **¿La bolsa es un entero con candado, y la consulta va adentro?** Con un semáforo el programa no termina. Con la consulta afuera, dos empleados se llevan la misma pieza.
2. **¿`fabricarPieza()` está fuera de la sección crítica?** Adentro, los E empleados rinden como uno y la bolsa de tareas pierde todo sentido.
3. **¿El informe lo hace uno solo, y es el último en TERMINAR?** No el que se llevó la última pieza, y no los E.
