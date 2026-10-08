# Teorías 5 y 6 — Pasaje de Mensajes (PMA y PMS)

> **Fuentes:** Teoría 5 (PMA), Teoría 6 (PMS), Explicación 4 de PMA y de PMS, y las consideraciones del enunciado de la Práctica 4.
> **Idea de este repaso:** no transcribir filminas. Ver **qué cambia respecto de memoria compartida** y dejar los **moldes base** listos para aplicar.

---

## 0. El cambio de cabeza: no hay memoria compartida

En semáforos y monitores, los procesos se comunicaban **escribiendo variables que otros leían**. Acá **no existen las variables compartidas**: cada proceso tiene sólo sus variables locales, y la única forma de hablar con otro es **mandarle un mensaje**.

Consecuencias directas:

| En memoria compartida… | En pasaje de mensajes… |
|---|---|
| `mutex`, `P(mutex)`, el monitor | **Desaparecen.** Sin variables compartidas no hay nada que proteger |
| Una variable compartida (`cola`, `libre`, `cant`) | Variable **local de un proceso servidor** que es el único que la toca |
| `wait(cond)` / `P(espera[id])` | `receive respuesta[id](...)`: me quedo esperando un mensaje para mí |
| `signal(cond)` / `V(espera[id])` | `send respuesta[id](...)`: le mando el mensaje al que espera |
| Coordinador "opcional" | **Casi siempre hay un servidor/administrador**, porque alguien tiene que guardar el estado |

> **Esto es una buena noticia:** el molde del **coordinador**, el que más trabajaste, pasa a ser **la forma natural de resolver casi todo**. Lo que en monitores era un `Procedure` acá es un **pedido por canal** a un proceso servidor.

### La traducción monitor ↔ servidor (Teoría 5, filmina 25)

| Monitor | Proceso servidor con PM |
|---|---|
| Variables permanentes | Variables locales del servidor |
| Llamar a un procedure | `send pedido(...)` + `receive respuesta[id](...)` |
| Entrar al monitor | El servidor hace `receive pedido(...)` |
| Retornar del procedure | El servidor hace `send respuesta[id](...)` |
| `wait(c)` | **Guardar el pedido en una cola de pendientes** (no se responde todavía) |
| `signal(c)` | **Sacar un pendiente de la cola y responderle** |

---

## 1. Los dos mecanismos lado a lado

| | **PMA** (asincrónico) | **PMS** (sincrónico) |
|---|---|---|
| Canales | **Se declaran**, globales: `chan pedidos(int, texto);` | **No se declaran.** Se nombra al proceso |
| Quién usa el canal | Cualquiera envía o recibe (mailbox) | Punto a punto: 1 emisor, 1 receptor |
| Envío | `send canal(datos)` **no bloquea** | `Destino!port(datos)` **bloquea** hasta que el otro reciba |
| Recepción | `receive canal(vars)` bloquea si está vacío | `Origen?port(vars)` bloquea |
| ¿Hay cola? | **Sí: el canal ES una cola FIFO** | **No.** Si querés cola, hay que programarla en un proceso |
| `empty(canal)` | **Sí**, con cuidado | **No existe** |
| Recibir de cualquiera | Un canal compartido ya recibe de todos | Comodín `Origen[*]?port(...)`, **sin orden** |
| Elegir entre varias cosas | `if`/`do` no determinístico con condiciones sobre `empty` y locales | **Comunicación guardada**: `cond; Origen?port(...) -> acciones` |
| Concurrencia | Mayor: el emisor sigue de largo | Menor: el emisor espera |
| Riesgo típico | Busy waiting con `empty` | **Deadlock** si un `!` y un `?` no se emparejan |

### Reglas de la cátedra que se corrigen

**PMA:**
- Los canales son compartidos por todos y son FIFO.
- `send` no bloquea; `receive` sí.
- `empty` sí, pero **no se puede preguntar cuántos mensajes hay**.
- `if`/`do` no determinístico: elige al azar una opción verdadera; si ninguna lo es, sigue de largo. **Ojo: eso puede generar busy waiting** (permitido, pero hay que evitarlo si se puede).
- El tiempo va con `delay`.

