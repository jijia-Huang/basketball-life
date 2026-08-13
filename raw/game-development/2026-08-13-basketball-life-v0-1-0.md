# Basketball Life v0.1.0 implementation snapshot

> Source: Local project files `index.html`, `GAME_DESIGN_DOCUMENT.md`, `BALANCE_AND_FORMULAS.md`, and browser acceptance run
> Collected: 2026-08-13
> Published: 2026-08-13

Basketball Life v0.1.0 is a playable, dependency-free, single-file basketball career simulator implemented in `index.html`.

The player starts at age 16. Initial height ranges from 165 cm to 211 cm, yearly growth ends after age 21, and height is capped at 230 cm. The model tracks weight, BMI, ten basketball abilities, potential, training carry, dynamic PG/SG/SF/PF/C positions, five offensive styles, and five defensive styles.

The playable career includes amateur basketball, local professional leagues, a two-round NBA-style draft, the NBA, G League, Taiwan, Japan, Europe, contracts, free agency, trades, injuries, load management, national-team selection, awards, championships, retirement, and a career score.

Season output is generated from role, games played, starts, minutes, usage, shot distribution, shooting percentages, rebounds, assists, steals, blocks, turnovers, team record, and seeded random variation. Low-ability NBA players can remain as DNP or fringe bench players while under contract.

The interface preserves `?seed=<code>` deterministic replay and provides `?selftest=1` for a runnable core-formula check. The self-test result was Selftest PASS. A browser acceptance run completed one full amateur season at a 390 by 844 mobile viewport with no page-script errors; the document width stayed within the viewport.

The public page target is `https://jijia-huang.github.io/basketball-life/` and the repository is `https://github.com/jijia-Huang/basketball-life`.
