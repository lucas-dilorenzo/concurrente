# Práctica 3 (Monitores) — Ejercicio 4

> **Enunciado.** Existen **N vehículos** que deben pasar por un **puente** de acuerdo con el **orden de llegada**. Considere que el puente **no soporta más de 50000 kg** y que **cada vehículo cuenta con su propio peso** (ningún vehículo supera el peso soportado por el puente).

---

## 0. Las consideraciones de la Práctica 3

Antes de escribir un monitor conviene tener presentes las reglas de la práctica, porque varias descartan soluciones que parecen naturales:

| Regla | Qué implica |
|---|---|
| Los monitores usan **signal and continue** | Vale la regla del `while`… salvo en Passing the Condition |
| A una `cond` **sólo** `wait`, `signal`, `signalall` | Nada de `empty(cv)`, `minrank(cv)` |
| **NO** `wait` con prioridades | El orden lo construís vos, con una cola propia |
| **NO** se puede saber cuántos hay encolados en una `cond` ni si está vacía | Pero **sí** podés consultar una **cola tuya** — es una estructura que declarás |
| **La única forma de comunicar datos** es por **invocaciones a los procedimientos** | Los datos entran y salen por **parámetros** |
| **No existen variables globales** | Todo el estado vive **adentro** del monitor |
| Maximizar la concurrencia · aprovechar la EM del monitor · sin busy waiting · el tiempo con `delay` | |

Las dos del medio son las que más cambian el hábito respecto de semáforos: **no hay arreglos compartidos afuera**, y **el peso del vehículo tiene que entrar por un parámetro**.

---

## 1. Qué cambia respecto de la versión con semáforos

Este problema con semáforos llevaría un `sem e = 1` para proteger el estado, y habría que acordarse de soltarlo antes de dormirse. Acá:

| Semáforos | Monitor |
|---|---|
| `sem e = 1` + `P(e)` / `V(e)` alrededor de todo | **Nada.** La exclusión mutua es implícita |
| `V(e)` antes de `P(priv[id])`, o deadlock | **`wait` suelta el monitor solo** |
| `sem priv[N] = ([N] 0)` | `cond cruzar[N]` |
| Passing the Baton | **Passing the Condition** (la misma idea) |

**Es más corto y más difícil de romper.** Desaparece el error más caro de la versión con semáforos: dormirse con el candado en la mano.

---

## 2. El análisis

### Variables permanentes: un número, no un booleano

La tentación es `bool hayLugar`. **No sirve**, y el motivo es el corazón del ejercicio:

> *"¿Hay lugar?"* **no es una pregunta de sí o no.** Con 20000 kg libres, hay lugar para un auto de 1000 y no lo hay para un camión de 40000. **La respuesta depende de quién pregunta.**

Por eso el estado es el número:

```
int peso_total = 0;      // kg arriba del puente ahora
int MAX_PESO = 50000;
cola colaDeEspera;       // pares {id, peso}, en orden de llegada
cond cruzar[N];          // un timbre con nombre por vehículo
```

**En la cola va el par `{id, peso}`, no sólo el id:** cuando alguien sale, el monitor tiene que decidir si el primero de la fila entra, y para eso necesita saber **cuánto pesa**.

### ¿`cond` sola o `cond[N]`?

El criterio **no** es si la condición es general. Es:

> **¿El que despierta tiene que ELEGIR a quién, o le da igual cuál sea?**

Acá el monitor despierta al primero de la fila, después al segundo si también entra, y **frena** en el tercero. Está eligiendo, uno por uno, y sabe exactamente quiénes. ⇒ **arreglo**.

*(Con una sola `cond` se podría, apoyándose en que su cola es FIFO, pero habría que argumentar que se mantiene sincronizada con la propia. Con el arreglo no hay nada que argumentar.)*

---

## 3. La decisión que define el ejercicio: el orden es estricto

> **Llega un auto de 1000 kg. Hay 20000 kg libres. Pero adelante suyo espera un camión de 40000 que no entra. ¿Pasa o espera?**
>
> **Espera.** *"De acuerdo con el orden de llegada"* es estricto: si el primero de la fila no entra, **nadie lo pasa**.

Eso tiene un costo real, y conviene decirlo en la justificación: **el puente puede quedar subutilizado**. No es un defecto de la solución; es lo que el enunciado pide.

