# Parcial Práctico de MC — 1ra fecha — Tema 2 (rendido el 05/10/2026)

> Foto del enunciado: `../Parcial 1- 1ra fecha 2026 - Tema 2.jpg`. Igual que en 2022, fueron **dos ejercicios**: uno con semáforos y otro con monitores.

---

# Ejercicio 1 — SEMÁFOROS

> Existen **N bandas** que deben usar la sala de ensayo de una escuela de música. La sala sólo puede ser usada **por una banda a la vez**, dando **prioridad de acuerdo con la antigüedad** de la banda (cuando la sala está libre la debe usar la banda con más años de trayectoria entre las que están esperando). Implemente una solución con **SEMÁFOROS** que considere **únicamente procesos Banda**. *Nota: la función `obtenerAntiguedad(i)` retorna la antigüedad para la banda i.*

## Cómo se reconoce

| Frase | Molde |
|---|---|
| *"por una banda a la vez"* | Recurso escaso → alguien espera a que otro libere |
| *"únicamente procesos Banda"* | **Prohíbe el coordinador** → Passing the Baton |
| *"prioridad de acuerdo con la antigüedad"* | Cola ordenada (`insertarOrdenado`) |

**Passing the Baton con prioridad.**

## Solución

```
// PRECONDICIONES: N bandas; la sala arranca libre y la cola vacía

sem mutex = 1;
sem espera[N] = ([N] 0);       // el V lo hace OTRA banda al liberar → arranca en 0
bool libre = true;
colaOrdenada fila;             // ordenada por antigüedad, mayor primero

Process Banda[id: 0..N-1] {
  int ant = obtenerAntiguedad(id);   // afuera del mutex: no toca nada compartido
  int sig;

  // --- pedir la sala ---
  P(mutex);
  if (libre) {
    libre = false;
    V(mutex);
  } else {
    fila.insertarOrdenado(id, ant);  // me encolo por antigüedad
    V(mutex);                        // suelto el mutex ANTES de dormirme
    P(espera[id]);                   // espero que me la ENTREGUEN
  }

  UsarSala();

  // --- liberar la sala ---
  P(mutex);
  if (fila.esVacia())
    libre = true;                    // nadie espera: queda libre
  else {
    fila.pop(sig);                   // la de más antigüedad
    V(espera[sig]);                  // se la entrego directo (libre sigue en false)
  }
  V(mutex);
}
```

## Lo que se corrige

1. `espera[N]` arranca en 0, uno por banda, indexado por `[id]`.
2. `V(mutex)` **antes** de `P(espera[id])`. Si no, hay deadlock.
3. **El recurso se entrega, no se libera:** si hay alguien en la fila, `libre` queda en `false`. Si no, otra banda que llega se cuela y saltea la prioridad.
4. La prioridad vive en la cola ordenada, no en el semáforo.
5. `obtenerAntiguedad` va fuera del mutex.

## Lo que hice ✅

Lo mismo, con `insertarEnColaOrdenada(cola, id, a)` y la antigüedad calculada antes del push. **Bien.**

---

# Ejercicio 2 — MONITORES

> Una aplicación de viajes compartidos junta pasajeros **de a 4** antes de salir. Existen **P pasajeros (P múltiplo de 4)**. Cada pasajero se anota en la aplicación, y **en cuanto se completan 4 anotados**, entonces forman un grupo y suben al auto para salir de viaje (el viaje dura un **tiempo fijo de 30 min**). Al terminar el viaje, los 4 pasajeros del grupo se bajan. Implemente una solución usando **MONITORES**. *Nota: maximizar la concurrencia.*

## Cómo se reconoce

| Frase | Molde |
|---|---|
| *"junta pasajeros de a 4"*, *"en cuanto se completan 4"* | **Formación de grupos = barrera por grupo** |
| *"maximizar la concurrencia"* | El `delay` del viaje va **fuera** del monitor |

> **Coordinador o barrera: la pregunta que los distingue.**
> **¿Alguien atiende o le entrega algo a otro?** Si sí (un empleado, una estación, un dato que va y vuelve), es **coordinador**. Si no, y se esperan entre ellos ("cuando se completan 4", "cuando llegan todos"), es **barrera**.
> Acá nadie atiende a nadie y no viaja ningún dato. La "aplicación" es el monitor mismo, no un proceso.

## Solución

```
// PRECONDICIONES: P pasajeros, P múltiplo de 4; autos ilimitados (el enunciado no los limita)

Monitor App {
  int cant = 0;
  cond grupo;

  Procedure anotarse() {
    cant++;
    if (cant < 4)
      wait(grupo);           // espero a completar el grupo
    else {
      signal_all(grupo);     // el 4º despierta a los otros 3
      cant = 0;              // reset para el próximo grupo
    }
  }
}

Process Pasajero[id: 1..P] {
  App.anotarse();
  delay(30 min);             // el viaje, FUERA del monitor
  // se baja
}
```

## Por qué funciona

- **Hay varios grupos viajando a la vez.** El grupo 1 suelta el monitor antes de viajar y el contador ya está en 0, así que el grupo 2 se forma mientras el 1 viaja.
- **Alcanza con `if` y no hace falta `while`.** Con `signal_all`, los 3 despertados salen de la cola de `grupo` en ese momento. Si llega un 5º antes de que vuelvan a entrar al monitor, ese 5º arranca un grupo nuevo (`cant = 1`) y no se mezcla con ellos.
- **No hace falta una barrera para bajar:** los 4 arrancan juntos y el viaje dura 30 minutos fijos para todos.
- **Si el enunciado dijera "hay A autos"**, el auto pasaría a ser un recurso escaso y habría que agregar un segundo molde: el grupo espera un auto libre.

## Lo que hice ❌

Lo planteé como **coordinador**. Sabía que no cerraba, pero no recordé el molde de grupos. El enunciado no prohíbe procesos extra, así que un coordinador puede sumar puntos si:

1. usa monitores (no semáforos),
2. espera a tener 4 y los libera juntos,
3. deja el `delay` fuera del monitor,
4. no tiene deadlock ni señales perdidas.

**Lección para el recuperatorio:** la falla fue de reconocimiento, no de contenido. Era el molde de las 10 canchas de paddle del gimnasio. Lo que hay que entrenar es la distinción **coordinador o barrera**.
