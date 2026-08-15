# j-up

Juego de lógica de números en una grilla, en un solo archivo HTML. Sin dependencias,
sin build, sin servidor: se abre `index.html` y se juega. Anda igual en la compu y en
el celular.

## Cómo se juega

Las paredes gruesas parten la grilla en **tramos**: cada secuencia de casillas
seguidas, horizontal o vertical, que no cruza ninguna pared.

- Un tramo de largo **L** lleva exactamente los números **1 a L**, uno de cada uno.
- Cada casilla pertenece a dos tramos: el horizontal y el vertical.
- No se repite dentro de un tramo. **Sí** puede repetirse dentro de una fila o
  columna completa, si son tramos distintos.
- Un tramo de una sola casilla siempre lleva 1.

Ejemplo resuelto de 4×4 (`|` y `—` son paredes):

```
4 3 2 1        H: [4]  [2|2]  [4]  [3|1]
1 2|1 2        V: [4]  [4]  [2|2]  [1|3]
2 4 1 3
3 1 2|1
```

Se juega con el mouse o el dedo: click en una casilla y después click en el número.
Tocar de nuevo el mismo número lo borra. Si el número no entra en esa casilla, el
botón pega un flash y no lo toma. Con teclado: `1`–`9`, el `0` es el 10, flechas para
moverse, `Supr` borra.

## Tamaños y dificultad

Grilla de 6×6 a 10×10, cuatro niveles. Los puzzles se generan en el momento y todos
tienen **solución única**. Fácil, Medio y Difícil se resuelven siempre por deducción
pura; Experto puede requerir ensayo y error.

Todo tablero usa el número más alto de su tamaño (en 8×8 aparece el 8), no tiene
zonas encerradas por paredes ni casillas aisladas de 1×1.

`?n=10&d=3` en la URL arranca directamente en ese tamaño y dificultad.

---

# Cómo funciona por dentro

Todo el motor está en el mismo `index.html`, delimitado por los comentarios
`// ==ENGINE-START==` y `// ==ENGINE-END==`. No toca el DOM, así que se puede
extraer y probar en Node sin navegador:

```js
const html = require('fs').readFileSync('index.html', 'utf8');
const src = html.match(/\/\/ ==ENGINE-START==([\s\S]*?)\/\/ ==ENGINE-END==/)[1];
const E = new Function(src + '\nreturn {generate,buildCtx,countSolutions,humanSolve};')();
const p = E.generate(8, 2, 12345, 6000);   // N, dificultad, semilla, presupuesto ms
```

`generate` es determinista dada la semilla: la misma semilla da siempre el mismo
tablero.

## Representación

```
vW[r*(N-1)+c]   pared entre (r,c) y (r,c+1)
hW[r*N+c]       pared entre (r,c) y (r+1,c)
```

`buildCtx` deriva de ahí la lista de **tramos**, y para cada casilla el id de su
tramo horizontal y el de su vertical. `maxv[i] = min(largoH, largoV)` es la cota
de esa casilla: ningún valor mayor puede ir ahí. Los dominios son bitmasks (bit
*v* = valor *v*), así que descartar un candidato es un `&`.

## El solver

`propagate` aplica dos reglas hasta que no cambia nada: el único candidato que
queda en una casilla, y el único lugar posible para un valor dentro de un tramo.
`search` hace backtracking eligiendo primero la casilla con menos candidatos, y
`countSolutions(ctx, given, 2)` corta al encontrar la segunda solución — que es
todo lo que hace falta para decidir si un puzzle es único.

## El generador

`buildBoard` **construye el tablero resolviéndolo**: recorre casilla por casilla y
en cada una decide a la vez si hay pared a la izquierda, si hay pared arriba, y
qué número va. No sortea paredes para después buscarles solución: eso casi nunca
funciona, por lo que sigue.

### Por qué el N obliga a una estructura escalonada

