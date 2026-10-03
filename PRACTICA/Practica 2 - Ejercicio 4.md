# Práctica 2 — Ejercicio 4

> **Enunciado.** Existe una BD que puede ser accedida por **6 usuarios como máximo** al mismo tiempo. Los usuarios se clasifican en **prioridad alta** y **prioridad baja**. Además: no puede haber más de **4** usuarios de prioridad **alta** al mismo tiempo usando la BD, ni más de **5** de prioridad **baja**.
> **Indique si la solución presentada es la más adecuada. Justifique la respuesta.**

```
Var
   total: sem := 6;
   alta:  sem := 4;
   baja:  sem := 5;

Process Usuario-Alta [I: 1..L]::          Process Usuario-Baja [I: 1..K]::
 { P(total);                               { P(total);
     P(alta);                                  P(baja);
     //usa la BD                               //usa la BD
     V(total);                                 V(total);
     V(alta);                                  V(baja);
 }                                         }
```

---

## 0. Qué tipo de ejercicio es

No hay que escribir código: hay que **auditar** uno que ya está. Y ojo con la redacción, porque marca toda la respuesta:

> No pregunta *"¿funciona?"* sino **"¿es la más adecuada?"**

Son dos preguntas distintas. Una solución puede ser perfectamente correcta y aun así ser mala.

---

## 1. La respuesta, como se escribe en el parcial

> **¿Es la más adecuada? No.** La solución es **correcta** —cumple las tres restricciones y no tiene deadlock— pero provoca **demora innecesaria**.
>
> **1. Las restricciones se cumplen.**
> Todo usuario debe pasar `P(total)` y `P(alta)`/`P(baja)` antes de usar la BD, y ambos `V` ocurren **después** del uso. Por lo tanto, en todo momento: usuarios en la BD ≤ 6, altas ≤ 4, bajas ≤ 5.
>
> **2. No hay deadlock.**
> Todos los procesos toman los semáforos en el **mismo orden** (`total`, luego `alta`/`baja`) y ninguno retiene `alta`/`baja` mientras espera `total`. No hay espera circular.
>
> **3. El defecto.**
> Un proceso bloqueado en `P(alta)` **ya consumió una unidad de `total`**: reserva un lugar en la BD sin usarlo.
>
> *Contraejemplo.* 4 altas usando la BD (`alta = 0`, `total = 2`). Llegan 2 altas más: pasan `P(total)` (`total = 0`) y se bloquean en `P(alta)`. Llega un baja: se bloquea en `P(total)`.
>
> En ese estado hay **sólo 4 usuarios usando la BD** y el baja queda rechazado, aunque las tres restricciones lo permitían: 5 ≤ 6 en total, 4 ≤ 4 altas, 1 ≤ 5 bajas. La BD opera a 4 de sus 6 lugares.
>
> **4. Mejora.** Invertir el orden de los dos `P`:
> ```
> P(alta);  P(total);  //usa la BD;  V(total);  V(alta);
> ```
> Así el que no consigue cupo de su tipo se bloquea **sin retener un lugar de la BD**. Se siguen cumpliendo las tres restricciones y sigue sin haber deadlock.

---

## 2. Por qué la respuesta está armada así

| Parte | Por qué está |
|---|---|
| **El veredicto primero** | El ayudante ya sabe adónde vas y lee el resto como respaldo |
| **Puntos 1 y 2, cortos** | Son lo que hay que **descartar** antes de acusar. Sin ellos, "tiene un problema" queda en el aire |
| **Punto 3: el contraejemplo** | **Es** la demostración. Sin números concretos, "puede haber demora innecesaria" es una opinión |
| **Punto 4: el arreglo** | Prueba que ubicaste el mecanismo que falla, no sólo el síntoma |

**Si hay que recortar por espacio, se recorta de 1 y 2 — nunca del contraejemplo.**

### Lo que conviene dejar afuera

- Explicar qué es un semáforo o qué hace `P`. Se da por sabido.
- Repetir el enunciado.
- La discusión sobre el `V(total)` antes del `V(alta)` (ver §4): llama la atención pero **no causa ningún error**, y meterlo desdibuja cuál es el defecto real.
- Adjetivos. *"Muy ineficiente"* no suma: el número —4 de 6 lugares— dice más.

