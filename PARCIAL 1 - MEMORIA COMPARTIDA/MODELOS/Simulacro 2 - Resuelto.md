# Simulacro 2 — Resuelto (4/10/2026)

> Dos ejercicios estilo parcial, tomados de los enunciados viejos. **Los dos son el molde del coordinador + un agregado.**

---

# Ej. 1 — SEMÁFOROS: el micro

> Una empresa de turismo posee **un micro con capacidad para 50 personas**. Hay **un único vendedor** que atiende a los **C clientes (C > 50)** **de acuerdo al orden de llegada** (la atención tarda un par de minutos). Si **aún hay lugares**, le indica **el número de asiento**. Cada cliente, luego de ser atendido, **se dirige al micro para subir** en caso de que le hayan dado asiento. **El micro espera a que los 50 pasajeros hayan subido** para realizar el viaje.

## Las flechas, antes de escribir

```
Cliente  ──── "llegué" ────►  Vendedor
Cliente  ◄── "tu asiento" ──  Vendedor
Cliente  ──── "subí" ──────►  Micro        (barrera de 50)
```

**Tres flechas ⇒ tres señales, todas en 0.** Más el candado de la cola, en 1.

## Solución

```
// PRE: C > 50

cola colaCli;                 sem mutexCola = 1;        // ← candado: P y V en el mismo proceso
sem hayCliente = 0;           // Cliente → Vendedor
sem atendido[C] = ([C] 0);    // Vendedor → Cliente
int asientoDe[C];             // el DATO (0 = no le tocó asiento)
int libres = 50;              // sólo lo toca el vendedor ⇒ sin candado

int subieron = 0;  sem mutexSub = 1;  sem microListo = 0;    // la barrera del micro

Process Cliente [id: 1..C]
{ int miAsiento;
  P(mutexCola);  push(colaCli, id);  V(mutexCola);
  V(hayCliente);                                 // AVISO yo

  P(atendido[id]);                               // ESPERO yo
  miAsiento = asientoDe[id];

  if (miAsiento > 0) {                           // sólo suben los que tienen asiento
     P(mutexSub);  subieron++;  V(mutexSub);
     V(microListo);
  }
}                                                // los que no tienen asiento, se van

Process Vendedor
{ int c;
  for i = 1 to C {                               // atiende a los C y se retira
     P(hayCliente);
     P(mutexCola);  pop(colaCli, c);  V(mutexCola);
     delay(atencion);                            // "tarda un par de minutos"

     if (libres > 0) { asientoDe[c] = 51 - libres;  libres--; }
     else             asientoDe[c] = 0;          // ya no hay lugar

     V(atendido[c]);
  }
}

Process Micro
{ for i = 1 to 50  P(microListo);                // espero a los 50
  Viajar();
}
```

## Las tres decisiones

1. **El `else` NO recarga el micro.** Hay un solo viaje: del cliente 51 en adelante, `asientoDe[c] = 0` y no sube.
2. **`libres` no lleva candado:** el único que lo toca es el vendedor.
3. **La barrera va en el Micro, no en los clientes.** Los 50 que suben hacen `V(microListo)` y el micro hace 50 `P`.

---

# Ej. 2 — MONITORES: la guardia

> En la **guardia de traumatología** trabajan **5 médicos y una enfermera**. Acuden **P pacientes** que al llegar se dirigen a la **enfermera**, para que les indique **a qué médico dirigirse** y **cuál es su gravedad** (1 a 10). Con esos datos van al médico y **esperan hasta que los termine de atender**. **Cada médico atiende en orden de gravedad.**

## La clave que define el ejercicio

> *"la enfermera les indica **a qué médico** deben dirigirse"* ⇒ **la enfermera REPARTE**. Cada médico tiene **su propia cola**, y el paciente tiene que **enterarse de cuál le tocó**.

Con una sola cola para los 5 médicos se pierde el reparto. Es la misma forma que la terminal de micros: **coordinador que asigna + un servidor por puesto** — con la **gravedad** como criterio de orden en vez de la llegada.

## Solución

