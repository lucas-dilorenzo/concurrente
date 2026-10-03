# Práctica 1 — Ejercicio 2

> **Enunciado.** Realice una solución concurrente de grano grueso (utilizando `<>` y/o `<await B; S>`) para el siguiente problema: dado un número `N`, verifique cuántas veces aparece ese número en un arreglo de longitud `M`. Escriba las pre-condiciones que considere necesarias.

---

## 0. El punto de partida: la solución secuencial

Nadie diseña concurrente desde cero. Se arranca por la versión de un solo proceso:

```
int total = 0;

for j = 0 .. M-1
  { if (A[j] == N)
       total = total + 1;
  }
```

Esto **no se tira**. Es el algoritmo, y se conserva casi intacto. Lo concurrente es sólo *quién ejecuta qué iteraciones* y *qué pasa con el acumulador*.

---

## 1. La receta (5 pasos)

| Paso | Pregunta | Respuesta en este ejercicio |
|---|---|---|
| 1 | ¿Cuál es la solución secuencial? | El `for` de arriba |
| 2 | ¿Las iteraciones son independientes entre sí? | **Sí** — mirar `A[5]` no depende de haber mirado `A[4]`. Por eso se pueden repartir. |
| 3 | ¿Cómo reparto las iteraciones entre P procesos? | Bloques contiguos (ver §3) |
| 4 | Variable por variable: ¿local o compartida? ¿se escribe? | Ver tabla §5 |
| 5 | ¿Algún proceso tiene que **esperar** a otro? | **No** → este ejercicio no lleva `await` |

Los pasos 1, 2 y 3 son razonamiento secuencial puro. El único genuinamente concurrente es el 4.

---

## 2. Qué significa `Process Buscador [id: 0..P-1]`

No declara *un* proceso: declara **P procesos que ejecutan el mismo código**, y lo único que los distingue es el valor de `id`. Con `P = 3` equivale a escribir:

```
Process Buscador_0 { int id = 0;  ...mismo cuerpo... }
Process Buscador_1 { int id = 1;  ...mismo cuerpo... }
Process Buscador_2 { int id = 2;  ...mismo cuerpo... }
```

**Consecuencia clave:** si el código es idéntico para todos, la única forma de que cada proceso haga algo distinto es **calcular qué le toca a partir de su `id`**. De ahí salen las líneas `desde` / `hasta`.

Es el mismo patrón que `Process Puerta[id: 0..3]` del Ejemplo 2 de la explicación.

---

## 3. El reparto del trabajo

Hay **un solo arreglo `A`** compartido por los P procesos. Lo que se reparte son los **índices**:

```
M = 12, P = 3      A = [ 7  3  7  1 | 9  7  2  7 | 7  0  4  7 ]
                    índice 0..11      ↑            ↑           ↑
                                  Buscador 0   Buscador 1  Buscador 2
                                   j: 0..3      j: 4..7     j: 8..11
```

Cada bloque mide `M/P` elementos. El proceso `id` se saltea los `id` bloques anteriores:

```
desde = id * (M/P)              // salteo id bloques
hasta = desde + (M/P) - 1       // desde + tamaño del bloque, menos 1
```

El `-1` es porque `hasta` es **inclusivo**: si el bloque arranca en 4 y mide 4, son los índices 4, 5, 6, 7 → el último es `4 + 4 - 1 = 7`.

| id | `desde = id*4` | `hasta = desde+3` | índices |
|---|---|---|---|
| 0 | 0 | 3 | 0, 1, 2, 3 |
| 1 | 4 | 7 | 4, 5, 6, 7 |
| 2 | 8 | 11 | 8, 9, 10, 11 |

**Lo que siempre hay que verificar en un reparto:** cobertura total + disjunto (sin agujeros ni solapamiento).

---

## 4. Solución

```
// ---------- PRECONDICIONES ----------
//   M > 0,  P > 0,  P <= M
//   M mod P == 0                        M debe ser divisible por P para que los bloques
//                                        cubran todos los elementos del arreglo, sin que
//                                        ninguno quede sin asignar.  (detalle en §6)
//   A[0..M-1] inicializado antes de crear los procesos
//   A y N NO se modifican durante la ejecución (sólo lectura)
//   total == 0 antes de crear los procesos
//   cada proceso tiene un id único en 0..P-1

int A[M];             // arreglo compartido, sólo lectura
int N;                // valor a buscar, sólo lectura
int total = 0;        // compartida — resultado final
int P;                // cantidad de procesos

Process Buscador [id: 0..P-1]
{ int parcial = 0;                       // LOCAL a cada proceso
  int desde = id * (M / P);
  int hasta = desde + (M / P) - 1;

  for j = desde .. hasta
    { if (A[j] == N)
         parcial = parcial + 1;          // local  -> SIN <>
    }

  < total = total + parcial >;           // compartida y escrita por P procesos -> CON <>
}
```

