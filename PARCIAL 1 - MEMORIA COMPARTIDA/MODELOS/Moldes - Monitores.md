# Moldes — Monitores

> Para escanear antes del parcial. **En los 23 enunciados viejos de monitores, TODOS piden orden de llegada o prioridad.** El molde 2 y su variante 3 son el 80% del trabajo.

---

## Las reglas que gobiernan todo

**1. La exclusión mutua es implícita.** Nunca escribas un candado adentro de un monitor.

**2. El valor inicial de una `cond` no existe** — la `cond` no tiene valor, tiene una cola. **Lo que arranca con un valor son tus variables permanentes.**

**3. `signal` sin nadie esperando SE PIERDE.** (`V` en semáforos queda guardado; `signal` no.) Todo lo que haya que **recordar** va en variables permanentes.

**4. Signal and continue ⇒ los tres tipos de `wait`:**

| Situación | Forma |
|---|---|
| *"esperá a que se cumpla una condición"* — después del `wait` **modificás estado** | **`while`** |
| *"te paso el turno, ya te dejé todo hecho"* — después del `wait` **no tocás nada** | **`if`** (Passing the Condition) |
| *"nos esperamos mutuamente y tengo un `signal` garantizado para mí"* | **`wait` pelado** (rendezvous) |

> **No hay default seguro.** `while` en un Passing the Condition te deja durmiendo **para siempre**.

**5. Un `wait` pelado sólo es seguro si te anotás y te dormís SIN soltar el monitor en el medio.** Si salís del monitor y volvés a entrar, la señal te puede pasar por al lado → ahí hace falta una **bandera**.

**6. Los datos viajan por parámetros.** No hay variables globales. El dato se escribe **antes** de la señal.

**7. `cond` arreglo vs. sola — y pasar o no el `id`: es la MISMA pregunta.** *¿El monitor tiene que distinguirlo?*
- Hay que despertar a **uno puntual** → `cond cv[N]` **+** el `id` como parámetro.
- Da igual a quién despertar ("alguno de los empleados") → **una sola `cond`**, sin `id`.

**8. `empty(miCola)` SÍ se puede** — es una estructura tuya. Lo prohibido es `empty(cv)`, `wait(cv,rank)` y `minrank(cv)` sobre variables condición.

**9. El `while(true)` va en el PROCESO, nunca adentro del monitor.** Un procedure tiene que terminar: si no, el monitor queda tomado para siempre.

**10. La operación larga va afuera del monitor.** Adentro **todo** se serializa, no sólo una sección crítica.

---

## 1 — Condición básica

*"de a uno a la vez" · "a lo sumo K simultáneos" · sin orden*

```
monitor Recurso {
   bool libre = true;
   cond esperar;                        // una sola: da igual a quién despertar

   procedure pedir ()
   { while (not libre) wait(esperar);   // ← WHILE: después modifico estado
     libre = false; }

   procedure liberar ()
   { libre = true;
     signal(esperar); }
}

Process X [id: 1..N]
{ Recurso.pedir();   usar();   Recurso.liberar(); }   // usar() AFUERA
```

**Con K simultáneos** (ej.: 5 consultas a la BD): cambiá `bool libre` por `int disponibles = K`, `while (disponibles == 0) wait(...)`, `disponibles--` / `disponibles++`.

---

## 2 — Passing the Condition ⭐⭐ (el molde del parcial)

*"de acuerdo con el orden de llegada"*

```
monitor Recurso {
   bool libre = true;
   cond puedeUsar[N];              // arreglo: hay que despertar a ESE
   cola espera;                    // ids, en orden de llegada

   procedure pedir (int id)
   { if (libre) libre = false;                  // libre: me lo quedo
     else { push(espera, id);                   // ocupado: me anoto
            wait(puedeUsar[id]); } }            // ← IF, no while

   procedure liberar ()                         // SIN parámetros
   { int sig;
     if (empty(espera)) libre = true;
     else { pop(espera, sig);
            signal(puedeUsar[sig]); } }         // libre SIGUE en false
}

Process X [id: 1..N]
{ Recurso.pedir(id);   usar();   Recurso.liberar(); }
```

**Es el molde de Passing the Baton sin el `sem e`** — la exclusión mutua la pone el monitor.

**Las tres que no se negocian:**
1. **`if`, no `while`** — el que libera ya te dejó el estado hecho.
2. **`libre` NO vuelve a `true`** en la rama del traspaso.
3. **La actualización del estado va en el `else`** del `pedir`. Si la ponés después del `if`, el que despierta la ejecuta **de nuevo** y se duplica.

> **Las decisiones 1 y 3 son la misma:** si después del `wait` hay código tuyo que modifica estado, entonces necesitabas `while`.

---

## 3 — Prioridad (una línea del molde 2)

*"mayor edad" · "mayor gravedad" · "menor valor = mayor prioridad" · "ancianos y embarazadas"*

```
procedure pedir (int id, int prioridad)        // ← el dato entra por parámetro
{ if (libre) libre = false;
  else { insertarOrdenado(espera, id, prioridad);     // ← LA ÚNICA LÍNEA QUE CAMBIA
         wait(puedeUsar[id]); } }
```

`liberar()` **no cambia una coma**: el `pop` ya saca al de mayor prioridad.

> **El criterio de orden vive en la estructura, no en el protocolo.**
> **Aclará el sentido** en un comentario: edad y gravedad son **descendentes**; *"menor valor indica mayor prioridad"* es **ascendente**.

---

## 4 — Servidor/coordinador con datos de ida y vuelta

*"el cliente entrega X y espera a que le devuelvan Y"* · rendezvous

**La regla de cuántos procedures:** *cada vez que un proceso tiene que hacer algo **afuera** del monitor en el medio de su secuencia, se corta en dos llamadas.* Típicamente: **4 procedures, 2 por actor.**

