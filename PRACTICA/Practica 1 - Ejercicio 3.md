# Práctica 1 — Ejercicio 3
                                                                                                
> **Enunciado.** Dada la siguiente solución de grano grueso:
> **(a)** Indicar si el código funciona para resolver el problema de Productor/Consumidor con un buffer de tamaño `N`. En caso de no funcionar, hacer las modificaciones necesarias.
> **(b)** Modificar el código para que funcione para `C` consumidores y `P` productores.

---

## 0. El código dado

```
int cant = 0;   int pri_ocupada = 0;   int pri_vacia = 0;   int buffer[N];

Process Productor::                          Process Consumidor::
{ while (true)                               { while (true)
   { produce elemento                           { <await (cant > 0); cant-->
     <await (cant < N); cant++>                   elemento = buffer[pri_ocupada];
     buffer[pri_vacia] = elemento;                pri_ocupada = (pri_ocupada + 1) mod N;
     pri_vacia = (pri_vacia + 1) mod N;           consume elemento
   }                                            }
}                                            }
```

**Respuesta corta: NO funciona.** Ni siquiera con un productor y un consumidor.

Es un buffer circular: `pri_vacia` marca dónde escribir, `pri_ocupada` dónde leer, y `cant` cuántos elementos hay. La idea está bien; el problema es **dónde** está puesta cada línea.

---

## 1. Primero, descartar las hipótesis fáciles

Antes de acusar a nadie conviene chequear lo obvio, porque las dos sospechas naturales son **falsas** y perder tiempo ahí es lo habitual.

**¿Hay carrera sobre `cant`?** No. `cant++` y `cant--` están **adentro** de los `< >`, así que son atómicos: no se pierde ningún incremento. Eso es exactamente lo que compra el `<>`.

**¿Puede haber deadlock?** No. El productor espera `cant < N` y el consumidor espera `cant > 0`. Con `N ≥ 1` esas dos condiciones **no pueden ser falsas al mismo tiempo**, así que siempre hay al menos un proceso habilitado para avanzar.

**Entonces, ¿dónde está el problema?** Tabla variable por variable, para el caso (a) — **1 productor, 1 consumidor**:

| Variable | Quién escribe | Quién lee | ¿Carrera? |
|---|---|---|---|
| `cant` | ambos | ambos | **No** — está dentro de `< >` |
| `pri_vacia` | solo Productor | solo Productor | **No** — dueño único |
| `pri_ocupada` | solo Consumidor | solo Consumidor | **No** — dueño único |
| `buffer[]` | Productor | Consumidor | **SÍ** ← acá está |

Ojo con esta tabla: en el **(a)** cada índice tiene un solo dueño, así que decir "faltan `<>` en los índices" es **incorrecto**. Eso recién se vuelve cierto en el **(b)**.

---

## 2. El bug: el contador va adelantado a los datos

Mirá el orden del productor:

```
<await (cant < N); cant++>        <- anuncia "hay un elemento"
buffer[pri_vacia] = elemento;     <- pero lo escribe DESPUES
```

Entre esas dos líneas hay una ventana en la que **`cant` miente**: dice que hay un elemento que todavía no existe. El invariante que el algoritmo necesita —`cant` = cantidad real de elementos en el buffer— se rompe.

### Traza 1 — el consumidor lee un slot no escrito

`N = 4`, todo inicializado en 0:

| # | Proceso | Acción | `cant` | Estado |
|---|---|---|---|---|
| 1 | Productor | `<await(cant<4); cant++>` | 1 | anuncia la `X`, no la escribió |
| 2 | Consumidor | `<await(cant>0); cant-->` → **pasa** | 0 | |
| 3 | Consumidor | `elemento = buffer[0]` | 0 | **lee basura** |
| 4 | Consumidor | `pri_ocupada = 1` | 0 | |
| 5 | Productor | `buffer[0] = X` | 0 | |
| 6 | Productor | `pri_vacia = 1` | 0 | |

