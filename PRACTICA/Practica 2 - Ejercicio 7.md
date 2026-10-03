# Práctica 2 — Ejercicio 7

> **Enunciado.** Se tiene un curso con **50 alumnos**. Cada alumno debe realizar una tarea y existen **10 enunciados posibles**. Una vez que **todos los alumnos eligieron** su tarea, comienzan a realizarla. Cada vez que un alumno termina su tarea, **le avisa al profesor** y se queda esperando el **puntaje del grupo** (depende de todos aquellos que comparten el mismo enunciado). Cuando un grupo terminó, el profesor les otorga un puntaje que representa **el orden en que se terminó esa tarea** de las 10 posibles.
>
> *Nota: para elegir la tarea suponga que existe una función `elegir` que le asigna una tarea a un alumno (esta función asignará 10 tareas diferentes entre 50 alumnos, es decir, que 5 alumnos tendrán la tarea 1, otros 5 la tarea 2 y así sucesivamente para las 10 tareas).*

---

## 0. La nota del final no es decoración

Antes de diseñar nada, leé la nota. Sin ella el problema es otro:

> Si `elegir()` repartiera de cualquier manera, **no sabrías el tamaño de ningún grupo hasta que eligieran los 50**, y entonces no podrías detectar "este grupo terminó" hasta tener todas las tareas hechas.

La nota garantiza **exactamente 5 alumnos por tarea**. Y eso cambia el diseño entero:

- Sabés de antemano que un grupo está completo cuando contaste **5**.
- **El profesor no espera a nadie más:** apenas los 5 de un grupo avisaron, ese grupo cobra su nota, aunque los otros 45 sigan trabajando.

> **La regla general:** en estos enunciados, las "Notas" del final casi siempre están **sacándote un problema de encima**. Leelas antes de diseñar, no después.

---

## 1. Las tres esperas

El ejercicio se resuelve enumerando dónde alguien tiene que esperar. Hay **tres**, y una tentación que hay que descartar:

| # | Quién espera | Por qué | Patrón |
|---|---|---|---|
| 1 | **Cada alumno**, después de elegir | Hasta que **todos** hayan elegido | **Barrera** |
| 2 | **El profesor** | Hasta que **alguien** termine su tarea | Productor/consumidor |
| 3 | **Cada alumno**, después de avisar | Hasta que **su grupo** tenga nota | Traspaso con coordinador |

**La espera falsa:** `delay(tarea)`, la realización de la tarea. **No lleva ninguna sincronización** — cada alumno trabaja sobre lo suyo y no comparte nada con nadie mientras tanto. Descartar una espera falsa vale tanto como encontrar las verdaderas: mucha gente mete un mutex ahí sin necesidad.

### El profesor es un proceso, no un semáforo

*"El profesor les otorga un puntaje"* → **decide** algo, no se limita a ser usado. Eso lo convierte en un **proceso**.

---

## 2. Las dos decisiones de diseño

### (a) ¿Quién cuenta los 5 de cada grupo?

Las dos opciones funcionan; hay que elegir **una** y sostenerla:

| | **A — cuentan los alumnos** | **B — cuenta el profesor** *(la que va acá)* |
|---|---|---|
| Quién lleva `terminados[10]` | Los alumnos, compartido | **El profesor** |
| Quién avisa | Sólo el 5º de cada grupo | **Los 50**, cada uno al terminar |
| Avisos que recibe el profesor | 10 | **50** |
| ¿El contador lleva mutex? | **Sí** (5 escritores por grupo) | **No** (único escritor) |

**Se elige B**, y el premio es concreto: `terminados[]` tiene un solo dueño, así que **no lleva candado**.

El costo es que el profesor recibe 50 avisos y **cada aviso tiene que decirle de qué grupo es** — y ahí aparece la segunda decisión.

### (b) ¿Cómo viaja el "de qué grupo soy"?

> **Un semáforo transmite un instante, no un dato.** `V(s)` dice *"ya"*; no puede decir *"ya, y soy del grupo 3"*.

Entonces el dato viaja aparte, y el semáforo sólo sincroniza quién escribe y quién lee. Son **50 avisando y uno consumiendo**, así que es un productor/consumidor sobre una cola:

```
// EL QUE AVISA                        // EL QUE RECIBE
P(mutexAviso);                         P(yaAvisado);         // espero la SEÑAL
  push(avisos, mi_grupo);  // DATO     P(mutexAviso);
V(mutexAviso);                           pop(avisos, grupo); // recojo el DATO
V(yaAvisado);              // SEÑAL    V(mutexAviso);
```

El candado hace falta porque **50 procesos escriben** la cola. No hay semáforo de "lugares libres" porque cada alumno hace un solo `push`: la cola nunca puede desbordar.

**Y el mismo mecanismo, en la otra dirección**, para que el profesor le pase la nota a los 5 del grupo — sólo que ahí es **uno escribiendo y 5 leyendo**, cada grupo su casillero:

