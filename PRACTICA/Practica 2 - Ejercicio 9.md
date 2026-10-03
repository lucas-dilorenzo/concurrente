# Práctica 2 — Ejercicio 9

> **Enunciado.** Resolver el funcionamiento en una **fábrica de ventanas** con **7 empleados** (4 carpinteros, 1 vidriero y 2 armadores) que trabajan de la siguiente manera:
> • Los **carpinteros** continuamente hacen marcos (cada marco es armado por un único carpintero) y los dejan en un depósito con capacidad para **30 marcos**.
> • El **vidriero** continuamente hace vidrios y los deja en otro depósito con capacidad para **50 vidrios**.
> • Los **armadores** continuamente toman **un marco y un vidrio (en ese orden)** de los depósitos correspondientes y arman la ventana (cada ventana es armada por un único armador).

---

## 0. Qué patrón es (y cuál NO)

Buscá en el enunciado las palabras *"orden de llegada"*, *"el primero que"*, *"por identificador"*. **No están.** A nadie le importa qué carpintero deposita primero ni qué armador saca primero.

> Esto es **buffer acotado, dos veces**. Las palabras que lo delatan: *depósito · capacidad para 30 · capacidad para 50 · los deja en · toman de*.

**Regla de conteo: tantos pares de semáforos cruzados como depósitos haya.** Acá son dos, y son **completamente independientes** entre sí.

```
// Depósito de marcos            // Depósito de vidrios
sem lugarMarcos  = 30;           sem lugarVidrios  = 50;
sem marcosListos = 0;            sem vidriosListos = 0;
```

> **Automatizá esto:** apenas leés *"depósito con capacidad para K"*, escribís **dos** semáforos — `lugar = K` y `listos = 0`. Nunca uno solo. Uno frena al que produce cuando está lleno; el otro frena al que consume cuando está vacío. Son dos esperas distintas.

---

## 1. Hacer el marco NO se sincroniza con nada

El error más caro del ejercicio es creer que *"cada marco es armado por un único carpintero"* significa *"un carpintero a la vez"*.

| Lo que dice | Lo que NO dice |
|---|---|
| Un carpintero por marco: no se juntan dos a hacer el mismo | Un carpintero a la vez |

Si pusieras un `sem carpintero = 1` alrededor del trabajo, **los 4 carpinteros rendirían como 1**: mientras uno martilla, los otros tres miran. ¿Para qué contratar cuatro?

> **`HacerMarco()` es trabajo propio**, como el `delay` de cualquier proceso que produce lo suyo. Lo único que se sincroniza es **dejarlo en el depósito**.

Lo mismo con `ArmarVentana()`: va afuera de todo candado.

---

## 2. Los índices: tres mutex, no cuatro

La pregunta no es si el arreglo es compartido, sino **cuántos procesos escriben cada índice**:

| Depósito | Índice de **carga** | Índice de **descarga** |
|---|---|---|
| **Marcos** (30) | 4 carpinteros → **mutex** | 2 armadores → **mutex** |
| **Vidrios** (50) | **1 vidriero → SIN mutex** (único dueño, puede ser local) | 2 armadores → **mutex** |

El vidriero es el único que carga su depósito, así que su índice es una **variable local** suya. No hay con quién competir.

### Y la escritura va ADENTRO del mutex, no sólo el índice

Donde hay varios del mismo rol, **tocar la celda y avanzar el índice tienen que ser un solo acto**:

```
P(mutexPonerMarco);
  depMarcos[posPonerMarco] = m;                   // la escritura, ADENTRO
  posPonerMarco = (posPonerMarco + 1) mod 30;
V(mutexPonerMarco);
V(marcosListos);
```

Si sólo reservaras el índice adentro y escribieras afuera, un carpintero podría hacer `V(marcosListos)` mientras otro todavía no escribió su celda anterior — y el armador, que recorre en orden circular, se llevaría un marco inexistente.

---

## 3. El *"(en ese orden)"*

No es decorativo: **los dos armadores tienen que pedir en el mismo orden**, primero marco y después vidrio.

