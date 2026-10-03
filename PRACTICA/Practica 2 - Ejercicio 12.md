# Práctica 2 — Ejercicio 12

> **Enunciado.** Simular la atención en una **Terminal de Micros** que posee **3 puestos** para hisopar a **150 pasajeros**. En cada puesto hay una **Enfermera** que atiende a los pasajeros **de acuerdo con el orden de llegada al mismo**. Cuando llega un pasajero se dirige al **Recepcionista**, quien le indica **qué puesto es el que tiene menos gente esperando**. Luego se dirige al puesto y **espera a que la enfermera correspondiente lo llame** para hisoparlo. Finalmente, se retira.
> **a)** Implemente una solución considerando los procesos **Pasajeros, Enfermera y Recepcionista**.
> **b)** Modifique la solución anterior para que **sólo haya procesos Pasajeros y Enfermera**, siendo **los pasajeros quienes determinan por su cuenta** qué puesto tiene menos personas esperando.
> *Nota: existe una función `Hisopar()` que simula la atención del pasajero por parte de la enfermera correspondiente.*

---

## 0. El mapa del ejercicio

Es el más largo de la práctica, pero no trae ningún patrón nuevo: son **dos traspasos encadenados**.

```
Pasajero  ──(1)──►  Recepcionista  ──(2)──►  se va al puesto asignado
                          │
                          └─ le devuelve QUÉ PUESTO

Pasajero  ──(3)──►  cola del puesto p  ──(4)──►  Enfermera[p] lo llama e hisopa
```

**Procesos:** `Pasajero[1..150]`, `Enfermera[1..3]`, `Recepcionista` (uno).
Tanto el recepcionista (*"le indica qué puesto"*) como las enfermeras (*"lo llame"*) **deciden** ⇒ los tres son procesos.

**Terminación:** cada pasajero se hisopa una vez. El recepcionista atiende **150** y se retira. Las enfermeras **no tienen un total propio** —cuántos le tocan a cada una depende del reparto, que es dinámico— y el enunciado no exige que terminen, así que van con `while(true)`.

---

## 1. Hay tres colas, no una

*"de acuerdo con el orden de llegada **al mismo**"* — el orden es **por puesto**, no global. Un pasajero que llegó primero a la terminal pero fue mandado al puesto 2 no tiene prioridad sobre uno del puesto 1.

> **Tres puestos ⇒ tres colas independientes, tres candados, tres señales.** Todo lo del puesto va indexado `[1..3]`.

## 2. `esperando[3]`: la variable que todos tocan

Para saber *"qué puesto tiene menos gente esperando"* hace falta un contador por puesto. Y tiene **dos tipos de escritores**:

| Quién | Qué hace |
|---|---|
| El **recepcionista** (o el pasajero, en el ítem b) | `esperando[p]++` al mandar a alguien |
| Cada **enfermera** | `esperando[p]--` al llamar a alguien |

Cuatro escritores en el ítem (a) ⇒ **`esperando[]` lleva candado**. Es la única variable del ejercicio que lo necesita por este motivo.

---

## 3. Solución (a)

