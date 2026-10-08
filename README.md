# Programación Concurrente — UNLP 2026

Apuntes propios de la cursada: resoluciones comentadas de las prácticas y repasos de teoría.

> **Qué hay acá:** no son las soluciones peladas. Cada ejercicio tiene el razonamiento de cómo se llega a la solución, las trazas que muestran por qué las alternativas fallan, y los errores que se penalizan.

---

## 🚀 Por dónde empezar

| Si querés… | Andá a |
|---|---|
| **El formulario** — todas las soluciones juntas, sin justificaciones | [`PRACTICA/Soluciones.md`](PRACTICA/Soluciones.md) |
| **Decidir qué mecanismo usar** leyendo un enunciado nuevo | [`REPASOS/Pistas de enunciados.md`](REPASOS/Pistas%20de%20enunciados.md) |
| **Entender un ejercicio en profundidad** | El `.md` individual en [`PRACTICA/`](PRACTICA/) |
| **Repasar teoría** | [`REPASOS/`](REPASOS/) |
| **Ver qué toca estudiar cada semana** hasta noviembre | [`Plan de estudio - Octubre-Noviembre.md`](Plan%20de%20estudio%20-%20Octubre-Noviembre.md) |

> **El material de la cátedra no está en el repo** (enunciados, filminas, parciales viejos en PDF): el repo es público. Los `.md` citan los enunciados en el texto.

---

## 📘 Repasos de teoría

| Archivo | Contenido |
|---|---|
| [Clase 1 — Repaso rápido](REPASOS/Clase%201%20-%20Repaso%20rapido.md) | Conceptos iniciales de concurrencia |
| [Teoría 2 — Repaso](REPASOS/Teoria%202%20-%20Repaso.md) | Las 4 propiedades de la sección crítica · grano grueso → TS → Tie-Breaker / Ticket / Bakery · barreras (contador, flags y coordinador, árboles, butterfly) |
| [Clase 3 — Semáforos](REPASOS/Clase%203%20-%20Repaso%20(Semaforos).md) | El semáforo y sus 3 usos según el valor inicial · SBS · contadores de recursos · alocación · **Passing the Baton** · lectores/escritores |
| [Teoría 4 — Monitores](REPASOS/Teoria%204%20-%20Repaso%20(Monitores).md) | Exclusión mutua implícita vs. sincronización explícita · `wait`/`signal` vs. `P`/`V` · **signal-and-continue y la regla del `while`** · las 6 técnicas · bloquearse dentro de un monitor |
| [Teorías 5 y 6 — PMA y PMS](REPASOS/Teoria%205%20y%206%20-%20Repaso%20(PMA%20y%20PMS).md) | Qué cambia sin memoria compartida · PMA vs. PMS lado a lado · **moldes base** (A1–A9 de PMA, S1–S5 de PMS) · machete de reconocimiento · mapa de la Práctica 4 · puente a RPC y Rendezvous |
| [**Pistas de enunciados**](REPASOS/Pistas%20de%20enunciados.md) | **Diccionario frase → mecanismo**, armado sobre los 19 enunciados de las prácticas 1 y 2. Incluye una chuleta de una carilla |
| [Glosario](REPASOS/Glosario.md) | Términos transversales |

---

## 📝 Práctica 1 — Variables compartidas

| Ej | Tema | Idea central |
|---|---|---|
| [1](PRACTICA/Practica%201%20-%20Ejercicio%201.md) | ¿`x` termina en 56, 22 o 23? | Referencias críticas y la propiedad **A lo sumo una vez** |
| [2](PRACTICA/Practica%201%20-%20Ejercicio%202.md) | Contar apariciones de N en un arreglo | Reparto por bloques + acumulador local + merge atómico |
| [3](PRACTICA/Practica%201%20-%20Ejercicio%203.md) | Productor/Consumidor con buffer de N | El bug del `cant++` antes de escribir el buffer |
| [4](PRACTICA/Practica%201%20-%20Ejercicio%204.md) | 5 instancias de un recurso en una cola | `<await B; S>` y el uso fuera de la sección crítica |

## 🚦 Práctica 2 — Semáforos

| Ej | Tema | Idea central |
|---|---|---|
| [1](PRACTICA/Practica%202%20-%20Ejercicio%201.md) | Detector de metales | Receta de modelado · recurso pasivo vs. proceso activo |
| [2](PRACTICA/Practica%202%20-%20Ejercicio%202.md) | Historial de fallos | Repartir bien el trabajo puede **eliminar** la sincronización |
| [3](PRACTICA/Practica%202%20-%20Ejercicio%203.md) | 5 instancias en una cola | Contador + cola de identidades · dos secciones críticas |
| [4](PRACTICA/Practica%202%20-%20Ejercicio%204.md) | BD con límites 6/4/5 | Demora innecesaria y el orden de los `P` |
| [5](PRACTICA/Practica%202%20-%20Ejercicio%205.md) | Sala de contenedores | **Buffer acotado**: semáforos cruzados · cuándo NO hace falta mutex |
| [6](PRACTICA/Practica%202%20-%20Ejercicio%206.md) | Impresora (a–e) | **Passing the Baton** · la posta · coordinador · 5 impresoras |
| [7](PRACTICA/Practica%202%20-%20Ejercicio%207.md) | 50 alumnos, 10 tareas | **Barrera** + productor/consumidor + traspaso con coordinador |
| [8](PRACTICA/Practica%202%20-%20Ejercicio%208.md) | Fábrica de piezas | **Bolsa de tareas**: por qué un semáforo no sirve como bolsa |
| [9](PRACTICA/Practica%202%20-%20Ejercicio%209.md) | Fábrica de ventanas | Dos buffers acotados · orden fijo de los `P` contra el deadlock |
| [10](PRACTICA/Practica%202%20-%20Ejercicio%2010.md) | Cerealera | Contadores múltiples · el coordinador se bloquea en nombre del cliente |
| [11](PRACTICA/Practica%202%20-%20Ejercicio%2011.md) | Vacunatorio | *"Al menos 5"* = cinco `P` seguidos |
| [12](PRACTICA/Practica%202%20-%20Ejercicio%2012.md) | Terminal de micros | Tres colas · elegir el mínimo e incrementarlo, atómico |