### Justificación (esto es lo que suma puntos)

> `parcial` es local a cada proceso, por lo que no requiere `<>`. `A` y `N` son compartidas pero de sólo lectura, no hay interferencia posible. `total` es compartida y escrita por los P procesos: el incremento se descompone en Load/Add/Store, así que debe ser atómico. Se acumula localmente y se hace **una sola** sección crítica por proceso (P en total) en lugar de una por aparición.

---

## 5. Decisión de granularidad, variable por variable

| Variable | ¿Compartida? | ¿Se escribe? | ¿`<>`? | Por qué |
|---|---|---|---|---|
| `A[j]`, `N` | sí | **no** | No | Sólo lectura: nadie las modifica, no hay interferencia |
| `j`, `desde`, `hasta` | no (local) | sí | No | Cada proceso tiene su propia instancia |
| `parcial` | no (local) | sí | **No** | Ídem — mismo caso que `Parcial` en el Ejemplo 2 (puertas) |
| `total` | **sí** | **sí** | **Sí** | Load/Add/Store: sin `<>` se solapan y queda `total < Σ parciales` |

### El error que se penaliza

```
for j = desde .. hasta
  { if (A[j] == N) < total = total + 1 >; }   // <- ejecuta <> UNA VEZ POR APARICIÓN
```

Es **funcionalmente correcto**, pero es un error conceptual: serializa los procesos en cada coincidencia. Con el acumulador local, la sección crítica se ejecuta **exactamente P veces** en todo el programa, sin importar cuántas apariciones haya.

> **Regla del Ejemplo 2:** usar variables locales siempre que se pueda y reservar `<>` sólo para lo estrictamente compartido. Cada `<>` reduce la concurrencia.

---

## 6. Por qué la precondición `M mod P == 0`

### Definición corta (la que hay que saber de memoria)

> **`M` debe ser divisible por `P` para que los bloques cubran todos los elementos del arreglo, sin que ninguno quede sin asignar.**

El cierre *"sin que ninguno quede sin asignar"* es lo que la vuelve a prueba de repreguntas: deja explícito que el problema son los **elementos huérfanos**, no que los bloques queden de distinto tamaño.

Versión de bolsillo: **"el resto no lo mira nadie"**.

### Por qué

Probar el mismo reparto con `M = 10, P = 3`. Ahí `M/P = 3` (división entera, se pierde el resto):

| id | `desde = id*3` | `hasta = desde+2` | índices |
|---|---|---|---|
| 0 | 0 | 2 | 0, 1, 2 |
| 1 | 3 | 5 | 3, 4, 5 |
| 2 | 6 | 8 | 6, 7, **8** |

**El índice 9 no lo mira nadie.** Si `A[9] == N`, el resultado sale mal — y falla en silencio.

### En criollo (para acordarse)

> **Si `M` no es múltiplo de `P`, la división entera se come el resto y los últimos elementos del arreglo no se los asigna nadie.**

Gancho para memorizar: **"el resto no lo mira nadie"**, con el contraejemplo `M = 10, P = 3` → el índice 9 queda huérfano.

### Ojo con cómo se justifica

Decir *"es para que el arreglo se divida en partes iguales"* **no alcanza**: eso dice qué queremos, no qué se rompe si no pasa. Una precondición no es una preferencia estética, es la condición bajo la cual el algoritmo es **correcto**.

- Si al preguntar "¿y si no se cumple?" la respuesta es *"queda desparejo"* → no era una precondición.
- Si la respuesta es *"da mal"* → sí lo es.

Acá **da mal**: el conteo sale menor al real.

### Las tres versiones según dónde la pongas

**a) Comentario al lado de la precondición (corta):**

```
// M mod P == 0    -> sin esto, la división entera M/P descarta el resto
//                    y los últimos elementos no los recorre ningún proceso
```

**b) Justificación escrita (para entregar):**

> `M mod P == 0` garantiza que los P bloques de tamaño `M/P` cubran el arreglo completo. Si no se cumple, la división entera descarta el resto y los últimos `M mod P` elementos no quedan asignados a ningún proceso, con lo cual el conteo resulta menor al real. Es una precondición de **correctitud**, no de balanceo: el algoritmo no da "desparejo", da **mal**.

**c) Si te lo preguntan oral (una frase):**