Si uno pidiera vidrio→marco y el otro marco→vidrio, cada uno podría quedarse con la mitad de lo que necesita el otro: espera circular, el caso clásico de deadlock.

> **La regla general:** cuando un proceso necesita **dos recursos distintos**, todos los procesos que los pidan deben hacerlo **en el mismo orden**. Es la forma más barata de eliminar el deadlock.

---

## 4. Solución

```
// PRECONDICIONES: ambos depósitos arrancan vacíos

marco  depMarcos[30];
vidrio depVidrios[50];

// ── DEPÓSITO DE MARCOS ──
sem lugarMarcos  = 30;                      // lugares libres   → lo espera el CARPINTERO
sem marcosListos = 0;                       // marcos guardados → lo espera el ARMADOR
int posPonerMarco = 0;   sem mutexPonerMarco = 1;      // 4 carpinteros
int posSacarMarco = 0;   sem mutexSacarMarco = 1;      // 2 armadores

// ── DEPÓSITO DE VIDRIOS ──
sem lugarVidrios  = 50;                     // → lo espera el VIDRIERO
sem vidriosListos = 0;                      // → lo espera el ARMADOR
int posSacarVidrio = 0;  sem mutexSacarVidrio = 1;     // 2 armadores
                                            // el índice de carga es LOCAL del vidriero


Process Carpintero [id: 1..4]
{ marco m;                                  // LOCAL
  while (true) {
     m = HacerMarco();                      // trabajo propio: SIN sincronización
     P(lugarMarcos);                        // espero lugar en el depósito
     P(mutexPonerMarco);
       depMarcos[posPonerMarco] = m;        // escritura ADENTRO del candado
       posPonerMarco = (posPonerMarco + 1) mod 30;
     V(mutexPonerMarco);
     V(marcosListos);                       // aviso: hay un marco más
  }
}

Process Vidriero
{ vidrio v;                                 // LOCAL
  int pos = 0;                              // LOCAL: soy el único que carga
  while (true) {
     v = HacerVidrio();                     // trabajo propio
     P(lugarVidrios);
     depVidrios[pos] = v;                   // sin candado: único escritor
     pos = (pos + 1) mod 50;
     V(vidriosListos);
  }
}

Process Armador [id: 1..2]
{ marco m;  vidrio v;                       // LOCALES
  while (true) {
     // ── PRIMERO EL MARCO ──
     P(marcosListos);
     P(mutexSacarMarco);
       m = depMarcos[posSacarMarco];
       posSacarMarco = (posSacarMarco + 1) mod 30;
     V(mutexSacarMarco);
     V(lugarMarcos);                        // liberé un lugar del depósito

     // ── DESPUÉS EL VIDRIO ──
     P(vidriosListos);
     P(mutexSacarVidrio);
       v = depVidrios[posSacarVidrio];
       posSacarVidrio = (posSacarVidrio + 1) mod 50;
     V(mutexSacarVidrio);
     V(lugarVidrios);

     ArmarVentana(m, v);                    // AFUERA de todo candado
  }
}
```

### Por qué cada pieza está donde está

| Decisión | Por qué |
|---|---|
| `HacerMarco()` / `HacerVidrio()` **antes** del `P(lugar…)` | La pieza se arma en la mano, no dentro del depósito. Reservar lugar antes desperdicia capacidad |
| `V(…Listos)` **después** de escribir | Si avisás antes, el armador se lleva una celda vacía |
| `V(lugar…)` **apenas** sacaste | El lugar queda libre en ese momento. Retenerlo durante `ArmarVentana()` inmoviliza capacidad |
| `ArmarVentana()` **afuera** | Es la operación larga y no necesita ningún recurso compartido. Adentro, los 2 armadores rinden como 1 |
| `P(marcosListos)` **antes** que `P(vidriosListos)`, en los dos armadores | Orden consistente ⇒ sin espera circular |

---

## 5. Justificación (esto es lo que suma puntos)

