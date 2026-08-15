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