> "Porque el reparto usa división entera. Si `M` no es múltiplo de `P` el resto se pierde y esos elementos finales no los recorre nadie, así que el resultado sale incorrecto. Si quiero evitar la precondición, uso reparto intercalado."

El remate final es el que suma: muestra que la limitación es de **la elección de reparto**, no del problema.

---

## 6 bis. Regla general para escribir precondiciones

Cuando escribas una, completá mentalmente esta frase:

> *"Si esto no se cumple, entonces \_\_\_\_\_\_\_\_."*

| Precondición | Si no se cumple... |
|---|---|
| `M mod P == 0` | los últimos `M mod P` elementos no los recorre nadie → conteo incorrecto |
| `total == 0` al inicio | el resultado arranca corrido → conteo incorrecto |
| `P <= M` | hay procesos con bloque de tamaño 0 → no rompe, pero es trabajo inútil |
| `A` y `N` no se modifican | habría interferencia sobre datos que leo sin `<>` → resultado impredecible |
| `id` único por proceso | dos procesos recorren el mismo bloque y otro ninguno → conteo incorrecto |
| `M > 0`, `P > 0` | el reparto no tiene sentido (arreglo vacío / sin procesos) |

**Si no podés completar la frase con algo malo, esa precondición sobra.**

---

## 7. Repartos alternativos (los tres son válidos)

### a) Intercalado / cíclico — no necesita `M mod P == 0`

```
Process Buscador [id: 0..P-1]
{ int parcial = 0;
  int j = id;                            // arranco en mi id

  while (j < M)
    { if (A[j] == N) parcial = parcial + 1;
      j = j + P;                         // salto de a P
    }

  < total = total + parcial >;
}
```

```
M = 10, P = 3
Buscador 0 ->  0     3     6     9
Buscador 1 ->     1     4     7
Buscador 2 ->        2     5     8
```

Cobertura total, sin solapamiento, con cualquier `M` y `P`. **Una precondición menos.**

### b) Bolsa de tareas (*bag of tasks*) — reparto dinámico

```
int proximo = 0;                         // índice compartido

Process Buscador [id: 0..P-1]
{ int parcial = 0;
  int j;

  < j = proximo; proximo = proximo + 1 >;      // agarro el siguiente índice libre
  while (j < M)
    { if (A[j] == N) parcial = parcial + 1;
      < j = proximo; proximo = proximo + 1 >;
    }

  < total = total + parcial >;
}
```

### Criterio de elección

| Reparto | Tipo | Secciones críticas | Cuándo conviene |
|---|---|---|---|
| Bloques | estático | **P** | trabajo por elemento **uniforme y conocido** |
| Intercalado | estático | **P** | ídem, y `M` no divide exacto |
| Bolsa de tareas | dinámico | **M + P** | trabajo por elemento **variable o impredecible** |

Acá el trabajo por elemento es una comparación `A[j] == N`: constante y conocida. No hay desbalance que justifique pagar `M` secciones críticas en vez de `P` → **reparto estático**.

(La bolsa de tareas se gana su costo cuando no sabés cuánto tarda cada ítem — p. ej. "para cada elemento, verificar si es primo": el 3 tarda nada y el 999.999.937 tarda mucho. Ahí el reparto dinámico balancea la carga y las `M` secciones críticas se pagan solas.)

---

## 8. Detalle: quién lee el resultado

`total` recién es válido cuando terminaron los P procesos. Si el programa principal lo usa, hay que garantizarlo:

```
co Buscador(0) // Buscador(1) // ... // Buscador(P-1) oc;   // fork-join: espera a todos
// acá total ya tiene el resultado final
```

O, si los procesos van con `process` (background), sincronizando por condición:

```
int terminados = 0;
// al final de cada Buscador:   < terminados = terminados + 1 >;
// en el proceso que consume:   < await (terminados == P) >;
```

---

## 9. Chequeo de sanidad

Poner `P = 1`: entonces `id = 0`, `desde = 0`, `hasta = M-1`, un solo proceso recorre todo el arreglo y hace `total = 0 + parcial`.

**Queda el `for` secuencial original, letra por letra.** La versión concurrente es una *generalización* de la secuencial, no un algoritmo distinto.

---

## 10. Los tres chequeos del corrector

1. **¿El reparto cubre todo el arreglo sin solaparse?** → sí, dada `M mod P == 0`.
2. **¿Toda variable compartida y escrita está protegida?** → `total`, sí.
3. **¿No hay `<>` de más?** → también se penaliza. `<parcial = parcial + 1>` estaría funcionalmente bien pero es un error conceptual.
