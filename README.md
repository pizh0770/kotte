# Bomberman World — easiest possible mission for a bomber

**Student:** Pinqi Zhang

## How to play

- **Controls:** 
	**Movement:** W, A, S, D / Arrow keys
	**Place bomb:** Space
	**Activate shield:** E
	**Restart:** R, available after a round ends
	**Quit:** Q / Escape

- **Objective and rules:** 
	**Winning / losing:** Kill 10 enemies to win the game. The round ends if the player touches an enemy or gets caught in an explosion. If the player dies at the same moment as they kill the final enemy, the player loses.
	**Bomb:** The player can have up to 3 active bombs at once. Bombs block movement, but the player can move off a bomb after placing it. The player can't place multiple bombs on the same tile. An explosion touching another bomb triggers an immediate detonation.
	**Power-up:** Collect the blue item to increase the blast range of future bombs from 3 tiles to 4 tiles. Bombs already placed keep their original range.
	**Shield:** The shield gives the player 7 seconds of invulnerability. It can only be used once per round.

- **Configurations / demo setup:** 
	Default.

## Features and validation

| Feature                                | Check, observed result, and expected behaviour                                                                                                                                                                                                                                                                                                                                                                                                                                                                     |
| -------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| 1. Player damage and death             | I moved the player to a bomb's blast range and observed that the player disappeared from the screen when the bomb exploded. When an enemy moved onto the tile where the player had been, no collision occurred. The player took no damage standing near the edge of the blast range. I also moved the player into an enemy's path and confirmed that the player disappeared after colliding with the enemy.                                                                                                        |
| 2. Round states and objective          | I killed 10 enemies and confirmed that the win screen appeared. When a bomb explosion hit both the tenth enemy and the player at the same time, the lose screen appeared. Movement, bomb placement and shield activation stopped working when the game was in the win / lose state. Enemies continued to move, and bombs that had already been placed could still kill enemies, but the "enemies killed" count didn't change. Explosions that happened while the game was in the win state didn't kill the player. |
| 3. Safe restart                        | I placed two bombs and let an enemy kill the player. I restarted the round while the first bomb's blast effect was still visible and the second bomb had not exploded. The map reset, and all entities returned to their original starting positions. No explosions from the previous round appeared, and the bomb capacity reset.                                                                                                                                                                                 |
| 4. Solid bombs with owner escape       | I placed a bomb and confirmed that the player could leave that tile. The player couldn't re-enter the tile or place another bomb on it while the bomb was active. Enemies couldn't enter a tile with an active bomb. (\*An enemy already on the tile when the bomb was placed got stuck and couldn't leave until the bomb exploded.)                                                                                                                                                                               |
| 5. Blast-range pickup                  | I placed a bomb, collected the blue item and placed another bomb. The first bomb kept its original blast range and the second bomb had an increased range of 4 tiles. I moved to the pickup's former position and placed another bomb. The blast range didn't increase again.                                                                                                                                                                                                                                      |
| 6. Multiple bombs per owner            | I placed three bombs quickly and tried to place a fourth. The fourth bomb couldn't be placed until one of the active bombs exploded. Once a slot became available I was able to place another bomb.                                                                                                                                                                                                                                                                                                                |
| 7. Chain reactions                     | I placed two bombs within blast range. When the first bomb exploded, its blast triggered the second bomb immediately. The blast of the second bomb damaged the player and enemies as expected. The second bomb didn't explode again later.                                                                                                                                                                                                                                                                         |
| 8. Shield or temporary invulnerability | I pressed E to activate the shield. I moved the player to a bomb's blast range and observed that the player didn't die when the bomb exploded. Collision with an enemy didn't kill the player either. Once the timer ran out the player took damage normally. Pressing E later in the same round didn't activate the shield again.                                                                                                                                                                                 |

**Known issues:** None known after testing.

## Sources and assistance

This project builds on the shared Kotte foundations.

Visual Studio's IntelliSense and GitHub Copilot's inline code suggestions were enabled during development. Only suggestions that matched the intended implementation were accepted and the resulting code was checked manually.
## Architectural investigation — keeping track of bomb capacity

*How should the game know whether the player can place another bomb?*

### Decision

The current approach uses a vector (`active_bomb_handles_`) to store handles for active bombs. The maximum number of bombs is set to 3 (`bomb_capacity`). When the player tries to place a bomb, the function `owner_active_bomb_count()` counts the number of active bombs owned by the player. The placement limit is checked when the request is queued and when the bomb is created.
A plausible alternative could be using a separate counter stored on the player entity to check the number of active bombs. The counter would increase when a bomb is created and decrease when it is destroyed.

### Method

I added diagnostic text to `Application::render()` to monitor `owner_active_bomb_count(player_handle_)`, `bomb_capacity_` and `active_bomb_handles_.size()` in real time. I tested the game by starting a new round, placing three bombs to reach the limit, and trying to place a fourth. Then I waited for one bomb to explode and placed a replacement. Finally, I restarted the round by pressing R while 2 bombs were still active.
```
diagnostics = std::format(
    "enemies updated: {} | active bombs: {} / {} | bomb handles: {} | blast effects: {}",
    collision_diagnostics_.enemy_updates,
    owner_active_bomb_count(player_handle_),
    bomb_capacity_,
    active_bomb_handles_.size(),
    blast_effects_.size());
DrawText(diagnostics.c_str(), 10, 58, 20, PINK);
```

### Evidence

| Phase                         | Count / capacity | Handles | Observed result                                                    |
| ----------------------------- | ---------------- | ------- | ------------------------------------------------------------------ |
| Round start                   | 0 / 3            | 0       | Bomb placement accepted                                            |
| Placed three bombs            | 3 / 3            | 3       | Capacity reached                                                   |
| Tried placing another bomb    | 3 / 3            | 3       | Bomb placement was rejected                                        |
| One bomb exploded             | 2 / 3            | 2       | Handle removed. A new bomb could be placed                         |
| Restarted with two live bombs | 0 / 3            | 0       | Lists cleared. Full capacity restored. Three bombs could be placed |

### Interpretation

The current approach keeps the bomb count consistent by calculating it from `active_bomb_handles_` whenever `owner_active_bomb_count()` is called. It checks whether a bomb handle points to a valid entity, whether the bomb has a valid state, and whether the owner is the player. One limitation of this approach is that the function must iterate through the vector every time a check is required. If the game had a much larger number of active bombs, the performance cost of repeated counting might become too high. The alternative approach stores the active bomb count on the player, which would make capacity checks faster. However, a stored count must be updated correctly whenever a bomb is placed or destroyed, and missing an update could cause the count to become incorrect. With a maximum of three bombs in the game, the performance benefit of storing a separate counter would be very small, and the current approach avoids the work of maintaining an additional value.
### Conclusion

The current approach is appropriate for the game because the bomb capacity is limited to a small number. It simplifies count management and reduces the risk of counting errors without affecting game performance. The alternative would become preferable in larger games with more active objects and more frequent checks.

<!-- I love Markdown :) -->