```
monitor Guardia {
   cola recepcion;                    cond hayPaciente;          // ← espera la enfermera
   int medicoDe[P];  int gravedadDe[P];
   bool asignado[P] = ([P] false);    cond yaAsignado[P];        // ← espera el paciente

   cola porMedico[1..5];              cond hayEnMedico[1..5];    // ← espera el médico m
   bool atendido[P] = ([P] false);    cond yaAtendido[P];        // ← espera el paciente

   // ── PACIENTE ──
   procedure llegar (int id)
   { push(recepcion, id);  signal(hayPaciente); }

   procedure esperarAsignacion (int id; out int m)
   { if (not asignado[id]) wait(yaAsignado[id]);
     m = medicoDe[id]; }

   procedure anotarseConMedico (int id; int m)
   { insertarOrdenado(porMedico[m], id, gravedadDe[id]);   // mayor gravedad primero
     signal(hayEnMedico[m]); }

   procedure esperarAtencion (int id)
   { if (not atendido[id]) wait(yaAtendido[id]); }

   // ── ENFERMERA ──
   procedure proximoDeRecepcion (out int id)
   { while (empty(recepcion)) wait(hayPaciente);
     pop(recepcion, id); }

   procedure derivar (int id; int m; int g)
   { medicoDe[id] = m;  gravedadDe[id] = g;      // los DATOS primero
     asignado[id] = true;                        // la MEMORIA
     signal(yaAsignado[id]); }                   // la SEÑAL

   // ── MÉDICO ──
   procedure proximoPaciente (int m; out int id)
   { while (empty(porMedico[m])) wait(hayEnMedico[m]);
     pop(porMedico[m], id); }

   procedure terminarAtencion (int id)
   { atendido[id] = true;  signal(yaAtendido[id]); }
}


Process Paciente [id: 1..P]            Process Enfermera
{ int m;                               { int p, g, m;
  Guardia.llegar(id);                    for i = 1 to P {
  Guardia.esperarAsignacion(id, m);         Guardia.proximoDeRecepcion(p);
  Guardia.anotarseConMedico(id, m);         g = determinarGravedad(p);
  Guardia.esperarAtencion(id);              m = elegirMedico(p);
}                                           Guardia.derivar(p, m, g); } }

Process Medico [m: 1..5]
{ int p;
  while (true) {                       // reparto dinámico: no sabe cuántos le tocan
     Guardia.proximoPaciente(m, p);
     Atender(p);                       // AFUERA del monitor
     Guardia.terminarAtencion(p);
  }
}
```

## Las cuatro decisiones

1. **Cinco colas, una por médico.** El orden es *por médico*, no global.
2. **`insertarOrdenado` por gravedad**, descendente. Es la única línea que cambia respecto del orden de llegada.
3. **Las banderas `asignado[]` y `atendido[]`:** el paciente **sale del monitor** entre llamadas, así que el `signal` se puede perder.
4. **Los médicos van con `while(true)`:** ninguno sabe cuántos pacientes le van a tocar.

---

# 📝 Lo que erré

| Ej | Error | Regla |
|---|---|---|
| **1** | **Señales invertidas** (el cliente hacía `P` de "hay venta", el vendedor `V`) | *El que produce el hecho hace `V`* |
| **1** | `sem mutexCola = 0` | `P` y `V` en el mismo proceso ⇒ **candado ⇒ inicial 1** |
| **1** | `sem esperaDeAsiento = 1` | `P` y `V` en procesos distintos ⇒ **señal ⇒ inicial 0** |
| **1** | El `else` recargaba el micro (`contAsientos = 50`) | Hay un solo viaje |
| **1** | Faltaba la barrera del micro | *"espera a que los 50 hayan subido"* |
| **2** | Una sola cola para los 5 médicos | *"les indica a qué médico"* ⇒ la enfermera reparte |
| **2** | El paciente nunca recibía qué médico le tocó | El dato viaja por parámetro `out` |
| **2** | `signal(esperandoMedico[id])` con el id del **paciente** en un arreglo de **5 médicos** | Índices |
| **2** | Dos procedures con el mismo nombre (`recepcion`) | Nombres |

> **La causa raíz de casi todo el Ej. 1 fue una sola: la dirección de las señales.** Treinta segundos aplicando el test de abajo lo habrían cazado.

---

# 🔑 El procedimiento que faltaba: quién hace el `V`

**Paso 1 — Nombrá el semáforo como un HECHO, no como una acción.**

```
❌ ventaBoleto        →   ✅ hayCliente          ("hay un cliente esperando")
❌ esperaDeAsiento    →   ✅ tieneAsiento[id]    ("el cliente id ya sabe su asiento")
```

*Un nombre ambiguo te obliga a adivinar la dirección. Ese es el problema entero.*

**Paso 2 — ¿Quién hace que ese hecho sea VERDAD?** → ése hace **`V`**.
**Paso 3 — ¿Quién NECESITA que sea verdad para seguir?** → ése hace **`P`**.

**El valor inicial = cuántas veces el hecho YA es verdad al arrancar:**

```
"hay un cliente esperando"     →  al arrancar no llegó nadie  →  0
"la fotocopiadora está libre"  →  al arrancar sí está libre   →  1
"hay lugar para descargar"     →  al arrancar hay 7 lugares   →  7
```

## El test de 3 segundos

> **¿El `P` y el `V` están en el MISMO proceso?**
> **SÍ** → es un **permiso**: inicial **1** (candado) o **K** (contador).
> **NO** → es una **señal**: inicial **0**, siempre.

## Los dos chequeos que lo cierran

- **Nunca esperás algo que vos mismo producís.** Si un proceso hace `P` y `V` sobre la misma señal, está mal.
- **Cada señal tiene UN `V` de un lado y UN `P` del otro.** Dibujá las flechas antes de escribir: el origen avisa, el destino espera.