```
// PRECONDICIONES: 150 pasajeros, 3 puestos; todas las colas arrancan vacías

// ── Recepción ──
cola recepcion;                          // ids, en orden de llegada a la terminal
sem mutexRecepcion = 1;                  // 150 escritores
sem hayPasajero = 0;                     // señal: "llegó alguien a recepción"
sem asignado[150] = ([150] 0);           // señal: "ya sé tu puesto"
int puestoDe[150];                       // el DATO: qué puesto le tocó a cada uno

// ── Puestos ──
int esperando[1..3] = ([3] 0);           // cuántos esperan en cada puesto
sem mutexEsperando = 1;                  // lo escriben recepcionista y enfermeras

cola colaPuesto[1..3];                   // una cola POR PUESTO
sem mutexPuesto[1..3] = ([3] 1);
sem hayEnPuesto[1..3] = ([3] 0);         // señal: "llegó alguien a tu puesto"
sem hisopado[150] = ([150] 0);           // señal: "listo, te hisopé"


Process Pasajero [id: 1..150]
{ int mi_puesto;                                    // LOCAL

  // ── (1) voy al recepcionista ──
  P(mutexRecepcion);  push(recepcion, id);  V(mutexRecepcion);
  V(hayPasajero);

  // ── (2) espero que me diga el puesto ──
  P(asignado[id]);
  mi_puesto = puestoDe[id];                         // leo el DATO

  // ── (3) me anoto en la cola de ESE puesto ──
  P(mutexPuesto[mi_puesto]);
    push(colaPuesto[mi_puesto], id);
  V(mutexPuesto[mi_puesto]);
  V(hayEnPuesto[mi_puesto]);

  // ── (4) espero que la enfermera me llame e hisope ──
  P(hisopado[id]);
}                                                   // me retiro


Process Recepcionista
{ int i, p, menor;

  for i = 1 to 150 {                                // atiendo a los 150 y me retiro
     P(hayPasajero);
     P(mutexRecepcion);  pop(recepcion, p);  V(mutexRecepcion);

     P(mutexEsperando);
       menor = el índice de 1..3 con menor esperando[];
       esperando[menor]++;                          // lo anoto ya, ADENTRO
     V(mutexEsperando);

     puestoDe[p] = menor;                           // el DATO primero
     V(asignado[p]);                                // la SEÑAL después
  }
}


Process Enfermera [p: 1..3]
{ int id;
  while (true) {                                    // sin total propio conocido
     P(hayEnPuesto[p]);                             // espero que haya alguien en MI puesto
     P(mutexPuesto[p]);
       pop(colaPuesto[p], id);                      // el primero de MI fila
     V(mutexPuesto[p]);

     P(mutexEsperando);  esperando[p]--;  V(mutexEsperando);   // ya no espera

     Hisopar();                                     // AFUERA de todo candado
     V(hisopado[id]);                               // lo dejo ir
  }
}
```

### Las dos decisiones que hay que saber defender

**(a) Elegir el mínimo y anotarlo van en el MISMO candado.**
```
P(mutexEsperando);
  menor = índice del mínimo;
  esperando[menor]++;          // ← si esto quedara afuera…
V(mutexEsperando);
```
Si fueran dos secciones críticas separadas, dos asignaciones consecutivas verían el mismo mínimo y **mandarían a los dos al mismo puesto**, desbalanceando justo lo que se quería balancear.

**(b) `Hisopar()` afuera de todo candado.** Si estuviera dentro de `mutexEsperando`, las **tres enfermeras trabajarían de a una**. Es lo caro y no necesita nada compartido.

---

## 4. Solución (b) — sin recepcionista

> *"siendo los pasajeros quienes determinan por su cuenta qué puesto tiene menos personas esperando"*

**Desaparece el proceso Recepcionista y con él toda la recepción**: `cola recepcion`, `mutexRecepcion`, `hayPasajero`, `asignado[]` y `puestoDe[]`. No hace falta transmitirle el puesto a nadie — el pasajero lo calcula él mismo.

El pasajero reemplaza los pasos (1) y (2) por **una sola sección crítica**:

```
Process Pasajero [id: 1..150]
{ int mi_puesto;

  // ── (1+2) elijo mi puesto YO ──
  P(mutexEsperando);
    mi_puesto = el índice de 1..3 con menor esperando[];
    esperando[mi_puesto]++;              // elegir y anotarme: UN SOLO acto
  V(mutexEsperando);

  // ── (3) y (4): idénticos al ítem (a) ──
  P(mutexPuesto[mi_puesto]);  push(colaPuesto[mi_puesto], id);  V(mutexPuesto[mi_puesto]);
  V(hayEnPuesto[mi_puesto]);
  P(hisopado[id]);
}
```

**Las enfermeras no cambian una línea.** Eso es señal de que el modelo está bien: ellas atienden su cola, y no les importa quién decidió el reparto.