> Hay dos depósitos independientes, así que se usan **dos pares de semáforos cruzados**: `lugarMarcos = 30` / `marcosListos = 0` y `lugarVidrios = 50` / `vidriosListos = 0`. Cada productor hace `P` sobre los lugares libres y `V` sobre las piezas guardadas, y cada consumidor al revés; en reposo vale `lugar + listos = capacidad` en cada depósito. La fabricación de marcos y vidrios no requiere sincronización alguna —cada empleado produce su propia pieza— y la frase *"cada marco es armado por un único carpintero"* significa un carpintero por marco, no un carpintero por vez: serializar el trabajo convertiría a los cuatro en uno. Los índices llevan candado sólo donde hay más de un proceso del mismo rol: el de carga del depósito de marcos (4 carpinteros) y los dos de descarga (2 armadores); el índice de carga de los vidrios es local del vidriero, que es su único escritor. Donde hay candado, la escritura o la lectura de la celda va **dentro** de la sección crítica junto con el avance del índice, para no romper el orden circular que supone el otro extremo. `ArmarVentana` queda fuera de todo candado y los lugares se liberan apenas se retira la pieza, de modo que los dos armadores trabajen realmente en paralelo. Finalmente, ambos armadores piden **marco y después vidrio**, en ese orden: si lo hicieran en órdenes distintos, cada uno podría retener la mitad de lo que necesita el otro y se produciría un deadlock por espera circular.

---

## 6. Los errores que se penalizan

**(1) Un solo semáforo por depósito.** Con `sem marcos = 30` únicamente, sabés frenar al carpintero cuando está lleno, pero **no tenés con qué frenar al armador** cuando está vacío. Dos esperas distintas, dos semáforos.

**(2) Serializar a los carpinteros.** `sem carpintero = 1` alrededor de `HacerMarco()`. Los 4 rinden como 1. Es el error central del ejercicio.

**(3) `ArmarVentana()` adentro de un candado.** No corrompe nada, pero anula el paralelismo de los dos armadores.

**(4) Mezclar los dos depósitos en un solo par de semáforos.** Son recursos independientes con capacidades distintas: cada uno lleva el suyo.

**(5) Mutex en el índice de carga de los vidrios.** No hace falta: el vidriero es único escritor. No rompe nada, pero muestra que no contaste escritores.

**(6) Reservar el índice adentro del candado y escribir afuera.** Rompe el orden circular que supone el otro extremo: se avisa de una pieza que todavía no está.

**(7) Que los armadores pidan en órdenes distintos.** Deadlock por espera circular. El enunciado lo previene con el *"(en ese orden)"*.

**(8) `V(lugarMarcos)` después de `ArmarVentana()`.** El lugar queda inmovilizado durante toda la ventana. Correcto pero ineficiente.

---

## 7. Chequeos de sanidad

- **Contá por depósito:** piezas guardadas + lugares libres = la capacidad (30 y 50). `lugar + listos` coincide siempre.
- **Los `P` y `V` cruzados cierran en procesos distintos:** `P(lugarMarcos)` lo hace el carpintero y `V(lugarMarcos)` el armador. Si los ves de a pares en el mismo proceso, está mal modelado.
- **Con 1 carpintero y 1 armador**, los mutex de marcos sobran y la solución colapsa al buffer acotado simple. ✔
- **Sin deadlock:** ningún proceso se bloquea reteniendo un candado, y los dos armadores piden en el mismo orden.
- **`while (true)` en los tres procesos:** el enunciado dice *"continuamente"* y nunca da un total. No hay terminación.

---

## 8. Los tres chequeos del corrector

1. **¿Hay DOS pares de semáforos cruzados, uno por depósito, con las capacidades correctas (30 y 50)?**
2. **¿`HacerMarco()`, `HacerVidrio()` y `ArmarVentana()` están fuera de todo candado?** Adentro, los 4 carpinteros o los 2 armadores rinden como uno.
3. **¿Los candados están sólo donde hay varios del mismo rol, y los dos armadores piden en el mismo orden?**
