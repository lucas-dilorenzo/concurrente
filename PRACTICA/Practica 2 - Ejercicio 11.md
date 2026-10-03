# Práctica 2 — Ejercicio 11

> **Enunciado.** En un vacunatorio hay **un empleado de salud** para vacunar a **50 personas**. El empleado atiende a las personas **de acuerdo con el orden de llegada** y **de a 5 personas a la vez**. Es decir, que cuando está libre debe **esperar a que haya al menos 5 personas esperando**, luego **vacuna a las 5 primeras**, y al terminar **las deja ir** para esperar por otras 5. Cuando ha atendido a las 50 personas **el empleado se retira**.
> *Nota: todos los procesos deben terminar su ejecución; suponga que el empleado tiene una función `VacunarPersona()` que simula que el empleado está vacunando a UNA persona.*

---

## 0. Lo que define el ejercicio

Tres frases, tres decisiones:

| Frase | Qué implica |
|---|---|
| *"de acuerdo con el **orden de llegada**"* | Cola + semáforos privados. Los semáforos no son FIFO |
| *"esperar a que haya **al menos 5** personas esperando"* | El empleado no atiende de a uno: espera a juntar 5 |
| *"cuando ha atendido a las 50 **se retira**"* + *"todos los procesos deben terminar"* | Nada de `while(true)`. **50 / 5 = 10 tandas exactas** |

**Procesos:** `Persona[1..50]` y `Empleado` (uno). El empleado **decide** a quién llama ⇒ es un proceso, no un semáforo.

---

## 1. Cómo se espera "al menos 5"

Acá está el truco del ejercicio, y es más simple de lo que parece.

No hace falta contar con una variable ni consultar nada: **cinco `P` seguidos sobre el mismo semáforo**.

```
for k = 1 to 5  P(hayPersona);      // espero a que hayan llegado 5
```

Cada persona hace **un** `V(hayPersona)` al anotarse. Entonces cinco `P` exitosos significan, por definición, que **llegaron al menos cinco**. El semáforo lleva la cuenta solo — para eso guarda memoria.

> **En criollo:** *si cada uno toca el timbre una vez, esperar cinco timbrazos es esperar cinco personas.*

Compará con un semáforo binario: ahí tendrías que llevar un contador aparte y un mutex. Con un contador de recursos, la cuenta ya está adentro.

---

## 2. Solución

```
// PRECONDICIONES: 50 personas, atención de a 5 ⇒ 10 tandas exactas (50 mod 5 == 0)

cola espera;                       // ids, en orden de llegada
sem mutexCola   = 1;               // 50 escritores
sem hayPersona  = 0;               // señal: "llegó una persona más"
sem puedeIrse[50] = ([50] 0);      // señal: "ya te vacuné, andate"


Process Persona [id: 1..50]
{ P(mutexCola);
    push(espera, id);              // me anoto en la fila
  V(mutexCola);
  V(hayPersona);                   // toco el timbre

  P(puedeIrse[id]);                // espero a que me vacunen y me dejen ir
}                                  // me retiro. Sin while(true)


Process Empleado
{ int atendidos[1..5];             // LOCAL: los 5 de esta tanda
  int tanda, k;

  for tanda = 1 to 10 {            // 50 / 5 = 10 tandas, y me retiro

     for k = 1 to 5
        P(hayPersona);             // espero a que haya AL MENOS 5 esperando

     P(mutexCola);
       for k = 1 to 5
          pop(espera, atendidos[k]);    // los 5 PRIMEROS, en orden de llegada
     V(mutexCola);

     for k = 1 to 5
        VacunarPersona();          // vacuno de a una, AFUERA del candado

     for k = 1 to 5
        V(puedeIrse[atendidos[k]]);     // los dejo ir a los 5
  }
}
```

### Por qué cada pieza está donde está

| Decisión | Por qué |
|---|---|
| Los 5 `pop` en **una sola** toma del candado | Garantiza que son los 5 primeros consecutivos de la fila. Si soltaras el mutex en el medio, alguien podría anotarse entre medio (no rompe el orden, pero es más difícil de defender) |
| `VacunarPersona()` **afuera** del candado | Es la operación larga. Adentro, la fila no avanza mientras vacuna |
| `atendidos[]` **local** del empleado | Es su bloc de notas de la tanda. Nadie más lo toca |
| `puedeIrse[50]`, no un semáforo global | Hay que despertar a **esas 5** y no a cinco cualesquiera. Los semáforos no son FIFO |
| Se los deja ir **después** de las 5 vacunaciones | El enunciado dice *"vacuna a las 5 primeras, y **al terminar** las deja ir"* |

