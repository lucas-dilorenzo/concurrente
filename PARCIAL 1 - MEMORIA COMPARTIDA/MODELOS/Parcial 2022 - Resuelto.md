# Parcial Práctico de MC — 1ra fecha — 11/10/2022

> **Formato del parcial: DOS ejercicios.** Uno con semáforos, uno con monitores. La resolución original del alumno **no tiene ni un párrafo de justificación**: sólo código con comentarios cortos al margen.

---

# Ejercicio 1 — SEMÁFOROS

> En una **planta verificadora de vehículos** existen **7 estaciones** donde se dirigen **150 vehículos** para ser verificados. Cuando un vehículo llega a la planta, el **coordinador** le indica a qué estación debe dirigirse: selecciona **la estación que tenga menos vehículos asignados** en ese momento. Una vez que el vehículo sabe qué estación le fue asignada, se dirige a la misma y **espera a que lo llamen** para verificar. Luego de la revisión, la estación le entrega un **comprobante** que indica si pasó o no. Más allá del resultado, el vehículo se retira. *Nota: maximizar la concurrencia.*

## El análisis en 30 segundos

**Tres procesos.** Aplicá el test *¿decide algo o sólo se usa?*: el **coordinador** *"le indica a qué estación"* → decide. La **estación** *"lo llama"* y *"le entrega un comprobante"* → decide. Los puestos en sí serían recursos, pero acá **actúan**.

**Es el molde del coordinador, dos veces encadenadas.** No es Passing the Baton: hay coordinador y hay estaciones, o sea procesos que atienden.

```
Vehículo ──► COORDINADOR   → le devuelve QUÉ ESTACIÓN
Vehículo ──► ESTACIÓN[e]   → lo llama, verifica, le devuelve EL COMPROBANTE
```

## Solución

```
// PRECONDICIONES: 150 vehículos, 7 estaciones; todas las colas arrancan vacías

// ── Canal 1: la recepción ──
cola recepcion;                        sem mutexRecepcion = 1;    // 150 escritores
sem hayVehiculo = 0;                   // Vehículo → Coordinador
sem asignado[150] = ([150] 0);         // Coordinador → Vehículo
int estacionDe[150];                   // el DATO: qué estación le tocó

// ── El reparto ──
int asignados[1..7] = ([7] 0);         // cuántos tiene cada estación
sem mutexAsignados = 1;                // lo tocan el coordinador y las 7 estaciones

// ── Canal 2: las estaciones (todo indexado por estación) ──
cola colaEstacion[1..7];               sem mutexColaEst[1..7] = ([7] 1);
sem hayEnEstacion[1..7] = ([7] 0);     // Vehículo → Estación
sem verificado[150] = ([150] 0);       // Estación → Vehículo
comprobante compDe[150];               // el DATO: su comprobante


Process Vehiculo [id: 1..150]
{ int mi_estacion;  comprobante mi_comp;

  // (1) me presento al coordinador
  P(mutexRecepcion);  push(recepcion, id);  V(mutexRecepcion);
  V(hayVehiculo);

  // (2) espero que me asigne una estación
  P(asignado[id]);
  mi_estacion = estacionDe[id];

  // (3) me anoto en la cola de ESA estación
  P(mutexColaEst[mi_estacion]);
    push(colaEstacion[mi_estacion], id);
  V(mutexColaEst[mi_estacion]);
  V(hayEnEstacion[mi_estacion]);

  // (4) espero que me llamen, me verifiquen y me den el comprobante
  P(verificado[id]);
  mi_comp = compDe[id];
}                                      // me retiro. Sin while(true)


Process Coordinador
{ int i, v, menor;
  for i = 1 to 150 {                   // total conocido → termina
     P(hayVehiculo);
     P(mutexRecepcion);  pop(recepcion, v);  V(mutexRecepcion);

     P(mutexAsignados);
       menor = índice de 1..7 con menor asignados[];
       asignados[menor]++;             // elegir e incrementar: UN SOLO acto
     V(mutexAsignados);

     estacionDe[v] = menor;            // el DATO primero
     V(asignado[v]);                   // la SEÑAL después
  }
}


Process Estacion [e: 1..7]
{ int v;
  while (true) {                       // reparto dinámico: no sabe cuántos le tocan
     P(hayEnEstacion[e]);
     P(mutexColaEst[e]);  pop(colaEstacion[e], v);  V(mutexColaEst[e]);

     compDe[v] = Verificar();          // AFUERA de todo candado
     V(verificado[v]);                 // le entrego el comprobante

     P(mutexAsignados);  asignados[e]--;  V(mutexAsignados);   // ya no lo tengo asignado
  }
}
```