Para que en la grilla aparezca el número más alto hace falta una casilla con
tramo horizontal **y** vertical de largo N, o sea una fila entera y una columna
entera cruzándose. Pero además esa fila lleva 1..N, y **cada valor v necesita una
casilla cuya columna mida al menos v**: el N necesita una columna de largo N, el
N-1 dos columnas de largo ≥ N-1, y así (condición de Hall). Eso arrastra un
escalonado de líneas largas en las dos direcciones — es la razón de que estos
tableros tengan pocas paredes, no un capricho del generador.

Consecuencia práctica: **no se puede usar el largo máximo de tramo como palanca de
dificultad**. Si se topean los tramos por debajo de N, la fila entera se vuelve
matemáticamente irresoluble y el generador no produce nada.

La fila entera se planta siempre en la **fila 0** (`buildBoard(..., forceRow=0)`)
porque el constructor avanza de arriba abajo y así la resuelve antes de
comprometer nada; plantarla en el medio dispara un backtracking enorme. Para que
no salgan todos los tableros parecidos, `espejar()` aplica al azar una de las 8
simetrías del cuadrado al resultado terminado.

### Filtros de forma

Calibrados midiendo un tablero real de 8×8: **24 tramos, largo promedio 5,33
(0,67×N) y 33% de tramos de largo N**. Están todos juntos en `generate`:

| Filtro | Qué descarta |
|---|---|
| `isConnected` | zonas rodeadas de paredes por los cuatro lados (un sub-puzzle aparte) |
| `countTrivialCells > 0` | casillas 1×1, que son un 1 regalado y aislado |
| `countForcedOnes > 12%` | demasiados 1 forzados por tramos de largo 1 |
| `stats.avg` fuera de `0.56·N … 0.78·N` | tableros muy cortados o casi sin paredes |
| `stats.llenos` fuera de `0.18 … 0.45` | proporción de tramos de largo N |
| `stats.max !== N` | tableros donde el número más alto nunca se usa |

Si algún día querés tableros con más paredes, la palanca es `stats.avg` (y hay que
bajar `stats.llenos` en la misma proporción), no el largo de los tramos.

### Pistas y dificultad

`carve` arranca del tablero lleno y va sacando casillas al azar mientras la
solución siga siendo única, hasta que no se puede sacar ninguna más. Eso da el
puzzle más difícil posible con esas paredes.

`humanSolve` lo resuelve como una persona: en cada paso elige la deducción **más
barata** disponible y anota cuánto costó — 0 si la casilla tenía un solo candidato,
2 si fue mirar una casilla, o el largo del tramo si hubo que barrerlo entero. La
escala **no** es el paso más caro (un único paso difícil tiraría todo el tablero a
"difícil") ni el promedio (lo diluyen los muchos pasos baratos), sino **cuántas
veces hubo que barrer un tramo largo** — `caros`, con el umbral de "largo" en
`umbralCaro(N)`:

```
nivel 1   caros ≤ N/4
nivel 2   caros ≤ 0.75·N
nivel 3   caros > 0.75·N, o hizo falta una técnica avanzada
nivel 4   no se resuelve sin ensayo y error
```

Como sacar pistas sólo puede encarecer un tablero, `generate` busca el nivel desde
arriba: talla el mínimo, mide, y si quedó más difícil de lo pedido `easeTo`
devuelve pistas de a una, descartando las que lo abaratan de más. Antes de
entregar vuelve a verificar unicidad con el solver completo.

## Al tocar el motor

Vale la pena volver a correr la validación: generar unos cuantos puzzles de cada
tamaño y nivel y chequear que la solución respete los tramos, que
`countSolutions(ctx, given, 2)` dé exactamente 1, que `isConnected` sea cierto, que
no haya casillas 1×1 y que `max(sol) === N`. La última corrida completa fueron 120
puzzles (5 tamaños × 4 niveles × 6 cada uno) sin fallos, con el peor tiempo de
generación en 1 s.