Y tiene **dos consecuencias en el código**, que son las dos mitades de la misma idea:

### (a) Al llegar: te anotás aunque entres

No alcanza con preguntar *"¿entro por peso?"*. También hay que preguntar **"¿hay alguien esperando?"**:

```
if ((peso_total + peso > MAX_PESO) or (not empty(colaDeEspera)))
```

Sin la segunda mitad, el recién llegado **se cuela** delante de los que esperaban.

### (b) Al salir: frenás, no salteás

Mirás el primero de la fila. ¿Entra? Lo despertás. ¿El nuevo primero entra? Lo despertás. **¿No entra? Frenás.**

**No lo salteás para ver si el siguiente sí entra** — saltearlo es exactamente romper el orden de llegada.

---

## 4. Passing the Condition: por qué `if` y no `while`

Aplicá el test: **después del `wait`, ¿modificás variables permanentes?**

En esta solución, **no**. El que sale le suma el peso a `peso_total` **en nombre** del que despierta. Por eso:

- El despertado no tiene nada que re-chequear ni que actualizar ⇒ **`if`**.
- Y **`peso_total` nunca vuelve a un estado "disponible"** durante el traspaso, así que **nadie se puede colar** en el hueco entre el `signal` y el momento en que el despertado ejecuta.

Es la misma estructura que el semáforo con Passing the Condition de la teoría:

```
procedure P () { if (s == 0) { espera++;  wait(pos); }
                 else s = s - 1; }         ← la cuenta va en el ELSE
```

> **Las dos decisiones tienen que ser coherentes:** si el `wait` va con `if`, la actualización del estado va en el **`else`**. Si la pusieras después del `if` (o sea, también para el que se despierta), **el peso se contaría dos veces** y el puente se sobrecargaría.

---

## 5. Solución

```
// PRECONDICIONES: N > 0
//                 ningún vehículo pesa más de MAX_PESO (lo garantiza el enunciado)

monitor Puente {
   int peso_total = 0;                    // kg arriba del puente ahora
   int MAX_PESO   = 50000;
   cola colaDeEspera;                     // pares {id, peso}, en orden de llegada
   cond cruzar[N];                        // un timbre con nombre por vehículo

   procedure Entrar (int id; int peso)
   { if ((peso_total + peso > MAX_PESO) or (not empty(colaDeEspera)))
        { push(colaDeEspera, {id, peso});      // no entro, o hay fila: me anoto
          wait(cruzar[id]);                    // el que me despierte YA me sumó
        }
     else peso_total = peso_total + peso;      // entro directo: me sumo yo
   }

   procedure Salir (int peso)
   { registro v;
     peso_total = peso_total - peso;           // libero lo mío

     while ((not empty(colaDeEspera)) and
            (peso_total + verPrimero(colaDeEspera).peso <= MAX_PESO))
        { pop(colaDeEspera, v);
          peso_total = peso_total + v.peso;    // se lo sumo YO, en su nombre
          signal(cruzar[v.id]);                // y lo despierto
        }
   }
}


Process Vehiculo [id: 1..N]
{ int mi_peso;                             // cada vehículo trae el suyo
  Puente.Entrar(id, mi_peso);
  delay(cruce);                            // cruza — AFUERA del monitor
  Puente.Salir(mi_peso);
}                                          // sin while(true): cruza una vez
```

### Por qué cada pieza está donde está

| Decisión | Por qué |
|---|---|
| `delay(cruce)` **afuera** del monitor | Si se llamara adentro, el monitor quedaría ocupado todo el cruce y **pasaría uno solo a la vez** — se pierde toda la capacidad de 50000 kg |
| `mi_peso` viaja **por parámetro** | No hay variables globales, y los datos sólo se comunican por invocaciones a los procedures |
| `verPrimero` antes de `pop` | Hay que decidir **si** entra antes de sacarlo. Si no entra, se queda primero en la fila |
| El `while` del `Salir`, no un `if` | Cuando sale un camión de 40000 pueden entrar **varios** autos de golpe |
| Nada de `mutex` dentro del monitor | La exclusión mutua es **implícita**. Escribir un candado adentro muestra que no se entendió el mecanismo |

---

## 6. Justificación (esto es lo que suma puntos)

