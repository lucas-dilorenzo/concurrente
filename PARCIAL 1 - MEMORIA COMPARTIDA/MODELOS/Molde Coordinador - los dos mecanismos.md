# El molde del coordinador, en los dos mecanismos

> **El mismo problema resuelto con semáforos y con monitores.** Es el molde más frecuente (~10 de los 28 enunciados de semáforos) y el parcial pide uno de cada mecanismo, así que conviene tenerlo de los dos lados.

**El enunciado:** en un negocio hay **UN empleado** que diseña tarjetas digitales y atiende los pedidos de **C clientes** **por orden de llegada**. El cliente envía las indicaciones, el empleado arma la tarjeta y se la envía. Existe `HacerTarjeta(indicaciones)`. **Todos los procesos deben terminar.**

---

## Antes de escribir: dibujá las flechas

```
Cliente  ──── "llegué, acá van mis indicaciones" ────►  Empleado
Cliente  ◄─────────── "está tu tarjeta" ──────────────  Empleado
```

**Cada flecha es una señal. El origen avisa, el destino espera.** Dos flechas ⇒ dos señales.

Y el corte: **`HacerTarjeta()` ocurre entre las dos flechas**, afuera de todo candado. Por eso cada actor hace **dos llamadas**.

---

## Con SEMÁFOROS

```
// PRE: C > 0

cola pedidos;                    sem mutexCola = 1;        // candado: P y V en el mismo proceso
sem hayPedido = 0;               // Cliente → Empleado   ("hay un pedido")
dato tarjetaDe[C];               // el DATO que vuelve
sem listo[C] = ([C] 0);          // Empleado → Cliente   ("está la tuya")

Process Cliente [id: 1..C]
{ dato indicaciones, miTarjeta;
  indicaciones = cargarIndicaciones();

  P(mutexCola);  push(pedidos, id, indicaciones);  V(mutexCola);
  V(hayPedido);                                    // AVISO yo

  P(listo[id]);                                    // ESPERO yo
  miTarjeta = tarjetaDe[id];
}

Process Empleado
{ int c;  dato ind, r;
  for i = 1 to C {                                 // total conocido → TERMINA
     P(hayPedido);                                 // ESPERO yo
     P(mutexCola);  pop(pedidos, c, ind);  V(mutexCola);

     r = HacerTarjeta(ind);                        // AFUERA de todo candado

     tarjetaDe[c] = r;                             // el DATO primero
     V(listo[c]);                                  // la SEÑAL después
  }
}
```

**La simetría:** cada proceso hace **un `V` y un `P`**, cruzados con el otro. *Si un proceso hace `V` de algo que él mismo necesita, está invertido.*

---

## Con MONITORES

```
monitor Negocio {                      // ← el monitor es el MOSTRADOR, no una persona
   cola pedidos;                       // {id, indicaciones} — FLUJO ⇒ cola
   cond esperandoPedidos;              // una sola: hay un solo empleado

   dato tarjetaDe[C];                  // CASILLERO ⇒ arreglo
   bool listo[C] = ([C] false);        // la MEMORIA
   cond esperandoTarjeta[C];           // timbre con nombre

   procedure dejarPedido (int id; dato ind)                 // CLIENTE
   { push(pedidos, (id, ind));
     signal(esperandoPedidos); }

   procedure proximoPedido (out int c; out dato ind)        // EMPLEADO
   { while (empty(pedidos)) wait(esperandoPedidos);
     pop(pedidos, (c, ind)); }

   procedure entregarTarjeta (int c; dato t)                // EMPLEADO
   { tarjetaDe[c] = t;
     listo[c] = true;
     signal(esperandoTarjeta[c]); }

   procedure retirarTarjeta (int id; out dato t)            // CLIENTE
   { if (not listo[id]) wait(esperandoTarjeta[id]);
     t = tarjetaDe[id]; }
}

Process Cliente [id: 1..C]              Process Empleado
{ Negocio.dejarPedido(id, ind);         { for i = 1 to C {
  Negocio.retirarTarjeta(id, miT); }         Negocio.proximoPedido(c, ind);
                                             t = HacerTarjeta(ind);   // AFUERA
                                             Negocio.entregarTarjeta(c, t); } }
```

---

## Qué cambia al traducir

| Semáforos | Monitores |
|---|---|
| `sem mutexCola = 1` | **desaparece** — la EM es implícita |
| `sem hayPedido = 0` | `cond esperandoPedidos` |
| `sem listo[C] = ([C] 0)` | `cond esperandoTarjeta[C]` **+ `bool listo[C]`** |
| Dos "canales" sueltos | **Cuatro procedures**, dos por actor |

> **Por qué en monitores aparece la bandera y en semáforos no:** el semáforo **tiene memoria** (el `V` queda guardado en su valor, aunque nadie esté esperando). Una `cond` **no**: un `signal` sin nadie esperando **se pierde**. Y como el cliente **sale del monitor** entre sus dos llamadas, el empleado puede avisar justo en ese hueco.
>
> *El booleano recuerda, la `cond` duerme y despierta.*

---

## Flujo vs. casillero — por qué los pedidos van en cola y las tarjetas en arreglo

> **¿Importa quién recibe este elemento concreto?**
> **NO** → **flujo**: cola + señal compartida. *(El empleado saca el que sigue, le da igual de quién sea.)*
> **SÍ** → **casillero**: arreglo + señal indexada + bandera. *(La tarjeta del cliente 3 sólo le sirve al 3.)*

Con una `cola tarjetas` el cliente 7 hace `pop` y **se lleva la del 3**: una cola no tiene destinatario.
Y ni siquiera alcanza con despertar al correcto — si hay dos tarjetas pendientes, el `pop` devuelve la de arriba igual.

**Van de a pares: si el dato va indexado, la señal también.**
