# Moldes — Semáforos

> Para escanear antes del parcial. Cada molde: **cómo se reconoce · el código · las reglas que no se negocian.**

## Índice por frecuencia en los parciales viejos (28 enunciados)

| # | Molde | Veces |
|---|---|---|
| [7](#7--coordinador-) | **Coordinador** que atiende por orden y devuelve un dato | **~10** |
| [6](#6--passing-the-baton-) | **Passing the Baton** (orden sin coordinador) | **5** |
| [4](#4--barrera) | Barrera | ~6 |
| [9](#9--prioridad-variante-de-6-y-7) | Prioridad | 2 (+9 en monitores) |
| [10](#10--recurso-que-se-agota--repositor) | Recurso que se agota + repositor | 3 |
| [8](#8--posta-orden-por-identificador) | Posta (orden por identificador) | 3 |
| [5](#5--bolsa-de-tareas) | Bolsa de tareas | 3 |
| [2](#2--contador-de-recursos) · [3](#3--buffer-acotado-semáforos-cruzados) | Contador · Buffer acotado | varios |

---

## 🔑 Antes de todo: quién hace el `V` y cuál es el inicial

**Es el error que más caro sale y el más barato de evitar.** Tres pasos mecánicos:

**Paso 1 — Nombrá el semáforo como un HECHO, no como una acción.**

```
❌ ventaBoleto        →   ✅ hayCliente          ("hay un cliente esperando")
❌ esperaDeAsiento    →   ✅ tieneAsiento[id]    ("el cliente id ya sabe su asiento")
```

*Un nombre ambiguo te obliga a adivinar la dirección. Ese es el problema entero.*

**Paso 2 — ¿Quién hace que ese hecho sea VERDAD?** → ése hace **`V`**.
**Paso 3 — ¿Quién NECESITA que sea verdad para seguir?** → ése hace **`P`**.

**El valor inicial = cuántas veces el hecho YA es verdad al arrancar el programa:**

```
"hay un cliente esperando"     →  al arrancar no llegó nadie  →  0
"la fotocopiadora está libre"  →  al arrancar sí está libre   →  1
"hay lugar para descargar"     →  al arrancar hay 7 lugares   →  7
```

### El test de 3 segundos

> **¿El `P` y el `V` están en el MISMO proceso?**
> **SÍ** → es un **permiso**: inicial **1** (candado) o **K** (contador).
> **NO** → es una **señal**: inicial **0**, siempre.

### Dibujá las flechas antes de escribir

```
Cliente  ──── "llegué" ────►  Vendedor
Cliente  ◄── "tu asiento" ──  Vendedor
```

**Cada flecha es un semáforo en 0. El origen hace `V`, el destino hace `P`.**

Y los dos chequeos que lo cierran:

- **Nunca esperás algo que vos mismo producís.** Si un proceso hace `P` y `V` sobre la misma *señal*, está mal.
- **Cada señal tiene UN `V` de un lado y UN `P` del otro.**

---

## Las 4 reglas transversales

1. **El valor inicial lo define quién hace el `V`.** Mismo proceso que el `P` → candado/contador, inicial ≥ 1. **Otro** proceso → señal, inicial **0**.
2. **`P(contador)` antes que `P(mutex)`.** Nunca te bloqueás reteniendo un candado.
3. **La operación larga va afuera del candado** — salvo cuando el candado **es** el recurso (imprimir, fotocopiar).
4. **El dato se escribe ANTES de la señal.** Un semáforo transmite un instante, no un dato.

---

## 1 — Candado simple

*"de a uno a la vez" · "sin importar el orden"*

```
sem mutex = 1;
P(mutex);   usar();   V(mutex);
```

---

## 2 — Contador de recursos

*Un número de capacidad: "3 detectores", "7 camiones a la vez", "6 usuarios máximo"*

```
sem cupo = K;
P(cupo);   usar();   V(cupo);
```

**Varios límites a la vez** (7 en total, no más de 5 de cada tipo):

```
sem cupoTotal = 7;   sem cupoTrigo = 5;   sem cupoMaiz = 5;

P(cupoTrigo);      // ← ESPECÍFICO primero
P(cupoTotal);      // ← general después
descargar();
V(cupoTotal);  V(cupoTrigo);
```

> **Al revés te bloqueás reteniendo un lugar sin usarlo** → demora innecesaria. Pedí de lo más restrictivo a lo más general.

**Con identidad** (hay que saber *cuál* te tocó):

```
sem hayLibre = K;  cola libres;  sem mutexLibres = 1;   // libres = {1..K}
P(hayLibre);
P(mutexLibres);  pop(libres, r);  V(mutexLibres);
usar(r);
P(mutexLibres);  push(libres, r);  V(mutexLibres);
V(hayLibre);
```

---

## 3 — Buffer acotado (semáforos cruzados)

*"depósito" · "capacidad para N" · uno pone y otro saca*

```
sem vacios = N;      // lugares libres  → lo espera el que PONE
sem llenos = 0;      // cosas guardadas → lo espera el que SACA

Productor                        Consumidor
p = Producir();                  P(llenos);
P(vacios);                       x = buf[posSacar];
buf[posPoner] = p;               posSacar = (posSacar+1) mod N;
posPoner = (posPoner+1) mod N;   V(vacios);
V(llenos);                       Consumir(x);
```

- **Tantos pares cruzados como depósitos haya.**
- **Mutex en el índice sólo si hay más de uno del mismo rol.** Con 1 productor y 1 consumidor, ninguno.
- Donde hay mutex, **la escritura/lectura va ADENTRO** junto con el avance del índice.

---

## 4 — Barrera

*"una vez que **todos** … comienzan"*

```
int contador = 0;   sem mutexB = 1;   sem barrera = 0;

P(mutexB);
  contador++;
  ultimo = (contador == N);      // ← LOCAL, decidido ADENTRO
V(mutexB);
if (ultimo) { for j = 1 to N-1  V(barrera); }
else P(barrera);
```

> **Nunca consultes `contador` fuera del candado.** La variante en cascada (`if (not ultimo) P(barrera); V(barrera);`) también es correcta pero deja 1 permiso suelto.

**Barrera por grupos:** un `contador[G]` y una `barrera[G]`, todo indexado por grupo.

---

## 5 — Bolsa de tareas

*"mientras haya X por hacer, tomarán una" · "cada uno puede tardar distinto tiempo"*

```
int restantes = T;   sem mutexBolsa = 1;

hayParaMi = true;
while (hayParaMi) {
   P(mutexBolsa);
     hayParaMi = (restantes > 0);     // consulta y resta: UN SOLO acto
     if (hayParaMi) restantes--;
   V(mutexBolsa);                     // el candado se suelta EN CADA VUELTA
   if (hayParaMi) { hacer();  mias++; }   // AFUERA del candado
}
```

> **La bolsa NUNCA es un semáforo.** `P` bloquea cuando está vacío en vez de avisarte, y el programa no termina para ningún T y E. Va **entero + candado**.
>
> **La consulta va adentro y sale en una LOCAL.** `while (restantes > 0)` lee sin proteger.

---

## 6 — Passing the Baton ⭐

*"orden de llegada" **+** "sólo se pueden usar los procesos que representan a X"*

> Esa segunda frase **prohíbe el coordinador**. Aparece 5 veces: camiones · barcos areneros · pirosecuenciador · terminal SUBE · escaladores. **Es el mismo ejercicio.**

```
sem e = 1;                      // protege el ESTADO (no es el recurso)
sem priv[N] = ([N] 0);          // timbre con nombre
bool libre = true;
cola espera;

Process X [id: 1..N]
{ int sig;
  // ── LLEGADA ──
  P(e);
  if (libre) { libre = false;  V(e); }
  else { push(espera, id);  V(e);  P(priv[id]); }   // V(e) ANTES de dormirse

  usar();                       // AFUERA de e

  // ── SALIDA ──
  P(e);
  if (empty(espera)) libre = true;
  else { pop(espera, sig);  V(priv[sig]); }         // libre SIGUE en false
  V(e);
}
```

**La historia en 4 tiempos:** llego (¿libre? me lo quedo / me anoto y duermo) → uso → salgo (¿hay cola? le paso el permiso / lo dejo libre) → **el recurso no se libera, se entrega**.

**Las tres que no se negocian:**
1. `V(e)` **antes** del `P(priv[id])` — si no, deadlock.
2. `libre` **NO** vuelve a `true` al traspasar — si no, se cuela un recién llegado.
3. El uso, **afuera** de `e`.

---

## 7 — Coordinador ⭐⭐ (el más frecuente)

*"un X atiende a los N Y por orden de llegada" y le devuelve algo*

```
cola pedidos;                    sem mutexCola  = 1;      // N escritores
sem hayPedido  = 0;              // Cliente → Servidor
sem atendido[N] = ([N] 0);       // Servidor → Cliente
tipo dato[N];                    // el DATO que vuelve

Process Cliente [id: 1..N]
{ P(mutexCola);  push(pedidos, id);  V(mutexCola);
  V(hayPedido);
  P(atendido[id]);
  mi_dato = dato[id];
}

Process Servidor
{ int c;
  for i = 1 to N {               // total conocido → TERMINA
     P(hayPedido);
     P(mutexCola);  pop(pedidos, c);  V(mutexCola);
     ...atiendo...
     dato[c] = resultado;        // el DATO primero
     V(atendido[c]);             // la SEÑAL después
  }
}
```

**Variantes que aparecen:**

- **El servidor toma los cupos en nombre del cliente** (cerealera): mete `P(cupo)` antes del `V(atendido[c])`. Al bloquearse **él**, nadie adelanta al primero de la fila → eso **es** el orden de llegada.
- **Dos etapas encadenadas** (planta verificadora, terminal de micros): el molde **dos veces**, el segundo indexado por puesto/estación.
- **Servidores múltiples** (E empleados): `while (empty(cola)) wait/P` con **`while`**, y si el reparto es dinámico **no uses `for`** → ver §11.
- **Asignar el puesto con menos gente:** `int asignados[K]` + su mutex. **Elegir el mínimo e incrementarlo van en la MISMA sección crítica** — si no, dos clientes van al mismo. El servidor del puesto **decrementa** al atender.

---

## 8 — Posta (orden por identificador)

*"la persona X no puede usar hasta que termine X−1"* — orden **estático**, se conoce de antemano

```
sem turno[N] = ([1] 1, [N-1] 0);     // sólo el primero arranca

Process X [id: 1..N]
{ P(turno[id]);
  usar();
  if (id < N) V(turno[id+1]);        // el último no avisa a nadie
}
```

**Ni cola ni mutex:** existe un único permiso circulando. Costo: el recurso puede quedar **ocioso** esperando al que le toca.

> **Ojo:** si el enunciado dice *"de los que **han solicitado**, el de menor id"*, es prioridad **dinámica** → molde 6 o 7 con selección del mínimo, no posta.
> Y si dice *"no se puede usar el ID como turno"*, indexá por **turno**, no por id.

---

## 9 — Prioridad (variante de 6 y 7)

*"el de mayor edad" · "dando prioridad a las ambulancias" · "menor valor = mayor prioridad"*

> **Cambia UNA línea: cómo insertás en la cola.** Todo el resto del molde es idéntico.

```
push(cola, id)                               →  orden de llegada
insertarOrdenado(cola, id, edad)             →  mayor edad primero   (DESCENDENTE)
insertarOrdenado(cola, id, gravedad)         →  mayor gravedad       (DESCENDENTE)
insertarOrdenado(cola, id, prioridad)        →  menor valor primero  (ASCENDENTE)
```

**Aclará siempre el sentido del orden en un comentario.** Es gratis y en el parcial de 2022 el criterio es al revés del intuitivo.

*(Para dos categorías —ambulancias vs. autos— también sirven **dos colas**: se atiende la de prioridad mientras no esté vacía.)*

---

## 10 — Recurso que se agota + repositor

*"si la máquina se queda sin latas, le avisa al repositor"* · **nota clave:** *"mientras se repone, otros se deben poder agregar a la fila"*

**Molde 6 (o 7) + dos semáforos y un contador.** Tres tiempos:

```
int latas = 100;                 // sólo lo toca EL QUE TIENE LA MÁQUINA
sem avisarRepositor = 0;         // Usuario → Repositor
sem recargaLista    = 0;         // Repositor → Usuario

Process Usuario [id: 1..U]
{ // 1. CONSEGUIR EL TURNO   ← molde 6, intacto
  // 2. USAR
  if (latas == 0) { V(avisarRepositor);  P(recargaLista); }
  latas--;
  SacarLata();
  // 3. SOLTAR EL TURNO      ← molde 6, intacto
}

Process Repositor
{ while (true) {
     P(avisarRepositor);
     Reponer();                  // afuera de todo candado
     latas = 100;
     V(recargaLista);
  }
}
```

**Las tres claves:**
1. **Tener la máquina ≠ tener el candado.** El usuario soltó `e` al conseguir el turno, así que mientras espera la recarga **otros se anotan**. La nota se cumple sola.
2. **`latas` no lleva candado:** sólo lo toca el que tiene la máquina (uno por vez), y el repositor lo escribe mientras ese usuario está bloqueado.
3. **El que esperó la recarga NO vuelve a la cola:** ya tenía el turno.

---

## 11 — Terminación con reparto dinámico

*"deben terminar su ejecución cuando…"*

> **La pregunta: ¿cada proceso sabe de antemano cuántas vueltas le tocan?**
> **Sí** (reparto fijo) → **`for` acotado** y listo.
> **No** (reparto dinámico: si uno atiende 30 y otro 20) → **contador + despertar a los dormidos + bandera de salida**.

**Las tres piezas, siempre:**

```
1. un CONTADOR compartido de lo ya repartido
2. despertar a los que YA están dormidos cuando se acaba   ← el que más se olvida
3. una BANDERA de salida para el que despierta
```

**Olvidarse de la 2 es silencioso:** el programa da resultados correctos y simplemente **no termina**.

---

## ✅ Checklist de entrega (30 segundos)

```
1. Sólo los NOMBRES      ¿alguno significa dos cosas? ¿declaración y uso coinciden?
                         ¿loops anidados con variables distintas? ¿índices [id], no [N]?
2. Sólo los INICIALES    por cada semáforo: ¿quién hace el V?
                         mismo proceso → ≥1 · otro proceso → 0
3. Sólo las LLAMADAS     ¿cada procedure recibe los parámetros que declara?
4. Sólo los while(true)  ¿el enunciado da un total? entonces TIENE que terminar
5. Sólo las OPS LARGAS   ¿están afuera del candado?
```