> El estado del puente se representa con un entero `peso_total` y no con un booleano, porque la pregunta *"¿hay lugar?"* depende del peso del vehículo que consulta: con 20000 kg libres entra un auto de 1000 y no entra un camión de 40000. El orden de llegada debe registrarse explícitamente —no se puede usar `wait` con prioridades ni consultar la cola de una variable condición—, por lo que se mantiene una cola propia con los pares `{id, peso}`: el peso es necesario para que el monitor pueda decidir, al liberarse capacidad, si el primero de la fila entra. Se usa un arreglo `cond cruzar[N]` porque hay que despertar a vehículos determinados y en un orden preciso, no a uno cualquiera. Un vehículo se anota también cuando entraría por peso pero **la fila no está vacía**: sin esa condición, un vehículo liviano se colaría delante de uno pesado que espera desde antes. La contrapartida es que el puente puede quedar subutilizado, lo cual es consecuencia directa del orden de llegada estricto que pide el enunciado. La solución aplica **Passing the Condition**: quien sale libera su peso y luego recorre la cola despertando a todos los que entren, **sumando el peso de cada uno en su nombre** y frenando en el primero que no entra —sin saltearlo, porque saltearlo rompería el orden—. Como el vehículo despertado no modifica ninguna variable permanente y `peso_total` nunca vuelve a un estado disponible durante el traspaso, el `wait` va con `if` y no con `while`: nadie puede colarse en el intervalo entre el `signal` y su ejecución. El cruce se realiza fuera del monitor para que varios vehículos puedan estar sobre el puente simultáneamente, que es el objetivo del enunciado.

---

## 7. Los errores que se penalizan

**(1) `bool hayLugar` como estado.** No modela el problema: la disponibilidad depende del peso del que pregunta.

**(2) Condición de espera sólo por peso.** Sin el `or (not empty(colaDeEspera))`, los vehículos livianos se cuelan y se rompe el orden de llegada.

**(3) Sumar el peso también después del `wait`.** Se cuenta dos veces y el puente se sobrecarga. Va en el **`else`**.

**(4) `while` en vez de `if` en el `wait`.** Con Passing the Condition, el despertado re-chequearía, vería que su peso ya está sumado y se dormiría de nuevo **para siempre**.

**(5) Un solo `signal` en el `Salir`.** Cuando se libera mucho peso pueden entrar varios; hace falta el ciclo.

**(6) Saltear al primero si no entra.** Rompe el orden de llegada — es justo lo que el enunciado prohíbe.

**(7) `delay(cruce)` adentro del monitor.** El monitor queda tomado durante todo el cruce: pasa **uno por vez** y la capacidad de 50000 kg no sirve de nada.

**(8) Un `mutex` dentro del monitor.** La exclusión mutua es implícita. Además, `wait` suelta el monitor pero no soltaría ese candado → deadlock.

**(9) Variables globales o arreglos compartidos fuera del monitor.** La práctica lo prohíbe: todo el estado vive adentro y los datos viajan por parámetros.

---

## 8. Chequeos de sanidad

- **`peso_total` nunca supera `MAX_PESO`** ni se vuelve negativo.
- **`peso_total` = suma de los pesos de los vehículos que están cruzando** en ese momento.
- **Cada `cruzar[i]` recibe a lo sumo un `signal`**, y sólo si el vehículo `i` se anotó.
- **La cola respeta el orden:** si el vehículo A se anotó antes que B, A es despertado antes que B.
- **Con un solo vehículo:** entra directo por el `else` (fila vacía y entra por peso), cruza, sale, y la fila queda vacía. ✔
- **Con todos los vehículos pesando más de 25000:** pasan de a uno, y la solución sigue siendo correcta.
- **Sin deadlock:** el `wait` suelta el monitor, así que siempre puede entrar otro a hacer `Salir`.

---

## 9. Los tres chequeos del corrector

1. **¿El estado es numérico y la condición de espera incluye "hay fila"?** Con sólo el peso, los livianos se cuelan.
2. **¿El que sale suma el peso en nombre del que despierta, y por eso el `wait` va con `if`?** Las dos decisiones tienen que ser coherentes: la actualización va en el `else`.
3. **¿Se frena en el primero que no entra, sin saltearlo?** Y `delay(cruce)` fuera del monitor, o pasa uno por vez.