> **Lo único que hay que justificar del (b):** **elegir el mínimo y anotarse tienen que ser atómicos**. Si el pasajero mirara el mínimo, soltara el candado y después incrementara, dos pasajeros verían el mismo puesto vacío y los dos irían ahí. Es exactamente el mismo razonamiento que *"consultar y llevarse"* en una bolsa de tareas.

---

## 5. Justificación (esto es lo que suma puntos)

> Hay tres tipos de procesos porque tanto el recepcionista como las enfermeras **deciden** (a qué puesto mandar, a quién llamar), a diferencia de los puestos, que son sólo lugares. El orden exigido es **por puesto** —*"orden de llegada al mismo"*—, así que hay tres colas independientes con su candado y su señal, y no una cola global. Cada pasajero espera en semáforos privados porque hay que despertar a uno determinado y no puede suponerse que los semáforos sean FIFO. El recepcionista le comunica al pasajero **dos cosas distintas**: el instante, por `asignado[id]`, y el número de puesto, por el arreglo compartido `puestoDe[]`, escrito **antes** de la señal, ya que un semáforo no transmite datos. `esperando[]` requiere exclusión mutua porque lo incrementa el recepcionista y lo decrementan las tres enfermeras; la elección del mínimo y el incremento ocurren en la **misma** sección crítica, porque si se separaran dos asignaciones consecutivas elegirían el mismo puesto y se rompería el balanceo. `Hisopar()` queda fuera de todo candado para que las tres enfermeras atiendan en paralelo. El recepcionista atiende exactamente 150 pasajeros y se retira; las enfermeras no tienen un total propio conocido —el reparto es dinámico— y el enunciado no exige su terminación, por lo que ejecutan un ciclo indefinido. En el ítem (b) desaparece toda la recepción y el pasajero elige su propio puesto, manteniendo la atomicidad de *elegir e incrementar*; las enfermeras no requieren ningún cambio.

---

## 6. Los errores que se penalizan

**(1) Una sola cola global.** El orden es *"de llegada **al mismo** [puesto]"*. Con una cola global, un pasajero del puesto 3 podría bloquear a uno del puesto 1.

**(2) Elegir el mínimo y anotarse en secciones críticas separadas.** Dos pasajeros al mismo puesto. Es **el** error del ítem (b).

**(3) No decrementar `esperando[]`.** Los contadores sólo suben y el "menos gente esperando" deja de significar nada: todos terminan en el puesto que empezó vacío por casualidad.

**(4) `Hisopar()` dentro de un candado compartido.** Las 3 enfermeras rinden como 1.

**(5) Un semáforo global en vez de `hisopado[150]` o `asignado[150]`.** Despierta a *alguno*.

**(6) `V(asignado[p])` antes de escribir `puestoDe[p]`.** El pasajero se despierta y lee un casillero vacío. **El dato primero.**

**(7) `while(true)` en el recepcionista.** Atiende 150 y se retira; si no, queda colgado en `P(hayPasajero)`.

**(8) Dejar la recepción en el ítem (b).** Es exactamente lo que el ítem pide eliminar.

---

## 7. Chequeos de sanidad

- **Contá:** `hayPasajero` recibe 150 `V` y 150 `P`. Cada `asignado[i]` y cada `hisopado[i]`, exactamente un `V` y un `P`.
- **`esperando[1] + esperando[2] + esperando[3]`** = cantidad de pasajeros que fueron asignados pero todavía no fueron llamados. Nunca negativo.
- **`Hisopar()` se ejecuta 150 veces** en total, repartidas entre las tres enfermeras.
- **Con 1 puesto** el ejercicio colapsa a una sola cola con una enfermera, y el cálculo del mínimo es trivial. ✔
- **Sin deadlock:** ningún `P` se ejecuta reteniendo un candado; `mutexEsperando` se toma y se suelta sin bloquearse en el medio.

---

## 8. Los tres chequeos del corrector

1. **¿Hay tres colas (una por puesto) y no una global?** El orden es por puesto.
2. **¿Elegir el mínimo y anotarse son atómicos?** Separados, dos pasajeros van al mismo lugar.
3. **¿El número de puesto viaja por una variable, escrita antes de la señal?** Un semáforo transmite un instante, no un dato.