**Dos errores de una sola intercalación:** el consumidor consumió basura, y la `X` quedó en `buffer[0]` con `pri_ocupada` ya en 1, así que **nunca se va a consumir**.

### Traza 2 — el productor pisa un slot no leído

El mismo pecado, simétrico. Buffer lleno: `cant = 4`, `pri_vacia == pri_ocupada == 0`, `buffer = [A, B, C, D]`:

| # | Proceso | Acción | `cant` | Estado |
|---|---|---|---|---|
| 1 | Consumidor | `<await(cant>0); cant-->` | 3 | libera el lugar, no leyó nada |
| 2 | Productor | `<await(cant<4); cant++>` → **pasa** | 4 | |
| 3 | Productor | `buffer[0] = E` | 4 | **pisa la `A`** |
| 4 | Consumidor | `elemento = buffer[0]` | 4 | lee `E`, debía leer `A` |

La `A` se perdió. Peor que la traza 1: acá `cant` queda **coherente**, así que el error es silencioso.

### En criollo

> **El contador reserva el lugar, pero el dato llega tarde.**
> `cant++` promete un elemento que todavía no está, y `cant--` libera un lugar del que todavía no se sacó nada.

Es el mismo pecado del Ejercicio 1 con otra cara: una operación que *conceptualmente* es una unidad ("meter un elemento") está partida en dos, y el interleaving se mete en el medio. La diferencia es que en el Ejercicio 1 la partía el hardware; acá **la partiste vos, en el código fuente**.

---

## 3. Arreglo A — reordenar (sirve sólo para el (a))

Si el problema es que `cant` va adelantado, la corrección mínima es **actualizarlo después del dato**:

```
Process Productor::                          Process Consumidor::
{ while (true)                               { while (true)
   { produce elemento                           { <await (cant > 0)>
     <await (cant < N)>                           elemento = buffer[pri_ocupada];
     buffer[pri_vacia] = elemento;                pri_ocupada = (pri_ocupada + 1) mod N;
     pri_vacia = (pri_vacia + 1) mod N;           <cant-->
     <cant++>                                     consume elemento
   }                                            }
}                                            }
```

**Por qué funciona con 1P/1C.** El `<await (cant < N)>` sólo espera, no toca `cant`. Como hay un único productor, una vez que pasa nadie más le puede robar el lugar —`cant` sólo puede bajar, que libera más espacio—, así que el slot sigue libre cuando escribe. Del otro lado, `cant` recién se incrementa **después** de que el dato está en el buffer, así que `cant > 0` ahora es una promesa verdadera.

**Ventaja:** productor y consumidor nunca se excluyen mutuamente. Trabajan de verdad en paralelo.

### Dos detalles que se penalizan

**1. El `cant--` va antes del `consume`.** Si se escribe así:

```
pri_ocupada = (pri_ocupada + 1) mod N;
consume elemento          <- puede tardar muchísimo
<cant-->                  <- y el slot sigue contando como ocupado
```

el consumidor ya sacó el dato y ya avanzó el índice: el lugar está libre de hecho, pero `cant` lo sigue contando durante todo el `consume`, y el productor se bloquea sin motivo.

Cómo justificarlo con precisión: **no es un error de correctitud.** `cant` queda inflado, que es el lado seguro (el productor usa *menos* buffer del que hay). Es una **pérdida de concurrencia** — el mismo criterio por el que la Clase 1 deja `produce`/`consume` fuera de los `< >`: *la operación larga nunca adentro de la reserva*.

**2. No poner `<>` alrededor del buffer.** `<buffer[pri_vacia] = elemento>` no hace falta: nadie compite por esa línea, el productor es el único que escribe ese slot y el consumidor no lo mira hasta el `cant++`. Es la regla del Ejercicio 2 — *cada `<>` reduce la concurrencia*, y ponerlo donde no hay carrera es un error conceptual.

---

## 4. Arreglo B — todo adentro del `<>` (sirve para (a) y (b))

La otra opción: que la condición y **toda** la acción que la consume viajen pegadas en una sola acción atómica.