> Usar el nombre técnico —**"demora innecesaria"**— muestra que lo estás encuadrando en las cuatro propiedades del problema de la sección crítica, no improvisando.

---

## 3. La traza completa (el razonamiento detrás del contraejemplo)

Arranque: `total = 6`, `alta = 4`, `baja = 5`. Llegan 6 usuarios de prioridad alta y ninguno termina.

| Quién | Qué hace | `total` | `alta` | Estado |
|---|---|---|---|---|
| Alta 1 | `P(total)` → `P(alta)` | 5 | 3 | **usando la BD** |
| Alta 2 | `P(total)` → `P(alta)` | 4 | 2 | **usando la BD** |
| Alta 3 | `P(total)` → `P(alta)` | 3 | 1 | **usando la BD** |
| Alta 4 | `P(total)` → `P(alta)` | 2 | **0** | **usando la BD** |
| Alta 5 | `P(total)` ✓ → `P(alta)` ✗ | **1** | 0 | 🛑 bloqueado |
| Alta 6 | `P(total)` ✓ → `P(alta)` ✗ | **0** | 0 | 🛑 bloqueado |

Estado: `total = 0`, `alta = 0`, `baja = 5`. **Usando la BD: 4.** Bloqueados reteniendo un lugar: **2**.

Llega un usuario de prioridad baja → `P(total)` con `total = 0` → **bloqueado**. Pero tenía derecho a entrar:

| Restricción | Con el baja adentro | ¿Se cumple? |
|---|---|---|
| Máximo 6 en total | 4 + 1 = **5** | ✅ |
| Máximo 4 de prioridad alta | sigue **4** | ✅ |
| Máximo 5 de prioridad baja | **1** | ✅ |

### No hace falta que lleguen los 6 juntos

El escenario realista es más simple: 4 altas entran y se quedan trabajando un rato largo (usar la BD lleva tiempo, por eso el `delay`); mientras tanto llegan 2 altas más que pasan el `P(total)` y se duermen en el `P(alta)`; después llega un baja. Llegadas escalonadas durante una operación larga: el patrón de tráfico más común que hay.

Y en concurrencia alcanza con exhibir **una** ejecución posible donde el defecto ocurre. No hace falta que pase siempre.

---

## 4. Dos precisiones que evitan errores de diagnóstico

### El `V(total)` antes del `V(alta)` NO viola los límites

Es razonable desconfiar —queda raro— pero no rompe nada, y la razón es **cuándo** ocurren los `V`:

```
P(total);
P(alta);
//usa la BD       ← acá termina de usarla
V(total);         ← los dos V son DESPUÉS del uso
V(alta);
```

Para estar usando la BD hay que haber pasado el `P(alta)`, y el `V(alta)` recién ocurre cuando el proceso ya terminó. Entonces la cantidad de altas que pasaron `P(alta)` sin haber hecho `V(alta)` es **≤ 4** en todo momento, y los que usan la BD son un subconjunto de esos. Igual con `total`.

Lo único que provoca el `V(total)` adelantado es que un usuario nuevo pase el `P(total)` y se duerma en el `P(alta)`. No entra de más: espera en otro lugar.

### El defecto es transitorio, no permanente

Cuando la primera alta termina:

```
V(total);     → total pasa a 1     ← se libera el baja que esperaba
V(alta);      → alta pasa a 1      ← se libera una de las altas bloqueadas
```

El baja **no queda bloqueado para siempre**. El sistema progresa y todos terminan entrando. Por eso el defecto **no es**:

| No es | Por qué |
|---|---|
| ❌ Deadlock | No hay espera circular |
| ❌ Violación de los límites | Nunca se supera 6 / 4 / 5 |
| ❌ Inanición | Todos terminan entrando |

**Es demora innecesaria:** se bloquea a un usuario que las tres restricciones permitían, y la BD opera por debajo de su capacidad mientras tanto. Es la tercera de las cuatro propiedades del problema de la sección crítica.

---

## 5. El criterio general que deja el ejercicio

> Cuando hay que tomar **dos semáforos**, tomá primero **el que más probablemente te bloquee** y que **menos recursos reserve**.
> Bloquearte reteniendo el recurso escaso es siempre el peor orden.

Es la misma regla de *"no te bloquees en un semáforo teniendo otro tomado"*. Acá no produce deadlock —porque el orden de toma es consistente entre todos los procesos— pero produce **desperdicio de capacidad**.