```
// EL PROFESOR                         // EL ALUMNO
puntaje[grupo] = nota;     // DATO     P(grupos[mi_grupo]);        // espero
nota++;                                mi_nota = puntaje[mi_grupo]; // leo el DATO
for j = 1 to 5
   V(grupos[grupo]);       // SEÑAL
```

**El dato se escribe SIEMPRE antes de la señal.** Al revés, el alumno se despierta y lee un casillero vacío.

---

## 3. Solución

```
// PRECONDICIONES: 50 alumnos, 10 tareas
//                 elegir() reparte exactamente 5 alumnos por tarea (lo garantiza la nota)
//                 todas las variables compartidas inicializadas antes de crear los procesos

int contador = 0;                 // cuántos alumnos ya eligieron
sem mutexBarrera = 1;             // candado del contador  → arranca en 1
sem barrera = 0;                  // señal de la barrera   → arranca en 0

cola avisos;                      // grupos que avisaron que un miembro terminó
sem mutexAviso = 1;               // candado de la cola (50 escritores)
sem yaAvisado = 0;                // señal: "hay un aviso nuevo"

sem grupos[10] = ([10] 0);        // señal: "tu grupo ya tiene nota"
int puntaje[10];                  // el DATO: la nota de cada grupo


Process Alumno [id: 1..50]
{ int mi_grupo, mi_nota;                      // LOCALES
  bool ultimo;                                // LOCAL

  // ── BARRERA: nadie arranca hasta que todos eligieron ──
  mi_grupo = elegir();
  P(mutexBarrera);
    contador++;
    ultimo = (contador == 50);                // la decisión se toma ADENTRO
  V(mutexBarrera);
  if (not ultimo) P(barrera);                 // los 49 primeros esperan
  V(barrera);                                 // cascada: cada uno abre al siguiente

  // ── TRABAJO: sin sincronización ──
  delay(tarea);

  // ── AVISO AL PROFESOR ──
  P(mutexAviso);
    push(avisos, mi_grupo);                   // el DATO
  V(mutexAviso);
  V(yaAvisado);                               // la SEÑAL

  // ── ESPERO LA NOTA DE MI GRUPO ──
  P(grupos[mi_grupo]);
  mi_nota = puntaje[mi_grupo];                // leo el DATO
}                                             // sin while(true): cada alumno hace una tarea


Process Profesor
{ int terminados[10] = ([10] 0);              // LOCAL: único escritor ⇒ sin candado
  int nota = 1;                               // el puntaje que le toca al próximo grupo
  int grupo, i, j;

  for i = 1 to 50 {                           // exactamente 50 avisos, y me retiro
     P(yaAvisado);                            // espero que alguien termine
     P(mutexAviso);
       pop(avisos, grupo);                    // ¿de qué grupo es?
     V(mutexAviso);

     terminados[grupo]++;
     if (terminados[grupo] == 5) {            // ese grupo está completo
        puntaje[grupo] = nota;                // el DATO primero
        nota++;
        for j = 1 to 5                        // ← j, NO i (ver errores)
           V(grupos[grupo]);                  // la SEÑAL después: despierto a los 5
     }
  }
}
```

---

## 4. La barrera, pieza por pieza

Es la primera vez que aparece en la práctica, así que vale desarmarla.

```
P(mutexBarrera);
  contador++;
  ultimo = (contador == 50);      // ← LOCAL, y decidido ADENTRO del candado
V(mutexBarrera);
if (not ultimo) P(barrera);
V(barrera);
```

| Línea | Por qué |
|---|---|
| `contador++` adentro del candado | 50 escritores |
| `ultimo` calculado **adentro** y guardado en una **local** | Si consultaras `contador` **afuera**, estarías leyendo sin proteger una variable con 50 escritores. `ultimo` es tuya y nadie te la puede pisar |
| `if (not ultimo) P(barrera)` | Los 49 primeros esperan. El último no |
| `V(barrera)` que ejecutan **todos** | Es la **cascada**: el último despierta a uno, ese despierta al siguiente, y así hasta que pasan los 50 |

### Los dos idiomas de barrera

```
// A — cascada (la de arriba)          // B — el último libera a todos
if (not ultimo) P(barrera);            if (ultimo)
V(barrera);                               { for j = 1 to 49:  V(barrera); }
                                       else P(barrera);
```

| | A | B |
|---|---|---|
| Permisos que sobran al final | **1** | 0 |
| Reusable dentro de un loop | No, sin limpiar antes | Sí |
| Cuánto hay que explicar | El permiso que sobra | Nada: 49 `P` contra 49 `V` |

Las dos son correctas para un uso único. **Para el parcial conviene la B**: las cuentas cierran exactas y es trivial de defender.

---

## 5. Justificación (esto es lo que suma puntos)