**PMS:**
- No se declaran canales, no hay `empty`.
- Envío y recepción bloquean los dos.
- El comodín `[*]` **sólo en la recepción**. Para enviar hay que saber **a quién**, así que el id viaja en el mensaje.
- **En la práctica, las guardas sólo tienen recepciones** (`?`), nunca envíos (`!`). Las condiciones de la guarda sólo miran **variables locales**.
- Una guarda puede estar en tres estados:
  - **Elegible:** la condición es verdadera y el mensaje ya está esperando.
  - **No elegible:** la condición es falsa.
  - **Bloqueada:** la condición es verdadera pero el emisor todavía no llegó.

  En un `if`, si todas son no elegibles se sale sin hacer nada; si alguna está bloqueada, **espera**. El `do` repite hasta que todas sean no elegibles.

---

## 2. Moldes de PMA

Todos salen de la **Explicación 4 de PMA**, que es una escalera: cada ejercicio agrega una sola cosa al anterior. Aprendé la escalera y tenés los moldes.

### A1 — Dejar algo y seguir (sin respuesta)
**Se reconoce:** *"genera un reporte para que el empleado lo corrija"* y **no espera respuesta**.

```
chan Reportes(texto);

Process Persona[id: 0..N-1] {
  texto R;
  while (true) { R = generarReporte(); send Reportes(R); }
}
Process Empleado {
  texto Rep;
  while (true) { receive Reportes(Rep); resolver(Rep); }
}
```
El canal **ya es la cola por orden de llegada**. No hace falta nada más.

### A2 — Cliente / Servidor con respuesta ⭐ (el molde central)
**Se reconoce:** *"…y espera la respuesta"*, *"le entrega un comprobante"*.

```
chan Pedidos(int, texto);        // UN canal general: el id viaja en el mensaje
chan Respuestas[N](texto);       // UN canal PRIVADO por cliente

Process Cliente[id: 0..N-1] {
  texto R, Res;
  send Pedidos(id, R);
  receive Respuestas[id](Res);   // espero MI respuesta
}
Process Servidor {
  texto Rep, Res; int idC;
  while (true) {
    receive Pedidos(idC, Rep);
    Res = resolver(Rep);
    send Respuestas[idC](Res);
  }
}
```
**Los dos errores que la explicación muestra a propósito:**
1. Usar **un solo canal de respuestas para todos**: un cliente se lleva la respuesta de otro.
2. **No mandar el `id` en el pedido**: el servidor no sabe a quién contestarle.

> Es **el coordinador de semáforos**, sin mutex ni semáforos: `Pedidos` es la cola + `hayPedido`, y `Respuestas[id]` es el `sem atendido[id]` + el dato.

### A3 — Varios servidores idénticos
**Se reconoce:** *"alguno de los 3 empleados"*.

**No cambia nada:** `Process Empleado[id: 0..2]` con el mismo código. Todos leen del **mismo** canal `Pedidos`.
Un canal por empleado **sólo** si el cliente elige a un empleado en particular.

### A4 — El servidor hace otra cosa si no hay pedidos (UN servidor)
**Se reconoce:** *"si no hay clientes, realiza tareas administrativas 15 min"*.

```
while (true) {
  if (not empty(Reportes)) { receive Reportes(Rep); resolver(Rep); }
  else delay(15 min);
}
```
**Sólo vale si hay un único receptor de ese canal.**

### A5 — Varios servidores que hacen otra cosa → Coordinador intermedio
**Se reconoce:** A4 + *"2 empleados"* o *"3 vendedores"*.

**Por qué A4 falla con varios:** dos empleados ven `not empty` con **un solo** mensaje; uno lo saca y el otro **se queda bloqueado en el `receive`** cuando debería estar trabajando en otra cosa. Eso es **demora innecesaria**.

**Arreglo:** un **Coordinador** es el único que lee `Reportes`, y **siempre responde**, haya o no pedido:

```
chan Reportes(texto);
chan Pedido(int);                // el empleado pide trabajo
chan Siguiente[3](texto);        // canal privado por empleado

Process Empleado[id: 0..2] {
  texto Rep;
  while (true) {
    send Pedido(id);
    receive Siguiente[id](Rep);
    if (Rep <> "VACIO") resolver(Rep);
    else delay(600);
  }
}
Process Coordinador {
  texto Rep; int idE;
  while (true) {
    receive Pedido(idE);
    if (empty(Reportes)) Rep = "VACIO";
    else receive Reportes(Rep);
    send Siguiente[idE](Rep);    // SIEMPRE responde
  }
}
```