## 🖥️ Práctica 3 — Monitores

| Ej | Tema | Idea central |
|---|---|---|
| [4](PRACTICA/Practica%203%20-%20Ejercicio%204.md) | Puente de 50000 kg | **Passing the Condition** · el estado es un número, no un booleano |
| [5](PRACTICA/Practica%203%20-%20Ejercicio%205.md) | Corralón (a, b, c) | Rendezvous bidireccional · **los 3 tipos de `wait`** · la señal perdida · la regla general de terminación |

*(En progreso — faltan los ejercicios 1, 2, 3, 6, 7, 8, 9 y 10.)*

En [`REPASOS CONCU/`](REPASOS%20CONCU/) están mis escrituras de práctica de los moldes. Los que empiezan con `_BORRADOR CON ERRORES` quedaron así a propósito: **no copiarlos**.

## ✉️ Práctica 4 — Pasaje de mensajes (PMA y PMS)

*(Arrancando.)* Por ahora, el mapa de qué molde va en cada ejercicio está en la sección 5 del [repaso de PMA y PMS](REPASOS/Teoria%205%20y%206%20-%20Repaso%20(PMA%20y%20PMS).md).

---

## 🎯 Preparación de los parciales

| Fecha | Qué |
|---|---|
| 5 oct | Parcial de **Memoria Compartida** (semáforos y monitores) — ya rendido |
| **9 nov** | Parcial de **Memoria Distribuida** (PMA, PMS y Ada) |
| **30 nov** | **Fecha 2**, primer recuperatorio. Para promocionar el final, la cursada tiene que estar aprobada a más tardar acá |
| 14 dic | Fecha 3 y examen de promoción |

### Memoria Compartida

En [`PARCIAL 1 - MEMORIA COMPARTIDA/MODELOS/`](PARCIAL%201%20-%20MEMORIA%20COMPARTIDA/MODELOS/) está **lo que conviene tener a mano el día antes**:

| Archivo | Qué tiene |
|---|---|
| [**Moldes — Semáforos**](PARCIAL%201%20-%20MEMORIA%20COMPARTIDA/MODELOS/Moldes%20-%20Semaforos.md) | 11 moldes **ordenados por frecuencia** en los parciales viejos, las 4 reglas transversales y un checklist de entrega |
| [**Moldes — Monitores**](PARCIAL%201%20-%20MEMORIA%20COMPARTIDA/MODELOS/Moldes%20-%20Monitores.md) | 7 moldes y las 10 reglas que gobiernan todo: los tres tipos de `wait`, la señal perdida, `cond` arreglo vs. sola |
| [**Molde Coordinador — los dos mecanismos**](PARCIAL%201%20-%20MEMORIA%20COMPARTIDA/MODELOS/Molde%20Coordinador%20-%20los%20dos%20mecanismos.md) | El mismo problema con semáforos y con monitores, y qué cambia al traducir |
| [**Parcial 2022 — Resuelto**](PARCIAL%201%20-%20MEMORIA%20COMPARTIDA/MODELOS/Parcial%202022%20-%20Resuelto.md) | Los dos ejercicios resueltos y comentados |
| [**Simulacro 2 — Resuelto**](PARCIAL%201%20-%20MEMORIA%20COMPARTIDA/MODELOS/Simulacro%202%20-%20Resuelto.md) | El micro (semáforos) y la guardia de traumatología (monitores) |
| [**Parcial 2026 — Resuelto**](PARCIAL%201%20-%20MEMORIA%20COMPARTIDA/MODELOS/Parcial%202026%20-%20Resuelto.md) | El parcial real: sala de ensayo con prioridad (PtB) y viajes compartidos de a 4 (grupos con monitores), con lo que salió bien y mal |

**Dato del formato:** el parcial son **dos ejercicios** —uno de semáforos y uno de monitores— y **no se escribe justificación**: código, precondiciones y comentarios cortos. En los 51 enunciados viejos relevados, **49 piden orden de llegada o prioridad**.

---

## 🧭 Las dos ideas que atraviesan todo

**1. Un semáforo transmite un instante, no un dato.**
`V(s)` dice *"ya"*; no puede decir *"ya, y el número es 4"*. El dato viaja por una variable compartida (o por un parámetro, en monitores), **escrita antes de la señal**.

**2. El recurso no se libera: se entrega.**
Es el corazón de **Passing the Baton** (semáforos) y **Passing the Condition** (monitores) — la misma idea en dos mecanismos. La variable de estado **no vuelve a "libre"** mientras haya cola; si volviera, un recién llegado se colaría y se perdería el orden que costó construir.

---

## Convenciones

- Pseudocódigo con la notación de la cátedra: `P`/`V` para semáforos, `wait`/`signal`/`signalall` para monitores, `send`/`receive` para PMA, `Destino!port(...)`/`Origen?port(...)` para PMS, y operaciones funcionales sobre colas (`push(cola, x)`, `pop(cola, x)`, `empty(cola)`).
- Cada `.md` sigue la misma estructura: enunciado → análisis → solución comentada → justificación → errores que se penalizan → chequeos de sanidad.
