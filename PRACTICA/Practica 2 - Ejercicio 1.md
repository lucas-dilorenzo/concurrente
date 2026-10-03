 # Práctica 2 — Ejercicio 1

> **Enunciado.** Existen N personas que deben ser chequeadas por un detector de metales antes de poder ingresar al avión.
> **(a)** Analice el problema y defina qué procesos, recursos y semáforos/sincronizaciones serán necesarios/convenientes.
> **(b)** Implemente una solución que modele el acceso de las personas a **un** detector (si está libre la persona lo puede utilizar; si no, debe esperar).
> **(c)** Modifique la solución para el caso en que haya **tres** detectores.
> **(d)** Modifique la solución anterior para el caso en que cada persona pueda pasar **más de una vez**, siendo **aleatoria** esa cantidad de veces.

---

## 0. Las reglas de la práctica

Están al principio del enunciado y **se corrigen**:

| Regla | Qué implica |
|---|---|
| Semáforos **declarados** e **inicializados** siempre | `sem s;` no existe |
| **No se puede setear ni consultar el valor de un semáforo** | No existe `if (mutex > 0)`. Sólo `P` y `V` |
| **Evitar busy waiting** | Nada de `while (cond) skip` |
| El tiempo se representa con **`delay`** | "usar el detector" es `delay(tiempo)` |

La segunda es la que más cambia cómo pensás: si necesitás **saber** cuántas unidades quedan, no le podés preguntar al semáforo — llevás un entero aparte protegido por un mutex.

---

## 1. La receta para modelar (sirve para toda la práctica)

El ítem (a) es, en realidad, el método que se reusa en los 12 ejercicios:

| # | Pregunta | Qué produce |
|---|---|---|
| 1 | ¿Qué entidades **actúan**? | Los `Process`, y cuántas instancias de cada tipo |
| 2 | ¿Qué se **comparte**? | Recursos y variables globales |
| 3 | ¿Cuántas **unidades** tiene cada recurso? | El **valor inicial** del semáforo |
| 4 | ¿Quién **espera** a quién, y por qué? | Las sincronizaciones |
| 5 | Cada sincronización, ¿de qué **tipo** es? | mutex (1) / contador (n) / señalización (0) |
| 6 | ¿Qué es **local** y qué **compartido**? | Declaraciones adentro vs. afuera del proceso |

### La trampa de la pregunta 1: recurso pasivo vs. proceso activo

> Si algo simplemente **se usa** → es un **recurso**, y se modela con un semáforo.
> Si **decide, atiende, ordena o avisa** → es un **proceso**.

Acá el detector **no hace nada**: la persona pasa por él. Es pasivo → **no** es un proceso.

Distinto es un Coordinador que reparte turnos, un Recepcionista que te manda a un puesto, o una Enfermera que te llama: esos toman decisiones, y ahí sí hacen falta procesos.

**Inventar un `Process Detector` que no tiene nada que hacer es el error de modelado más común de la práctica.**

### Recordatorio: el valor inicial dice el rol

| Init | Rol | Cómo se usa |
|---|---|---|
| `= 1` | **Mutex** | `P` y `V` los hace **el mismo** proceso |
| `= 0` | **Señalización** | El `V` y el `P` los hacen procesos **distintos** |
| `= n` | **Contador de recursos** | Cuenta unidades libres |

---

## 2. Ítem (a) — el análisis

**Procesos**

```
Process Persona[id: 1..N]        ← un solo TIPO, N instancias
```

Ojo con el vocabulario: `Process Persona[id: 1..N]` no declara *un* proceso, declara **N procesos** que ejecutan el mismo código y se distinguen sólo por el valor de `id`. La respuesta correcta al (a) es *"N procesos Persona, un solo tipo"*, no *"un proceso"*.

No hay proceso Detector (es pasivo), ni coordinador (el enunciado no pide orden).

**Recursos compartidos**

| Recurso | Unidades en (b) | Unidades en (c) |
|---|---|---|
| El detector de metales | **1** | **3** |

No hay variables compartidas: no hay nada que contar ni acumular.

**Sincronizaciones**

Una sola: **una persona no puede usar un detector ocupado**. Es competencia por unidades de un recurso.

**Semáforos**

```
sem detector = 1;       // (b)
sem detectores = 3;     // (c)
```

---

## 3. Ítem (b) — solución

```
sem detector = 1;

Process Persona[id: 1..N]
{ P(detector);
  delay(tiempo);        // pasa por el detector
  V(detector);
}
```

Detalles de notación que se corrigen: **`P` y `V` en mayúscula** (son los nombres de las operaciones, no variables), y **`delay` lleva un tiempo** como argumento — la descripción va en un comentario al lado.

### La decisión que importa: cómo lo leés