### A6 — Prioridad
**Se reconoce:** *"prioridad a los urgentes"*, *"el director tiene prioridad"*, *"prioridad a los que terminaron"*.

**Un canal por tipo** + `if` no determinístico, con la condición de la opción menos prioritaria **reforzada** con `empty(urgentes)`, + un canal `Aviso` para no hacer busy waiting:

```
chan ReportesU(int, texto);
chan ReportesN(int, texto);
chan Respuestas[N](texto);
chan Aviso(int);                 // "hay algo": evita el busy waiting

Process Persona[id: 0..N-1] {
  if (esUrgente) send ReportesU(id, R) else send ReportesN(id, R);
  send Aviso(1);                 // DESPUÉS del pedido
  receive Respuestas[id](Res);
}
Process Empleado {
  int ok, idP;
  while (true) {
    receive Aviso(ok);           // duermo hasta que haya ALGO
    if (not empty(ReportesU)) ->
          receive ReportesU(idP, Rep); Res = resolverUrg(Rep);
    [] (not empty(ReportesN)) and (empty(ReportesU)) ->
          receive ReportesN(idP, Rep); Res = resolverNormal(Rep);
    fi
    send Respuestas[idP](Res);
  }
}
```
- Sin el `and empty(ReportesU)`, el `if` elige **al azar** y no hay prioridad.
- Sin el `Aviso`, el `while` da vueltas con todo vacío: **busy waiting**.
- `Aviso` funciona como un **semáforo contador**: un mensaje por pedido.

### A7 — Administrador de un recurso con N unidades (= monitor activo)
**Se reconoce:** *"3 impresoras"*, *"10 cabinas"*, *"5 cajas"*, recursos que se piden y se devuelven.

```
chan pedido(int idC);
chan liberar(int idU);
chan respuesta[N](int idU);

Process Admin {
  int disponible = MAX; set unidades;
  while (true) {
    if (not empty(pedido)) and (disponible > 0) ->
          receive pedido(idC); disponible--; remove(unidades, idU);
          send respuesta[idC](idU);
    [] (not empty(liberar)) ->
          receive liberar(idU); disponible++; insert(unidades, idU);
    fi
  }
}
Process Cliente[id: 0..N-1] {
  send pedido(id); receive respuesta[id](idU);
  // usa la unidad idU
  send liberar(idU);
}
```
Es **el pozo de recursos** de semáforos (contador + cola de identidades), pero el contador vive en el servidor.
Para respetar **orden de llegada** con espera: el servidor guarda los pedidos que no puede atender en una `cola pendientes`, y al liberar hace `pop` + `send` (versión de la filmina 24). Ese es el **"el recurso se entrega, no se libera"** de siempre.

### A8 — Continuidad conversacional
**Se reconoce:** *"cuando un docente lo atiende, le hace consultas hasta que no le queden dudas"*.

1. El cliente pide atención por un canal **general** (`send atencion(id)`).
2. Algún servidor libre lo toma y le responde **quién es** (`send rtaAtencion[id](d)`).
3. De ahí en más hablan por **canales privados**: `consulta[d]` y `respuesta[id]`, hasta un `"FIN"`.

### A9 — Barrera en PMA
**Se reconoce:** *"no empiezan hasta que todos hayan llegado"*.
Todos hacen `send llegue(id)`; un coordinador hace **N `receive`** y después **N `send`** a `empezar[i]`. Es la barrera con coordinador de semáforos.

---

## 3. Moldes de PMS

La gran diferencia: **no hay cola implícita**. Si querés orden de llegada, buffer o no bloquear al emisor, **lo tenés que programar en un proceso Admin**. La Explicación 4 de PMS es otra escalera.

### S1 — Cliente / Servidor directo
```
Process Cliente[id: 0..N-1] {
  Servidor!pedido(id, datos);
  Servidor?respuesta(res);
}
Process Servidor {
  int idC;
  while (true) {
    Cliente[*]?pedido(idC, datos);   // de CUALQUIERA
    res = resolver(datos);
    Cliente[idC]!respuesta(res);     // al que me pidió: el id vino en el mensaje
  }
}
```
**Ojo: `[*]` NO respeta el orden de llegada.** Elige al azar entre los que están esperando. Si el enunciado dice "sin importar el orden", esto alcanza.

