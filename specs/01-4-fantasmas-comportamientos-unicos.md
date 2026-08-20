# SPEC 01 — 4 Fantasmas con Comportamientos Únicos y Modo Asustado

> **Status:** Draft
> **Depends on:** None
> **Date:** 2026-08-20
> **Objective:** Implementar 4 fantasmas con comportamientos diferenciados (hunter, ambusher, 2× random) y modo asustado activado por Power Pellets.

---

## Scope

**In:**

- 4 fantasmas con comportamientos únicos: `hunter`, `ambusher`, 2× `random`.
- Power Pellets en las 4 esquinas clásicas del laberinto.
- Modo asustado clásico: fantasmas azules, movimiento aleatorio, comestibles por Pacman.
- Sistema de puntos progresivo al comer fantasmas asustados (200, 400, 800, 1600).

**Out of scope (for future specs):**

- Niveles de dificultad progresivos.
- Modos de juego diferentes (multiplayer, etc.).
- Animaciones especiales de fantasmas más allá del color azul en modo asustado.
- Fruita (fruta bonus por nivel).

---

## Data model

### Posiciones iniciales (`maze.js`)

```js
const GHOST_STARTS = [
  { x: 13, y: 14, kind: 'hunter' },   // Blinky — persigue agresivamente
  { x: 14, y: 14, kind: 'random' },   // Pinky — movimiento aleatorio
  { x: 13, y: 12, kind: 'ambusher' }, // Inky — prioriza posiciones estratégicas
  { x: 14, y: 12, kind: 'random' },   // Clyde — movimiento aleatorio
];
```

### Posiciones de Power Pellets (`maze.js`)

```js
const POWER_PELLET_POSITIONS = [
  { x: 1, y: 5 },
  { x: 26, y: 5 },
  { x: 1, y: 29 },
  { x: 26, y: 29 },
];
```

### Estado del modo asustado (`game.js`)

```js
// Agregado al objeto game retornado por createGame():
{
  // ...existente...
  scaredMode: false,
  scaredTimer: 0,
  scaredGhostEaten: 0, // contador para puntos progresivos
}
```

### Grid: Power Pellets como valor 4

El laberinto usa valores numéricos: 0=vacío, 1=pared, 2=dot, 3=puerta. Power Pellets se representan como `4` en el grid.

---

## Implementation plan

### Paso 1 — Actualizar posiciones iniciales de fantasmas (`maze.js`)

Agregar 2 entradas a `GHOST_STARTS` y la constante `POWER_PELLET_POSITIONS`.

- Modificar `src/js/maze.js:54-57`.
- Agregar `POWER_PELLET_POSITIONS` como nueva constante.
- Verificar que las 4 posiciones de Power Pellets coinciden con celdas vacías (valor 0) en el laberinto actual.

### Paso 2 — Marcar Power Pellets en el grid (`maze.js`)

Al copiar el grid en `createGame()`, reemplazar las celdas de Power Pellets por valor `4`.

- Modificar `src/js/game.js:18-24`.
- Después de copiar el grid, iterar `POWER_PELLET_POSITIONS` y asignar `grid[y][x] = 4`.
- Contar Power Pellets junto con dots en `dotsRemaining`.

### Paso 3 — Renderizar Power Pellets (`render.js`)

Dibujar Power Pellets como círculos grandes pulsantes (efecto de parpadeo).

- Agregar función `drawPowerPellets(ctx, grid)` en `src/js/render.js`.
- Power Pellet: círculo radio 5px, color blanco, con opacidad que oscila según frame (parpadeo).
- Llamar desde `draw()` después de `drawDots()`.
- Power Pellets solo se muestran si `grid[y][x] === 4`.

### Paso 4 — Comer Power Pellets (`game.js`)

En `movePacman()`, detectar cuando Pacman entra en una celda con valor `4`.

- Cambiar `grid[y][x]` de `4` a `0`.
- Restar de `dotsRemaining`.
- Sumar 50 puntos (clásico).
- Activar modo asustado: `game.scaredMode = true`, `game.scaredTimer = 360` (~6 segundos a 60fps).
- Resetear `scaredGhostEaten = 0`.

### Paso 5 — Implementar comportamiento `ambusher` (`game.js`)

En `decideGhost()`, agregar caso para `g.kind === 'ambusher'`.

- Elegir dirección que maximiza la distancia a Pacman (inverso del hunter).
- Priorizar celdas que estén en cuadrantes estratégicos (esquinas del laberinto).
- Si hay empate, elegir al azar entre las opciones.
- El ambusher siempre está activo (sin condición de activación).