Con una sola unidad, `sem detector = 1` admite dos lecturas. **El código es idéntico**, pero una te ahorra trabajo en el (c):

| Lectura | Cómo lo pensás | Qué pasa en el (c) |
|---|---|---|
| **Mutex** | "un candado que protege el detector" | Te obliga a **repensar**: un candado no se generaliza a 3 |
| **Contador de recursos** | "hay 1 unidad disponible" | Cambiás el `1` por un `3` y **terminaste** |

Acá la correcta es la segunda: el detector **es un recurso con unidades**, no una sección crítica.

> **Regla:** si el enunciado dice *"hay K de algo"*, es un **contador de recursos** inicializado en K — aunque K valga 1.

---

## 4. Ítem (c) — tres detectores

```
sem detectores = 3;

Process Persona[id: 1..N]
{ P(detectores);
  delay(tiempo);
  V(detectores);
}
```

Un solo cambio: el valor inicial.

### La pregunta que hay que hacerse: ¿hace falta saber CUÁL?

El enunciado **no lo pide**, así que el contador alcanza. Pero vale entender qué cambiaría si lo pidiera, porque es la diferencia entre dos modelos:

| | `sem detectores = 3` | Cola de detectores |
|---|---|---|
| Qué controla | **cuántos** entran a la vez | **cuál** te tocó |
| Qué te da | un permiso | el objeto concreto |
| Alcanza si | las unidades son **anónimas** | tenés que **nombrar** la que usás |

Si hiciera falta la identidad:

```
cola detectores;                      // contiene los ids 1, 2, 3
sem disponibles = 3, mutex = 1;

Process Persona[id: 1..N]
{ int d;
  P(disponibles);                             // espero a que haya alguno libre
  P(mutex); pop(detectores, d); V(mutex);     // me entero de CUÁL me tocó
  delay(tiempo);                              // uso el detector d
  P(mutex); push(detectores, d); V(mutex);
  V(disponibles);
}
```

### Por qué acá conviven el contador Y la cola

Parece estado redundante —el semáforo cuenta lo mismo que el tamaño de la cola— pero no lo es, y la razón es una limitación real de los semáforos:

> Con `await` podías escribir `<await (not vacia(cola)); d = pop(cola)>`: la condición era **cualquier expresión** sobre la cola.
> Con semáforos **no podés esperar una condición arbitraria**: sólo podés hacer `P` sobre un semáforo.

Así que el `sem disponibles = 3` **no es un contador de más**: es el **único mecanismo de bloqueo** disponible. La cola te da la identidad, el semáforo te da la espera. Cada uno hace un trabajo distinto.

> **Con semáforos, la condición tiene que estar codificada en el valor de un semáforo.** Por eso aparecen semáforos que con `await` no hacían falta.

---

## 5. Ítem (d) — cantidad aleatoria de pasadas

```
Process Persona[id: 1..N]
{ int veces = random(1..K);        // LOCAL: cada persona tiene la suya
  for j = 1 to veces
     { P(detectores);
       delay(tiempo);
       V(detectores);
     }
}
```

**Dos cosas que se corrigen:**

**`veces` es local.** Cada persona sortea su propia cantidad. Si fuera global, todas pasarían el mismo número de veces y además habría carrera al escribirla.

**No es `while (true)`.** El enunciado dice "más de una vez, siendo **aleatoria** esa cantidad": aleatoria pero **finita**. Cada persona pasa unas cuantas veces y **termina**.

### El error que se penaliza

```
P(detectores);                    ← MAL
for j = 1 to veces
   { delay(tiempo); }
V(detectores);
```

Así una persona **monopoliza el detector** para todas sus pasadas seguidas, y las demás esperan a que termine su tanda entera. El enunciado quiere que **vuelva a competir en cada pasada**.

> **Criterio general:** la sección crítica lo más chica posible. El `P` y el `V` envuelven **una** pasada, no la tanda.

---

## 6. Chequeo de sanidad

- Con **N = 1** (una sola persona) nunca hay espera: el `P` pasa siempre de largo. La solución colapsa al caso trivial.
- En (c), con **3 personas o menos** ninguna espera; con la cuarta empieza la demora. Es lo esperable.
- Todo `P` tiene su `V` en el mismo proceso y en el mismo camino de ejecución. Si un camino hace `P` y no hace `V`, el recurso se pierde para siempre.

---

## 7. Los tres chequeos del corrector

1. **¿Modeló el detector como recurso y no como proceso?** Un `Process Detector` vacío es error de modelado.
2. **¿El `delay` está fuera de la sección crítica en el (d)?** Es decir, ¿el `P`/`V` envuelve una sola pasada? Monopolizar el recurso durante toda la tanda es el error del ítem.
3. **¿Están los semáforos declarados e inicializados?** Y en (c), ¿generalizó cambiando el valor inicial, o reescribió todo?
