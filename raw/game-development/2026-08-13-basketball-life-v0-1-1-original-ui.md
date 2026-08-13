# Basketball Life v0.1.1 original-interface restoration

> Source: Local project `index.html` and browser acceptance run
> Collected: 2026-08-13
> Published: 2026-08-13

Basketball Life v0.1.1 keeps the basketball rules and restores the interaction structure of the source YaKyoLife interface.

The restored shell uses the deep-green scoreboard as its default, a four-cell career board, three season-phase lamps, a desktop three-column layout, a career timeline, four selectable visual themes, and a collapsible bottom action area on mobile.

Training again follows a two-step flow. First, the game reveals the seeded dice result and presents a dedicated allocation button. In allocation, the current die is highlighted, used dice fade, ability rows show current value and potential, undo restores both the ability and the current die, and confirmation appears only after all dice are assigned. Individual allocations do not add log cards; one summary card is added after confirmation.

Browser acceptance used a 390 by 844 mobile viewport and a 1280 by 800 desktop viewport. The test verified a four-die result of 6, 6, 4, and 1, undo restoration, confirmation gating, phase progression from preseason to midseason, no horizontal overflow, and no page-script errors. The built-in formula test returned Selftest PASS.