```
Process Productor::                     Process Consumidor::
{ while (true)                          { while (true)
   { produce elemento                      { <await (cant > 0);
     <await (cant < N);                        elemento = buffer[pri_ocupada];
        buffer[pri_vacia] = elemento;          pri_ocupada = (pri_ocupada + 1) mod N;
        pri_vacia = (pri_vacia + 1) mod N;     cant--
        cant++                              >
     >                                       consume elemento
   }                                       }
}                                       }
```

Ahora no existe ninguna ventana: nadie puede observar un estado donde `cant` y el buffer no coincidan. Es la solución de la Clase 1 (slide 40).

---

## 5. El criterio para elegir: rompé el Arreglo A con dos productores

El Arreglo A parece mejor —más concurrente, menos invasivo—, pero **se cae apenas hay más de un productor**. `N = 4`, buffer vacío, `cant = 0`, `pri_vacia = 0`:

| # | Proceso | Acción | `pri_vacia` | `cant` |
|---|---|---|---|---|
| 1 | Productor A | `<await (cant<4)>` → pasa | 0 | 0 |
| 2 | Productor B | `<await (cant<4)>` → **también pasa** | 0 | 0 |
| 3 | A | `buffer[0] = X` | 0 | 0 |
| 4 | B | `buffer[0] = Y` → **pisa la `X`** | 0 | 0 |
| 5 | A | `pri_vacia = (0+1) mod 4` | 1 | 0 |
| 6 | B | `pri_vacia = (0+1) mod 4` → **el mismo 1** | 1 | 0 |
| 7 | A | `<cant++>` | 1 | 1 |
| 8 | B | `<cant++>` | 1 | **2** |

Desastre completo: la `X` se perdió, `pri_vacia` avanzó **un** lugar habiendo entrado **dos** elementos, y `cant` dice 2 cuando hay 1. El consumidor va a leer `buffer[1]`, que es basura.

El Arreglo B no tiene ese problema: como el `< >` es indivisible, el segundo productor no puede entrar hasta que el primero terminó, y ve `pri_vacia` ya avanzado.

| | Arreglo A (reordenar) | Arreglo B (todo adentro) |
|---|---|---|
| Correcto con 1P / 1C | Sí | Sí |
| Correcto con P / C | **No** | Sí |
| Concurrencia | Mayor: P y C se solapan | Menor: se serializan |
| Sirve para el (b) | No | **Sí, sin cambios** |

> **En criollo:** con **un solo dueño** por variable alcanza con **ordenar**. Apenas hay **dos dueños**, hay que **encerrar**.

**Para entregar conviene el Arreglo B**, porque resuelve (a) y (b) con el mismo código. Vale mencionar el A como la solución de máxima concurrencia para el caso 1P/1C, aclarando por qué no escala.

---

## 6. Parte (b) — P productores y C consumidores

El Arreglo B escala **agregando sólo el cuantificador**. Lo único nuevo es una declaración:

```
// ---------- PRECONDICIONES ----------
//   N > 0   (tamaño del buffer)
//   P > 0, C > 0
//   cant == 0, pri_vacia == 0, pri_ocupada == 0 antes de crear los procesos

int cant = 0;   int pri_ocupada = 0;   int pri_vacia = 0;   int buffer[N];

Process Productor [i: 1..P]                Process Consumidor [j: 1..C]
{ int elemento;              // LOCAL     { int elemento;              // LOCAL
  while (true)                              while (true)
   { produce elemento                        { <await (cant > 0);
     <await (cant < N);                          elemento = buffer[pri_ocupada];
        buffer[pri_vacia] = elemento;            pri_ocupada = (pri_ocupada + 1) mod N;
        pri_vacia = (pri_vacia + 1) mod N;       cant--
        cant++                                >
     >                                        consume elemento
   }                                        }
}                                         }
```

### `elemento` tiene que ser local — y es sutil

