# SPEC 02 — Salida de los fantasmas de la jaula

> **Status:** Approved
> **Depends on:** SPEC 01
> **Date:** 2026-08-21
> **Objective:** Sacar a los fantasmas de la jaula al mapa mediante un estado de salida guiada ('house' → 'exiting' → 'normal'), con salida escalonada al iniciar partida e inmediata al reaparecer.

---

## Scope

**In:**

- Nuevo estado por fantasma (`houseState`: `'house'` | `'exiting'` | `'normal'`) en `src/js/game.js`.
- Ruta guiada de salida: alinearse en la columna de la puerta (13 o 14) y subir hasta la fila 11.
- Salida escalonada al iniciar partida: primer fantasma inmediato, siguientes cada ~2 segundos.
- Reaparición tras ser comido o tras perder vida: mismo mecanismo guiado, pero con salida inmediata (sin cola).

**Out of scope (for future specs):**

- Comportamientos `hunter` / `ambusher` / `random` (SPEC 01). No se modifican.
- Modo asustado, puntos y velocidades. No se tocan.
- Rebote vertical de espera dentro de la jaula (animación idle clásica).
- Timers de salida por nivel o dificultad.
- Que Pacman pueda cruzar la puerta (sigue bloqueado por `isWall`, src/js/game.js:69).

---

## Data model

### Constantes nuevas (`game.js`)

```js
const EXIT_ROW = 11;             // primera fila fuera de la jaula
const RELEASE_DELAY_FRAMES = 120; // ~2 s entre salidas escalonadas a 60 fps
```

### Estado nuevo por fantasma (`game.js`, en `createGame()`)

```js
ghosts: GHOST_STARTS.map( ( g, i ) => ( {
  x: g.x,
  y: g.y,
  dir: 'up',
  speed: GHOST_SPEED,
  kind: g.kind,
  houseState: 'house',                    // 'house' | 'exiting' | 'normal'
  releaseTimer: i * RELEASE_DELAY_FRAMES, // escalonado por índice
} ) ),
```

No hay valores nuevos de grid ni cambios en `maze.js`. La puerta sigue siendo valor `3`.

---

## Implementation plan

### Paso 1 — Estado de jaula en `createGame()` (`game.js`)

Agregar constantes `EXIT_ROW` y `RELEASE_DELAY_FRAMES`, y los campos `houseState` + `releaseTimer` a cada fantasma.

- Modificar `src/js/game.js:47-53`.
- Verificación manual: recargar; el juego funciona igual porque el estado todavía no se usa.

### Paso 2 — Bucle de estados en `update()` (`game.js`)

En `update()`, para cada fantasma: si `houseState !== 'normal'`, no llamar `moveGhost()`/`decideGhost()`; en su lugar:

- `'house'`: decrementar `releaseTimer`; al llegar a ≤ 0 → `'exiting'`.
- `'exiting'`: avanzar por ruta guiada con función nueva `moveExitingGhost( game, g )`:
  - Si `g.y > EXIT_ROW` y no está alineado a su columna objetivo → moverse horizontal hacia ella (columna 13 si `g.x <= 13`, sino 14).
  - Ya alineado en columna → moverse `up` una celda por frame (respeta `GHOST_SPEED`).
  - Al alcanzar `y === EXIT_ROW` → `houseState = 'normal'`; en la siguiente celda decide la IA normal de `decideGhost()`.
- Modificar `src/js/game.js:211-247`.
- Verificación manual: partida nueva; hunter sale al instante y el resto cada ~2 s, todos por la puerta.

### Paso 3 — Reapariciones usan la misma salida (`game.js`)

Reutilizar la mecánica sin cola de espera:

- En "comer fantasma asustado" (`src/js/game.js:226-232`): reposicionar a `GHOST_STARTS[i]` con `houseState = 'exiting'` y `releaseTimer = 0`.
- En `resetPositions()` (`src/js/game.js:194-205`): los 4 fantasmas quedan con `houseState = 'exiting'` y `releaseTimer = 0` (salen juntos al momento).
- Verificación manual: comer un fantasma asustado y verlo salir otra vez por la puerta; perder una vida y ver salir a los 4.

### Paso 4 — Verificación manual final

Recorrer los criterios de aceptación completos en partida nueva, tras muerte de Pacman y tras comer un fantasma.

---

## Acceptance criteria

- [ ] Al iniciar partida, hunter (Blinky) inicia su salida de inmediato.
- [ ] Pinky, Inky y Clyde inician su salida ~2 s después cada uno, en orden de `GHOST_STARTS`.
- [ ] La salida es físicamente por la puerta (celdas 13–14 de la fila 12) hasta la fila 11.
- [ ] Ningún fantasma circula indefinidamente dentro de la jaula; todos están en el mapa en menos de 10 segundos.
- [ ] Tras perder vida, los 4 vuelven a salir por la puerta de inmediato.
- [ ] Un fantasma comido en modo asustado reaparece en su celda inicial y sale otra vez por la puerta de inmediato.
- [ ] Pacman sigue sin poder entrar a la jaula.
- [ ] Los comportamientos hunter/ambusher/random funcionan igual una vez en el mapa.
- [ ] El juego carga sin errores en la consola.

---

## Decisions

- **Sí:** Estado explícito `'house'/'exiting'/'normal'` por fantasma. Convierte la salida en un problema determinista en vez de dejarla a la suerte de la IA aleatoria (causa raíz del bug).
- **Sí:** Salida escalonada clásica. Elección del usuario frente a salida simultánea.
- **Sí:** Reapariciones con salida inmediata sin cola. Consistencia total; evita re-introducir el atrapamiento tras ser comidos o perder vida.
- **Sí:** Columna de puerta según posición inicial (13 si `x <= 13`, sino 14). Camino mínimo sin cruces innecesarios.
- **No:** Mover `GHOST_STARTS` fuera de la jaula. Descartado: dejaba la jaula decorativa y cambiaba la semántica de respawn.
- **No:** Rebote vertical de espera dentro de la jaula. Animación extra sin valor para este fix.
- **No:** Timers de salida por nivel/dificultad. Reservado para una spec futura de niveles.
- **Propuesta editable:** intervalo de 120 frames (~2 s) y orden según índice de `GHOST_STARTS` (hunter → Pinky → Inky → Clyde). El usuario pidió "escalonada clásica" sin fijar parámetros; estos valores son la propuesta por defecto y se pueden cambiar durante la revisión de este `Draft`.

---

## Risks

| Risk | Mitigation |
|------|------------|
| Fantasmas 'exiting' se atascan a mitad de camino | La ruta guiada solo usa movimientos de 1 celda alineada, igual que el resto del juego; no depende de `decideGhost()`. |
| Colisión fantasma–Pacman durante la salida cerca de la fila 11 | La detección de colisión existente no se toca; a la fila 11 pasan a 'normal'. |
| Romper el modo asustado al tocar `update()` | El timer de asustado se decrementa fuera del bucle de fantasmas (src/js/game.js:216); ese bloque queda intacto. |

---

## What is **not** in this spec

- Cambios a comportamientos hunter/ambusher/random.
- Cambios al modo asustado, puntos o velocidades.
- Animación de espera dentro de la jaula.
- Dificultad progresiva o timers por nivel.
- Acceso de Pacman a la jaula.