### Paso 6 — Renderizar fantasmas en modo asustado (`render.js`)

Cuando `game.scaredMode === true`, cambiar el color de todos los fantasmas.

- Color azul: `#2121ff` (mismo que las paredes, o azul claro `#4444ff` para diferenciar).
- Los ojos permanecen igual (mirando en la dirección del movimiento).
- Lógica en `drawGhost()`: si `game.scaredMode`, usar color azul en vez del color original.

### Paso 7 — Lógica de timeout del modo asustado (`game.js`)

En `update()`, decrementar `scaredTimer` cada frame cuando `scaredMode === true`.

- Cuando `scaredTimer <= 0`, desactivar: `scaredMode = false`.
- Resetear colores de fantasmas (vuelven al comportamiento normal).

### Paso 8 — Comer fantasmas asustados (`game.js`)

En la detección de colisión (`update()`), distinguir si el fantasma está asustado.

- Si `scaredMode === true` y colisiona con Pacman:
  - Sumar puntos: `200 * Math.pow(2, scaredGhostEaten)`.
  - Incrementar `scaredGhostEaten`.
  - Reiniciar posición del fantasma a su `GHOST_STARTS` original.
  - No restar vida a Pacman.
- Si `scaredMode === false` y colisiona: comportamiento actual (restar vida).

### Paso 9 — Verificación manual

Probar que:
- Los 4 fantasmas se mueven de forma diferenciada.
- Power Pellets parpadean en las 4 esquinas.
- Comer un Power Pellet activa modo asustado (~6 segundos).
- Fantasmas se vuelven azules durante el modo asustado.
- Comer fantasmas asustados otorga 200, 400, 800, 1600 puntos progresivamente.
- Al expirar el timer, fantasmas vuelven al comportamiento normal.
- El juego no tiene errores en consola.

---

## Acceptance criteria

- [ ] 4 fantasmas aparecen en el laberinto con posiciones correctas.
- [ ] Cada fantasma tiene un comportamiento visualmente distinto (hunter persigue, ambusher prioriza esquinas, random se mueve al azar).
- [ ] 4 Power Pellets parpadean en las esquinas del laberinto.
- [ ] Comer un Power Pellet activa modo asustado por ~6 segundos.
- [ ] Fantasmas se vuelven azules durante el modo asustado.
- [ ] Comer un fantasma asustado otorga 200 puntos la primera vez.
- [ ] Cada fantasma asustado subsiguiente otorga el doble (400, 800, 1600).
- [ ] Fantasmas comidos reaparecen en su posición inicial.
- [ ] Al expirar el timer, los fantasmas vuelven a su color y comportamiento normal.
- [ ] El juego carga sin errores en la consola.
- [ ] `dotsRemaining` incluye tanto dots como Power Pellets.

---

## Decisions

- **Sí:** Power Pellets como valor `4` en el grid. Mantiene consistencia con el sistema existente de valores numéricos (0-3).
- **Sí:** 4 Power Pellets en posiciones clásicas (esquinas). Fidelidad al juego original.
- **Sí:** ~6 segundos de modo asustado (360 frames a 60fps). Equilibrio entre riesgo y recompensa.
- **Sí:** Puntos progresivos (200, 400, 800, 1600). Incentiva al jugador a aprovechar el modo asustado.
- **Sí:** Ambusher siempre activo. Simplifica la implementación y mantiene presión constante.
- **No:** Power Pellets temporales (que desaparecen si no se comen). Complejidad innecesaria.
- **No:** Modo asustado con duración variable por nivel. Se reserva para un spec futuro de niveles.
- **No:** Fantasmas con velocidades diferentes. Mantener la misma velocidad simplifica el balance.

---

## Risks

| Risk | Mitigation |
|------|------------|
| Power Pellet en celda que no es vacía (0) | Verificar posiciones en Paso 1 antes de implementar. |
| Timer de modo asustado se desincroniza con el frame rate | Usar contador de frames, no tiempo real. |
| Colisión fantasma asustado + normal simultánea | En `update()`, verificar `scaredMode` antes de restar vida. |

---

## What is **not** in this spec

- Niveles de dificultad progresivos.
- Modos de juego diferentes.
- Animaciones especiales de fantasmas más allá del color azul.
- Fruita (fruta bonus por nivel).
- Power Pellets temporales o que desaparecen.