Si `elemento` fuera compartida, con dos consumidores pasa esto: el consumidor 1 hace `elemento = buffer[3]` y **sale** del `< >`; antes de que llegue a consumirlo, el consumidor 2 entra, hace `elemento = buffer[4]` y lo pisa. El consumidor 1 termina consumiendo el elemento del otro, y el suyo se pierde.

Justamente **porque el `consume` está afuera** del `< >`, esa variable queda expuesta. Lo mismo del lado del productor con lo que devuelve `produce`. Es el tipo de detalle que el corrector busca específicamente en el (b).

### Justificación (esto es lo que suma puntos)

> Las cuatro variables compartidas (`cant`, `pri_vacia`, `pri_ocupada`, `buffer`) se acceden **únicamente** dentro de acciones atómicas, de modo que ningún proceso puede observar un estado intermedio: es imposible que dos productores lean el mismo `pri_vacia`, o que un consumidor lea un slot todavía no escrito. `elemento` es local a cada proceso, con lo cual `produce` y `consume` —que quedan fuera de la sección crítica, para no serializar el trabajo pesado— no interfieren entre sí. El invariante `cant` = cantidad real de elementos en el buffer se mantiene en todo estado observable.

---

## 7. Decisión de granularidad, variable por variable

Para la solución final del (b):

| Variable | ¿Compartida? | ¿Se escribe? | ¿`<>`? | Por qué |
|---|---|---|---|---|
| `elemento` | no (local) | sí | No | Cada proceso tiene la suya. **Si fuera compartida se rompe** (§6) |
| `cant` | **sí** | **sí** | **Sí** | Load/Add/Store + es la condición de guarda de ambos lados |
| `pri_vacia` | **sí** | **sí** | **Sí** | En el (b) la comparten los P productores |
| `pri_ocupada` | **sí** | **sí** | **Sí** | En el (b) la comparten los C consumidores |
| `buffer[]` | **sí** | **sí** | **Sí** | Debe moverse junto con el índice y el contador, o `cant` miente |

**La razón de fondo por la que van todas en el *mismo* `<>`** y no en `<>` separados: no alcanza con que cada operación sea atómica por su cuenta. Lo que tiene que ser indivisible es la **transición completa de estado** — reservar el lugar, escribir el dato y contarlo. Un `<>` por línea deja exactamente las mismas ventanas que el código original.

---

## 8. Deadlock e inanición

**Deadlock: no puede haber.** `cant < N` y `cant > 0` no pueden ser falsas simultáneamente con `N ≥ 1`, así que siempre hay al menos un proceso habilitado.

**Inanición: sí es posible, pero no es un defecto del algoritmo.** Con P productores compitiendo por el mismo `< >`, uno puntual podría no entrar nunca si la política de scheduling es injusta. Eso depende de la **fairness** con que esté implementado el `await` (fairness incondicional / débil / fuerte, Teoría 1), no del diseño de la solución. En un parcial alcanza con nombrarlo y aclarar la distinción.

---

## 9. Chequeo de sanidad

Con `P = 1` y `C = 1`, la solución del (b) colapsa exactamente al Arreglo B del (a) — el cuantificador `[i: 1..1]` declara un solo proceso. Es el mismo chequeo que en el Ejercicio 2 con `P = 1`.

Otro, más útil: si en cualquier momento parás el programa **entre** dos acciones atómicas y contás los elementos que hay realmente en el buffer, tiene que dar exactamente `cant`. En el código original ese chequeo falla; en la solución final no puede fallar, porque no existe ningún punto observable entre la escritura y el conteo.

---

## 10. Los tres chequeos del corrector

1. **¿Detectó el bug correcto?** No es una carrera sobre `cant` (está protegida) ni un deadlock (imposible). Es que **`cant` se actualiza antes que el dato**, con dos síntomas: leer un slot no escrito y pisar un slot no leído.
2. **¿La corrección mantiene el invariante?** Condición + acción en la **misma** acción atómica. Un `<>` por línea no alcanza.
3. **¿El (b) contempla `elemento` local?** Es lo que separa una respuesta completa de una a medias, porque `produce`/`consume` quedan fuera del `< >`.
