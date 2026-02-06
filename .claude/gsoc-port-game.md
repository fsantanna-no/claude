### Port an Existing Game to Atmos Evaluating Structured Reactive Patterns

[Atmos](https://github.com/atmos-lang/atmos/) is an experimental
programming language that compiles to Lua 5.4 and reconciles structured
concurrency, event-driven programming, and functional streams.
Combined with [pico-lua](https://github.com/fsantanna/pico-sdl/tree/main/lua/)
(a Lua binding for
[pico-sdl](https://github.com/fsantanna/pico-sdl/)),
[pico-atmos](https://github.com/atmos-lang/atmos/) enables 2D game
development with structured reactive primitives such as `await`/`emit`,
`par`/`par_or`/`par_and`, and `spawn`/`task`.

A [previous case study](https://fsantanna.github.io/pingus/) rewrote
the game logic of [Pingus](https://pingus.github.io/) (an open-source
Lemmings clone) from C++ to
[Céu](http://www.ceu-lang.org/), the predecessor of Atmos.
That study identified **5 control-flow patterns** commonly found in
game development:

1.  **Finite State Machines** --- entities whose behavior maps event
    occurrences to transitions between states (e.g., double-click
    detection, character animations).
2.  **Continuation Passing** --- long-lasting activities that carry an
    action to execute next upon completion (e.g., interactive dialogs,
    menu transitions).
3.  **Dispatching Hierarchies** --- container entities that
    automatically forward stimuli to managed children (e.g., redraw and
    update callbacks).
4.  **Lifespan Hierarchies** --- container entities whose termination
    automatically destroys managed children (e.g., UI containers,
    particle systems).
5.  **Signaling Mechanisms** --- entities that communicate explicitly
    when no hierarchy relationship exists between them (e.g., key
    shortcuts, screen pausing).

The Pingus rewrite reduced 515 lines of C++ to 173 lines of Céu in
the engine alone, mostly by eliminating dispatching code.  Across the
full game logic, 9,186 lines were rewritten with an overall Céu/C++
ratio of 0.89.

This project proposes to **port an existing open-source 2D game to
Atmos/pico-atmos**, systematically applying and evaluating these
patterns.  The contributor will select a game (written in Lua, C, C++,
or another language) that exercises most or all of the five patterns
above.  Good candidates include well-known open-source games with menus,
animations, entity hierarchies, and inter-entity communication --- for
example, games built with [LÖVE](https://love2d.org/) or other Lua
frameworks.

The project has two interleaved tracks:

*   **Implementation** --- Incrementally port the game to pico-atmos,
    replacing callbacks, state variables, and dispatching code with
    Atmos constructs (`await`/`emit`, `par`/`par_or`, `spawn`, `task`,
    pattern matching, etc.).
*   **Evaluation** --- For each rewritten module, classify the applied
    patterns against the Pingus taxonomy, compare lines of code, and
    document qualitative observations (readability, expressiveness,
    limitations encountered).

#### Expected results

*   A fully playable port of the chosen game running on pico-atmos.
*   A written report mapping each rewritten module to one or more of
    the 5 control-flow patterns, including quantitative metrics (lines
    of code, number of files, callback elimination ratio) and
    qualitative observations.
*   Identification of any new patterns not covered by the original
    Pingus taxonomy.
*   Source code published in a public repository with clear commit
    history showing the incremental porting process.

#### Prerequisites

*   Programming experience in Lua and at least one other language
    (C, C++, or Python).
*   Familiarity with 2D game development concepts (game loop, sprites,
    collision detection, state machines).
*   Willingness to learn Atmos (the
    [manual](https://github.com/atmos-lang/atmos/blob/main/doc/manual-out.md)
    and existing games
    [Birds](https://github.com/atmos-lang/pico-birds/) and
    [Rocks](https://github.com/atmos-lang/pico-rocks/) serve as
    learning material).
*   Reading the Pingus case study:
    [web version](https://fsantanna.github.io/pingus/) and
    [SBGames 2018 paper](http://www.ceu-lang.org/chico/ceu_sbgames18_pre.pdf).

#### Skill level

*   Intermediate

#### Project size

*   Medium (175 hours) or Large (350 hours), depending on the
    complexity of the chosen game.

#### Mentors

*   Francisco Sant'Anna