> **Ojo con el último punto:** es tentador hacer `VacunarPersona(); V(puedeIrse[k]);` de a uno, dejando ir a cada uno apenas lo vacunás. **El enunciado no dice eso**: dice que las deja ir al terminar con las cinco. Es un grupo que entra y sale junto.

---

## 3. Justificación (esto es lo que suma puntos)

> El orden de llegada debe registrarse explícitamente porque no puede suponerse que los semáforos sean FIFO: se usa una cola protegida por `mutexCola`, ya que las 50 personas escriben en ella. Cada persona se anota, señaliza con `V(hayPersona)` y se demora en su semáforo privado `puedeIrse[id]`, inicializado en 0 por ser una señal cuyo `V` lo ejecuta otro proceso; hace falta uno por persona porque el empleado debe liberar exactamente a las cinco que atendió y no a cinco cualesquiera. La condición *"al menos 5 esperando"* se resuelve con cinco `P(hayPersona)` consecutivos: como cada persona emite un único `V`, cinco `P` exitosos implican que llegaron al menos cinco, sin necesidad de contadores auxiliares. El empleado extrae las cinco primeras en una única sección crítica, las vacuna una por una fuera del candado —para no bloquear la fila durante la atención— y recién al terminar con las cinco las libera, tal como indica el enunciado. Como son 50 personas atendidas de a 5, el empleado ejecuta exactamente 10 tandas y se retira; todos los procesos terminan.

---

## 4. Los errores que se penalizan

**(1) `while(true)` en el empleado.** El enunciado dice explícitamente que se retira, y la Nota exige que **todos** los procesos terminen. Son 10 tandas.

**(2) Un semáforo global en vez de `puedeIrse[50]`.** Con `V(listo)` ×5 sobre un semáforo compartido se despiertan **cinco cualesquiera** de los que estén esperando — no necesariamente los cinco vacunados. Se rompe el orden de llegada y hay gente que se va sin vacunar.

**(3) Atender de a uno.** `P(hayPersona); pop; VacunarPersona(); V(puedeIrse)` en un loop de 50. Funciona mecánicamente pero **no modela el enunciado**: no espera a juntar 5 ni los deja ir como grupo.

**(4) Dejarlos ir de a uno, a medida que los vacuna.** El enunciado dice *"al terminar las deja ir"*: los cinco `V` van después de las cinco vacunaciones.

**(5) `VacunarPersona()` adentro del candado de la cola.** Nadie puede anotarse durante toda la tanda.

**(6) Llevar un contador de "cuántos esperan" con su mutex.** No está mal, pero es trabajo de más: el semáforo contador ya lleva esa cuenta.

**(7) La persona durmiéndose con `mutexCola` tomado.** Deadlock instantáneo: nadie más puede anotarse y el empleado se cuelga en `P(mutexCola)`.

---

## 5. Chequeos de sanidad

- **Contá:** `hayPersona` recibe 50 `V` (uno por persona) y 50 `P` (5 × 10 tandas). Cierra.
- Cada `puedeIrse[i]` recibe **exactamente un** `V` y un `P`.
- `VacunarPersona()` se ejecuta **50 veces** en total.
- **La cola queda vacía** al final: 50 `push` y 50 `pop`.
- **Todos terminan:** las 50 personas son liberadas y el empleado sale del `for`.
- **50 mod 5 == 0** — por eso las tandas son exactas. Si el enunciado dijera 52 personas, habría que tratar la última tanda incompleta, y conviene escribirlo como precondición.

---

## 6. Los tres chequeos del corrector

1. **¿El empleado termina?** 10 tandas, nada de `while(true)`. La Nota lo pide explícitamente.
2. **¿Los semáforos de liberación son un arreglo?** Hay que dejar ir **a esos cinco**, no a cinco cualesquiera.
3. **¿Se juntan 5 antes de empezar y se los libera recién al terminar los 5?** Es lo que distingue este ejercicio de una atención de a uno.
