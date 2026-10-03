# Práctica 2 — Ejercicio 10

> **Enunciado.** A una cerealera van **T camiones** a descargar trigo y **M camiones** a descargar maíz. Sólo hay lugar para que **7 camiones a la vez** descarguen, pero **no pueden ser más de 5 del mismo tipo** de cereal.
> **a)** Implemente una solución que use **un proceso extra que actúe como coordinador**. El coordinador debe atender a los camiones **según el orden de llegada**. Además, debe **retirarse cuando todos los camiones han descargado**.
> **b)** Implemente una solución que **no use procesos adicionales** (sólo camiones). **No importa el orden de llegada** para descargar. *Nota: maximice la concurrencia.*

---

## 0. Las restricciones: un contador por límite

Hay **dos límites simultáneos**, y cada uno es un semáforo contador:

| Límite | Semáforo |
|---|---|
| 7 camiones descargando a la vez | `sem cupoTotal = 7` |
| No más de 5 de trigo | `sem cupoTrigo = 5` |
| No más de 5 de maíz | `sem cupoMaiz = 5` |

Fijate que `5 + 5 = 10 > 7`: los límites por tipo **no alcanzan** para garantizar el total, y el total no garantiza los límites por tipo. Por eso hacen falta **los tres**.

**Los camiones descargan con los semáforos tomados**, y eso es correcto: acá el semáforo **es** el lugar de descarga. Ocuparlo mientras descargás es justamente lo que se quiere modelar.

---

# b) Sin coordinador — arrancá por acá

Es el más corto, y deja el esqueleto de restricciones armado.

Fijate lo que el ítem **no** pide: *"no importa el orden de llegada"*. **Sin orden, no hay traspaso ni cola ni semáforos privados.** Son sólo los tres contadores.

## Solución (b)

```
// PRECONDICIONES: T >= 0, M >= 0

sem cupoTotal = 7;        // lugares de descarga
sem cupoTrigo = 5;        // cupo de camiones de trigo
sem cupoMaiz  = 5;        // cupo de camiones de maíz

Process CamionTrigo [id: 1..T]        Process CamionMaiz [id: 1..M]
{ P(cupoTrigo);      // específico     { P(cupoMaiz);       // específico
  P(cupoTotal);      // general          P(cupoTotal);      // general
  descargar();                           descargar();
  V(cupoTotal);                          V(cupoTotal);
  V(cupoTrigo);                          V(cupoMaiz);
}                                      }
```

Sin `while(true)`: cada camión descarga una vez y se va. **Ningún mutex**: no hay estructura compartida, sólo cupos.

## La decisión que vale el ítem: el orden de los `P`

> **Primero el cupo específico, después el general.**

Si fuera al revés —`P(cupoTotal)` y después `P(cupoTrigo)`— un camión de trigo que no consigue cupo de su cereal quedaría bloqueado **reteniendo uno de los 7 lugares de descarga sin usarlo**. Un camión de maíz que sí podía entrar se quedaría afuera por un lugar que nadie está ocupando.

**Contraejemplo concreto:** 5 camiones de trigo descargando (`cupoTrigo = 0`, `cupoTotal = 2`). Llegan 2 de trigo más: con el orden invertido pasarían `P(cupoTotal)` (dejándolo en 0) y se bloquearían en `P(cupoTrigo)`. Llega uno de maíz: **se bloquea**, aunque las tres restricciones lo permitían (6 ≤ 7, trigo 5 ≤ 5, maíz 1 ≤ 5). La cerealera opera a 5 de sus 7 lugares.

Eso es exactamente lo que pide la nota *"maximice la concurrencia"*.

> **La regla general:** **el que no consigue lo que necesita debe bloquearse sin retener nada.** Pedí siempre de lo más restrictivo a lo más general.

*(El orden de los `V` es indiferente: ninguno bloquea.)*

---

# a) Con coordinador y orden de llegada

## Qué cambia

Los camiones **no desaparecen**: cambia **quién decide**.

| | (b) sin coordinador | (a) con coordinador |
|---|---|---|
| Quién toma los cupos | El camión | **El coordinador, en nombre del camión** |
| Qué hace el camión | Se sirve solo | **Pide permiso y espera que lo llamen** |
| Quién libera | El camión | El camión (no cambia) |
| Orden | El que agarre primero | **Estricto, por llegada** |

## La pieza no obvia: el coordinador se bloquea por vos

> **El coordinador hace los `P` de los cupos en nombre del camión.** Si no hay lugar, **el que se bloquea es el coordinador** — y como está bloqueado, no atiende a nadie más.

Eso es lo que hace cumplir el *"orden de llegada"*: **si el primero de la fila no puede entrar, nadie detrás lo pasa.** El estado del sistema es *dónde está parado el coordinador*.

## Las tres piezas del traspaso, más el dato

*"Según el orden de llegada"* activa siempre lo mismo: **cola** (el orden) + **semáforos privados** (despertar a uno puntual) + **candado** de la cola.

Y como el coordinador necesita saber **de qué cereal** es cada camión para tomar el cupo correcto, en la cola viaja el par `{id, tipo}`: **el semáforo despierta, el dato viaja aparte.**

## Solución (a)