> Se usa una **barrera** porque nadie empieza a trabajar hasta que todos eligieron: cada alumno incrementa un contador protegido por `mutexBarrera` y decide **dentro** de la sección crítica si es el último, guardándolo en una variable local para no leer sin protección una variable con 50 escritores; los primeros 49 se bloquean en `barrera` y el último inicia la cascada de liberación. La realización de la tarea no requiere sincronización alguna: cada alumno trabaja sobre datos propios. El aviso al profesor es un **productor/consumidor**: los 50 alumnos depositan su grupo en la cola `avisos`, protegida por `mutexAviso` porque hay 50 escritores, y señalizan con `V(yaAvisado)`; el profesor consume de a uno. No hace falta un semáforo de lugares libres porque cada alumno hace un único `push` y la cola nunca desborda. `terminados[]` **no lleva candado** porque su único escritor es el profesor —consecuencia de haber elegido que la cuenta la lleve él—. Cuando el contador de un grupo llega a 5, ese grupo terminó y recibe el valor corriente de `nota`, que luego se incrementa: así el puntaje refleja el orden en que se completaron las tareas, y el profesor no necesita esperar a los demás grupos. Como un semáforo transmite un instante y no un dato, el puntaje viaja por el arreglo compartido `puntaje[]`, escrito **antes** de las señales; tampoco requiere exclusión mutua porque tiene un único escritor y sus lectores quedan ordenados por el propio semáforo. Se emiten 5 `V` sobre `grupos[k]` porque son exactamente 5 los alumnos demorados de cada grupo, y se usa un semáforo por grupo en vez de uno global para no despertar a alumnos de un grupo que todavía no terminó. El profesor itera exactamente 50 veces y se retira.

---

## 6. Los errores que se penalizan

**(1) Loops anidados con la misma variable.**
```
for i = 1 to 50 {
    ...
    for i = 1 to 5 { V(grupos[grupo]); }     ← ✘ pisa la i de afuera
}
```
El loop interno deja `i` en 5 y el externo sigue contando desde ahí: el profesor da **muchas más de 50 vueltas** y se cuelga para siempre en `P(yaAvisado)` esperando avisos que ya no van a llegar. **El programa no termina.** Con 50 y 5 se ve; con nombres parecidos, se escapa.

**(2) Un solo `V(grupos[grupo])`.** Son **5** alumnos demorados en ese semáforo. Con un `V` despertás a uno y los otros cuatro quedan dormidos para siempre.

**(3) Un semáforo global en vez de `grupos[10]`.** Si hacés `V(nota)` sobre un semáforo compartido por los 50, despertás a *alguno* — perfectamente uno del grupo 7, que todavía no terminó. Los semáforos **no son FIFO** y no podés dirigir a quién despiertan.

**(4) `grupos[10]` inicializado en 1.** Es una **señal** (el `P` lo hace el alumno, el `V` el profesor) ⇒ arranca en **0**. Con 1, el primer alumno de cada grupo pasa de largo sin esperar y "recibe" una nota que todavía no se calculó; los otros cuatro quedan colgados.

**(5) El profesor sin loop.** Atiende un aviso y termina; los otros 49 alumnos quedan esperando para siempre.

**(6) Consultar `contador` fuera del candado.**
```
V(mutexBarrera);
if (contador < 50) P(barrera);      ← ✘ lectura sin proteger, 50 escritores
```
Entre que soltás el candado y llegás al `if`, otro alumno puede incrementarlo. No se cuelga nadie, pero es una carrera y se penaliza.

**(7) `V(grupos[...])` antes de escribir `puntaje[...]`.** El alumno se despierta y lee un casillero vacío. **El dato primero, la señal después.**

**(8) Mutex sobre `terminados[]` o `puntaje[]`.** No los necesitan: un único escritor cada uno. Ponerlos no rompe nada pero muestra que no aplicaste la regla de contar escritores.

---

## 7. Chequeos de sanidad

- **Contá los avisos:** 50 `push` contra 50 `pop`, y 50 `V(yaAvisado)` contra 50 `P(yaAvisado)`. Cierra.
- **Contá por grupo:** cada grupo recibe exactamente 5 avisos, así que `terminados[k]` llega a 5 **una sola vez**. Nunca se pasa de 5 ni se queda en 4.
- **Contá las notas:** 10 grupos completándose ⇒ `nota` recorre **1, 2, …, 10**. Si te da más o menos de 10, algo está mal.
- **Contá los despertares:** 5 `V` sobre `grupos[k]` contra 5 alumnos haciendo `P(grupos[k])`. Nadie queda dormido y no sobra ningún permiso.
- **Terminación:** el profesor da 50 vueltas; al salir, los 10 grupos ya cobraron. Los 50 alumnos leyeron su nota y terminaron.
- **Sin deadlock:** nadie se bloquea reteniendo un candado. Los `P(barrera)`, `P(yaAvisado)` y `P(grupos[])` se hacen todos con las manos libres.

---

## 8. Los tres chequeos del corrector

1. **¿Está la barrera, y `ultimo` se decide dentro del candado?** Es la mitad del ejercicio y el lugar donde más se lee `contador` sin proteger.
2. **¿El puntaje efectivamente viaja?** Que `nota` se incremente no alcanza: tiene que escribirse en `puntaje[grupo]` **antes** de las señales, y el alumno tiene que leerlo **después** de su `P`. Un semáforo no lleva datos.
3. **¿Se despiertan los 5 del grupo correcto?** Cinco `V`, sobre el semáforo **de ese grupo**, no sobre uno global.