### S2 — Admin con cola ⭐ (orden de llegada y/o no bloquear al emisor)
**Se reconoce:** *"en orden de llegada"* o *"no debe esperar"* / *"maximizar la concurrencia"* con PMS.

```
Process Admin {
  cola Fila; texto R; int idC;
  do Cliente[*]?pedido(idC, R) -> push(Fila, (idC, R));
  [] not empty(Fila); Servidor?listo() -> pop(Fila, (idC, R));
                                          Servidor!trabajo(idC, R);
  od
}
Process Servidor {
  while (true) {
    Admin!listo();                  // "estoy libre, dame uno"
    Admin?trabajo(idC, R);
    res = resolver(R);
    Cliente[idC]!respuesta(res);
  }
}
Process Cliente[id: 0..N-1] {
  Admin!pedido(id, R);
  Servidor?respuesta(res);
}
```
**Las tres claves:**
1. **El servidor PIDE** (`Admin!listo()`) en vez de que el Admin le envíe. **Un envío no puede ir en una guarda**, así que el Admin espera a que le pidan y recién ahí envía.
2. **`not empty(Fila)` en la guarda** del pedido. Sin eso, el Admin acepta el pedido y después no tiene qué darle. `Fila` es **local** del Admin, por eso se puede consultar.
3. El Admin es **la cola de la terminal de micros**: guarda en orden y reparte cuando hay alguien libre.

### S3 — Varios servidores con Admin
Igual que S2, pero el servidor **manda su id** (`Admin!listo(id)`) y el Admin recibe de cualquiera y contesta a ese:
```
[] not empty(Fila); Servidor[*]?listo(idS) -> pop(Fila, (idC, R));
                                               Servidor[idS]!trabajo(idC, R);
```
El cliente recibe la respuesta con `Servidor[*]?respuesta(res)`, porque no sabe cuál lo atendió.

### S4 — Exclusión mutua de un recurso con empleado (simulador, máquina)
**Se reconoce:** *"el empleado las deja acceder de a una"*.

| Variante | Cómo |
|---|---|
| Sin orden | `Persona[*]?pido(id)` → `Persona[id]!pasa()` → `Persona[id]?termine()` |
| **Orden por id** | `for i = 0..P-1`: `Persona[i]?pido()` → `Persona[i]!pasa()` → `Persona[i]?termine()`. Es **la posta**, pero la lleva el empleado |
| **Orden de llegada** | Un Admin con cola (S2) entre las personas y el empleado |

### S5 — Barrera en PMS
`for i: Persona[*]?llegue()` N veces, y después `for i: Persona[i]!empiecen()`.

### El deadlock típico de PMS
Si **dos procesos hacen `!` al mismo tiempo**, uno al otro, y recién después `?`, los dos quedan bloqueados en el envío. **Siempre chequeá que cada `!` tenga su `?` del otro lado, en el orden correcto.**

---

## 4. Cómo reconocer el molde (machete)

| Si el enunciado dice… | PMA | PMS |
|---|---|---|
| *"deja X y sigue"* (sin respuesta) | **A1**: un canal | Admin buffer (**S2**) si no debe esperar |
| *"espera la respuesta"*, *"le entrega un comprobante"* | **A2**: `Pedidos` general + `Respuestas[id]` | **S1** |
| *"en orden de llegada"* | **Gratis:** el canal es FIFO | **S2**: Admin con cola, **`[*]` no alcanza** |
| *"alguno de los K empleados"* | **A3**: mismo canal, K procesos | **S3**: Admin + `Servidor[*]?listo(idS)` |
| *"si no hay clientes, hace otra cosa"* | **A4** si es 1; **A5** (coordinador) si son varios | Guarda que no se bloquee / Admin que responde "VACÍO" |
| *"prioridad"* | **A6**: dos canales + `and empty(U)` + `Aviso` | Guardas ordenando con variables locales del Admin |
| *"N unidades de un recurso"* | **A7**: Admin con contador + cola de pendientes | `do disponible > 0; Cliente[*]?acquire(...)` |
| *"de a uno, según su id"* | Posta con canales privados | **S4** con `for i` |
| *"no empiezan hasta que lleguen todos"* | **A9** | **S5** |
| *"lo atiende y le hace varias consultas"* | **A8** | Ídem con canales implícitos |