```
// PRECONDICIONES: T >= 0, M >= 0
//                 camiones numerados 1..T (trigo) y T+1..T+M (maíz)

sem cupoTotal = 7;   sem cupoTrigo = 5;   sem cupoMaiz = 5;

cola llegada;                      // pares {id, tipo}, en orden de llegada
sem mutexLlegada = 1;              // T+M escritores
sem hayPedido = 0;                 // señal: "llegó un camión"
sem permiso[T+M] = ([T+M] 0);      // timbre con nombre de cada camión
sem termino = 0;                   // señal: "terminé de descargar"


Process CamionTrigo [id: 1..T]
{ P(mutexLlegada);
    push(llegada, {id, TRIGO});          // el DATO: quién soy y qué traigo
  V(mutexLlegada);
  V(hayPedido);                          // la SEÑAL

  P(permiso[id]);                        // espero que me habiliten

  descargar();

  V(cupoTotal);                          // libero los cupos que él tomó por mí
  V(cupoTrigo);
  V(termino);                            // le aviso que terminé
}

Process CamionMaiz [id: T+1..T+M]        // idéntico, con MAIZ y cupoMaiz
{ P(mutexLlegada);  push(llegada, {id, MAIZ});  V(mutexLlegada);
  V(hayPedido);
  P(permiso[id]);
  descargar();
  V(cupoTotal);  V(cupoMaiz);  V(termino);
}

Process Coordinador
{ int i;  registro c;

  for i = 1 to T+M {                     // reparto los T+M permisos, en orden
     P(hayPedido);                       // espero que llegue alguien
     P(mutexLlegada);
       pop(llegada, c);                  // el primero de la fila
     V(mutexLlegada);

     if (c.tipo == TRIGO) P(cupoTrigo);  // específico PRIMERO — me bloqueo YO
     else                 P(cupoMaiz);
     P(cupoTotal);                       // general DESPUÉS

     V(permiso[c.id]);                   // lo habilito
  }

  for i = 1 to T+M  P(termino);          // me retiro cuando TODOS descargaron
}
```

## Los dos `for` del coordinador

No son un capricho: son dos requisitos distintos del enunciado.

| Loop | Qué cumple |
|---|---|
| El primero (`T+M` permisos) | *"atender a los camiones según el orden de llegada"* |
| El segundo (`T+M` avisos de fin) | *"retirarse cuando todos los camiones **han descargado**"* |

Sin el segundo, el coordinador se iría cuando terminó de **repartir** permisos, no cuando los camiones terminaron de **descargar**. Son momentos distintos: el último camión habilitado recién está empezando.

---

## Justificación (esto es lo que suma puntos)

> Las dos restricciones simultáneas se modelan con tres semáforos contadores: `cupoTotal = 7`, `cupoTrigo = 5` y `cupoMaiz = 5`. Son necesarios los tres porque 5 + 5 > 7: ni los límites por tipo garantizan el total ni el total garantiza los límites por tipo. En ambos ítems se pide **primero el cupo específico y después el general**, para que un camión que no consigue cupo de su cereal no quede bloqueado reteniendo un lugar de descarga sin usarlo; el orden inverso produce demora innecesaria y viola la indicación de maximizar la concurrencia. En el ítem (b) no se pide ningún orden, de modo que alcanzan los tres contadores y no hace falta ni cola ni exclusión mutua. En el ítem (a), el orden de llegada exige registrarlo —no puede suponerse que los semáforos sean FIFO—, así que se usa una cola protegida por `mutexLlegada` donde cada camión deposita su identificador **y su tipo de cereal**, ya que el semáforo transmite un instante y no un dato, y un arreglo de semáforos privados `permiso[]` para habilitar a un camión determinado. El coordinador ejecuta los `P` de los cupos **en nombre del camión**: al bloquearse él, deja de atender y ningún camión posterior adelanta al primero de la fila, que es precisamente lo que garantiza el orden de llegada. La liberación de los cupos la hace cada camión al terminar. El coordinador itera T+M veces repartiendo permisos y luego espera T+M avisos de finalización, porque el enunciado exige que se retire cuando todos **hayan descargado**, no cuando terminó de repartir.

## Errores que se penalizan

**(1) Pedir el cupo general antes que el específico.** Demora innecesaria: te bloqueás reteniendo un lugar de descarga.

**(2) Usar un solo semáforo.** Con `cupoTotal = 7` solo, entrarían 7 de trigo. Con los de tipo solos, entrarían 10.

**(3) Meter cola y semáforos privados en el (b).** El ítem dice *"no importa el orden de llegada"*: toda esa maquinaria sobra y muestra que no leíste lo que **no** se pide.

**(4) Que el coordinador tome los cupos y los libere él mismo.** Los libera el camión, que es el que sabe cuándo terminó de descargar.

**(5) Un solo `for` en el coordinador.** Se retiraría al repartir el último permiso, no al terminar la última descarga.

**(6) Un semáforo global en vez de `permiso[T+M]`.** `V` despertaría a *alguno*, y se pierde el orden que costó registrar.

## Chequeos de sanidad

- **Contá:** en todo momento, camiones descargando ≤ 7, de trigo ≤ 5, de maíz ≤ 5.
- **(b) con T = M = 0:** no pasa nada. Con T = 3, M = 2: los 5 descargan juntos (5 ≤ 7, 3 ≤ 5, 2 ≤ 5).
- **(a):** `hayPedido` recibe T+M `V` y T+M `P`; `termino` lo mismo; cada `permiso[i]` exactamente un `V` y un `P`.
- **Sin deadlock:** el coordinador se bloquea sin retener ningún candado (suelta `mutexLlegada` antes de los `P` de cupo).