## Las cuatro decisiones que hay que saber defender

1. **Elegir el mínimo e incrementarlo van en la MISMA sección crítica.** Separados, dos vehículos consecutivos ven el mismo mínimo y van a la misma estación: se rompe el balanceo.
2. **`Verificar()` afuera de todo candado.** Adentro, las 7 estaciones trabajan de a una.
3. **Las estaciones van con `while(true)`.** Ninguna sabe cuántos vehículos le van a tocar — el reparto es dinámico. Un `for i = 1 to 150` por estación daría 1050 atenciones para 150 vehículos.
4. **Tres colas de estación, no una global.** El orden es *"de llegada a la estación"*, no a la planta.

---

# Ejercicio 2 — MONITORES

> En un sistema operativo se ejecutan **20 procesos** que periódicamente realizan cierto cómputo mediante `Procesar()`. **Los resultados son persistidos en un archivo**, para lo que se requiere acceso al **subsistema de E/S**. Sólo un proceso a la vez puede usarlo, y el acceso se define por la **prioridad del proceso** (**menor valor indica mayor prioridad**).

## El análisis en 10 segundos

Es el **molde de Passing the Condition con prioridad**: el recurso se usa de a uno, el orden es por prioridad. Cambia **una línea** respecto del molde por orden de llegada.

## Solución

```
monitor SO {
   bool libre = true;
   cola espera;                         // pares {id, prioridad}, ordenada
   cond esperaTurno[20];                // arreglo: hay que despertar a ESE

   procedure pedirAcceso (int id; int prioridad)
   { if (libre) libre = false;                          // libre: me lo quedo
     else { insertarOrdenado(espera, id, prioridad);    // ASCENDENTE:
                                                        // menor valor = mayor prioridad
            wait(esperaTurno[id]); } }                  // IF, no while

   procedure liberarAcceso ()                           // SIN parámetros
   { int sig;
     if (empty(espera)) libre = true;
     else { pop(espera, sig);
            signal(esperaTurno[sig]); } }               // libre SIGUE en false
}


Process Proceso [id: 1..20]
{ int prioridad;  resultado r;
  prioridad = ...;                      // cada proceso conoce la suya

  r = Procesar();                       // AFUERA: el cómputo NO necesita el subsistema

  SO.pedirAcceso(id, prioridad);
  guardarEnArchivo(r);                  // ESTO es lo que necesita el subsistema
  SO.liberarAcceso();
}
```

## Las tres decisiones

1. **`Procesar()` va ANTES de pedir el acceso.** El cómputo no necesita el subsistema de E/S; sólo la escritura del archivo. Adentro de la sección crítica, los 20 procesos computan de a uno — y la práctica pide *maximizar la concurrencia*.
2. **`insertarOrdenado` ASCENDENTE.** El enunciado dice *"menor valor indica mayor prioridad"*, al revés de lo intuitivo. **Aclaralo en un comentario.**
3. **`if` y no `while`**, y `libre` que no vuelve a `true` al traspasar. Passing the Condition.

---

# 📝 Lo que erré en el simulacro (3/10/2026)

| Ej | Error | Regla |
|---|---|---|
| **2** | `Procesar()` **adentro** de la sección crítica | La operación larga va afuera |
| **2** | No aclaré el sentido del orden | *"menor valor = mayor prioridad"* ⇒ ascendente |
| **2** | `int prioridad;` declarada y nunca asignada | — |
| **1** | `estacion[N]`, `resultado[N]`, `atendido2[N]` → iban con **`[id]`** | `N` es el tamaño, no quién sos |
| **1** | `int resultado` local **y** `int resultado[N]` compartido | **Nombres duplicados** — el error recurrente |
| **1** | `for i = 1 to 150` en **cada** estación | Reparto dinámico ⇒ `while(true)` |
| **1** | Faltaba `asignados[7]` + su mutex | Sin eso, *"la que tiene menos"* no está implementado |

**Lo que sí salió bien:** los tres procesos identificados correctamente · el molde del coordinador aplicado dos veces · el dato antes de la señal · el monitor del ejercicio 2 **perfecto** · usar `j` y no `id` para el vehículo dentro de `Estacion`.

**Tiempos:** monitores **10 min** · semáforos **~80 min** (40 de ellos trabado antes de reconocer el patrón).

> **La lección del simulacro no fue de contenido: fue de reconocimiento.** Una vez identificado *"es el molde del coordinador ×2"*, el ejercicio salió en 40 minutos. Los primeros 40 se fueron en no verlo.