**Las 5 reglas que valen siempre:**
1. **El id viaja en el mensaje** cuando el que recibe tiene que contestar.
2. **Respuesta por canal privado** (`Respuestas[id]` en PMA; `Cliente[id]!` en PMS).
3. **`empty` con varios receptores genera demora innecesaria** → poner un coordinador.
4. **En PMS no hay cola**: si hace falta orden o buffer, va un **Admin**, y el que consume **pide** (`Admin!listo()`).
5. **La operación larga va afuera**: el servidor responde y recién después el cliente usa el recurso.

---

## 5. Los ejercicios de la Práctica 4 según su molde

| Ejercicio | Molde |
|---|---|
| PMA 1 (banco, a/b/c) | **A2** → **A3** → **A5** (el (c) es justo la escalera de la explicación) |
| PMA 2 (5 cajas, la de menos gente) | **A7** + A2: coordinador que elige caja + una cola por caja. Es la **terminal de micros / planta verificadora** |
| PMA 3 (comida rápida: vendedores, cocineros) | **A5** (vendedores que reponen bebidas) + **A1** vendedor→cocinero + respuesta directa cocinero→cliente |
| PMA 4 (locutorio, 10 cabinas) | **A7** + A2; el (b) es **A6** (prioridad a los que pagan) |
| PMA 5 (3 impresoras, director) | **A1**/**A3** (impresoras leen del mismo canal); (b) **A6**; (c)–(e) terminación + evitar busy waiting |
| PMS 1 (antivirus) | (b) **S1**; (c) **S2**; (d) S2 como buffer |
| PMS 2 (laboratorio, 3 empleados en cadena) | **S2** como buffer entre el 1º y el 2º; el 2º y el 3º es **S1** |
| PMS 3 (examen, N alumnos, P profesores) | (a) **S2**; (b) **S3**; (c) + **S5** |
| PMS 4 (simulador de vuelo) | **S4**: (a) sin orden; (b) por id; (c) orden de llegada con Admin |
| PMS 5 (máquina expendedora) | **S4**(c): orden de llegada con Admin |

---

## 6. Puente a la Teoría 7: unidireccional o bidireccional

> **Aclaración del profe:** la diferencia entre PMA/PMS y RPC/Rendezvous es que **la comunicación es bidireccional**.

### PMA y PMS: unidireccional
Cada mensaje va **en un solo sentido**. Para que el cliente reciba una respuesta hace falta **un segundo mensaje en sentido contrario**. Por eso el molde A2 lleva **dos canales**:

```
send Pedidos(id, datos);          // ida
receive Respuestas[id](res);      // vuelta, por OTRO canal, privado
```

La Teoría 7 lo marca como la incomodidad de pasaje de mensajes para cliente/servidor: *"la comunicación bidireccional obliga a especificar 2 tipos de canales (requerimientos y respuestas). Además, cada cliente necesita un canal de reply distinto"*. Son justo los dos errores del molde A2.

### RPC y Rendezvous: bidireccional
**Una sola operación hace la ida y la vuelta**, como llamar a un procedimiento. El llamador manda los parámetros de entrada y **queda bloqueado hasta recibir los resultados**:

```
call Servidor.atender(datos, res);   // ida y vuelta en una sola llamada
```

Desaparecen el canal de respuestas, el `Respuestas[id]` y el id dentro del mensaje. **La interfaz es como la de un monitor**, pero entre procesos distribuidos.

### RPC o Rendezvous: quién atiende la llamada

| | RPC | Rendezvous (Ada) |
|---|---|---|
| Quién atiende | Se crea **un proceso nuevo por cada llamada**, que ejecuta el procedure | **Un proceso que ya existe** la acepta con `accept` |
| Servidor | **Pasivo**, como un monitor. Si hace falta sincronizar, va adentro | **Activo**: decide **cuándo** y **qué** atiende (`select when cond => accept ...`) |

> El `select when B => accept` de Ada es **la comunicación guardada de PMS** (`B; Origen?port -> S`), pero con ida y vuelta. Lo que entrenes con guardas en PMS sirve directo para la Práctica 5.