```
monitor Servicio {
   cola espera;                    // ids + datos, en orden de llegada
   cond hayCliente;                // UNA sola: cualquier servidor sirve
   cond esperaServidor[N];         // arreglo
   tipo resultadoDe[N];            // el DATO que vuelve
   bool listo[N] = ([N] false);    // la MEMORIA  ← evita la señal perdida
   cond resListo[N];

   procedure hacerFila (int id; tipo pedido)              // CLIENTE
   { push(espera, id, pedido);
     signal(hayCliente);
     wait(esperaServidor[id]); }                          // PELADO

   procedure proximoCliente (out int id; out tipo pedido) // SERVIDOR
   { while (empty(espera)) wait(hayCliente);              // WHILE
     pop(espera, id, pedido);
     signal(esperaServidor[id]); }

   procedure entregar (int id; tipo r)                    // SERVIDOR
   { resultadoDe[id] = r;
     listo[id] = true;                                    // ← el REGISTRO
     signal(resListo[id]); }

   procedure recibir (int id; out tipo r)                 // CLIENTE
   { if (not listo[id]) wait(resListo[id]);               // IF + bandera
     r = resultadoDe[id]; }
}

Process Cliente [id: 1..N]           Process Servidor [s: 1..E]
{ Servicio.hacerFila(id, pedido);    { int c;  tipo p, r;
  Servicio.recibir(id, r); }           while (true) {
                                          Servicio.proximoCliente(c, p);
                                          r = procesar(p);     // AFUERA
                                          Servicio.entregar(c, r); } }
```

**Por qué la bandera `listo[]`:** el cliente **sale del monitor** entre sus dos llamadas. Si el servidor termina rápido, el `signal` cae en el vacío y el cliente duerme para siempre. *El booleano recuerda, la `cond` duerme y despierta.*

---

## 5 — Terminación con reparto dinámico

*"los servidores deben terminar cuando se haya atendido a todos"*

> Si **cada proceso sabe cuántas vueltas le tocan** → `for` acotado. Si **el reparto es dinámico** (uno atiende 30 y otro 20) → estas tres piezas.

```
int tomados = 0;                     // 1. CONTADOR

procedure proximoCliente (out int id; out tipo p; out bool hay)
{ while (empty(espera) and (tomados < N))      // ← AND: dos motivos para dejar de esperar
     wait(hayCliente);

  if (tomados == N) { hay = false;  return; }  // 3. BANDERA de salida

  pop(espera, id, p);
  tomados++;
  if (tomados == N) signalall(hayCliente);     // 2. DESPERTAR a los dormidos
  signal(esperaServidor[id]);
  hay = true;
}

Process Servidor [s: 1..E]
{ hay = true;
  while (hay) {
     Servicio.proximoCliente(c, p, hay);
     if (hay) { r = procesar(p);  Servicio.entregar(c, r); } }
}                                              // TERMINA
```

- **Contar al SACAR de la cola, no al entregar el resultado.** Si contás las entregas, un servidor puede quedar dormido sin que nadie lo despierte.
- **`signalall`, no `signal`:** a todos se les acabó el trabajo al mismo tiempo, y el que despertara con `signal` ya no vuelve a entrar a despertar a los demás.
- **Olvidarse del `signalall` es silencioso:** el programa anda bien y no termina.

---

## 6 — Barrera / formación de grupos

*"una vez que todos llegaron, comienzan" · "cuando el equipo está completo"*

```
monitor Barrera {
   int llegaron = 0;
   cond esperar;

   procedure llegue ()
   { llegaron++;
     if (llegaron == N) signalall(esperar);     // el último libera a todos
     else wait(esperar); }                      // PELADO: un solo signalall garantizado
}
```

**Por grupos:** `int llegaron[G]`, `cond esperar[G]`, y todo indexado por grupo — el "todos" es relativo al grupo, no al total.

---

## 7 — Rendezvous (dos procesos que se esperan por etapas)

*La firma: pares cruzados `signal(X); wait(Y);` en los dos lados.*

```
procedure corte_de_pelo() {                procedure proximo_cliente() {
    while (peluquero == 0)                     peluquero++;
       wait(peluquero_disponible);             signal(peluquero_disponible);
    peluquero--;                               wait(silla_ocupada); }       // PELADO
    signal(silla_ocupada);
    wait(puerta_abierta);        // PELADO  procedure corte_terminado() {
    signal(salio_cliente); }                   signal(puerta_abierta);
                                               wait(salio_cliente); }       // PELADO
```

---

## ⚠️ Dato de teórico: bloquearse adentro de un monitor

> **El único mecanismo que SUELTA el monitor al bloquearte es `wait`.**

- **Monitor dentro de monitor:** si hacés `wait` adentro de B estando en A, **B se libera pero A queda tomado con vos adentro** → el que podía despertarte necesita A. *Nested monitor problem.*
- **Semáforos adentro de un monitor:** `P` **no suelta el monitor** → mismo deadlock. Y un `mutex` adentro sobra: la EM es implícita.

---

## ✅ Checklist de entrega (30 segundos)

```
1. Sólo los NOMBRES      ¿alguno significa dos cosas? ¿índices correctos?
                         ¿el id del servidor donde va el del cliente?
2. Sólo los wait         por cada uno: ¿después modifico estado?
                         SÍ → while · NO → if · rendezvous → pelado
3. Sólo las LLAMADAS     ¿cada procedure recibe los parámetros que declara?
4. Sólo los while(true)  ¿está en el PROCESO y no en el monitor?
                         ¿el enunciado pide que termine?
5. Sólo las OPS LARGAS   procesar/usar/vacunar: ¿AFUERA del monitor?
```
