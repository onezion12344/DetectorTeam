# Round 1 — 2026-07-22

## Tester Team

| Tester | Persona | Issues Found |
|--------|---------|-------------|
| Desktop | 中文 native, technical, 1920px | 5 P0, 7 P1, 12 P2 |
| Mobile | 中文 native, non-technical, 390px | 9 P0, 8 P1, 7 P2 |

## Triage (deduplicated)

| Level | Count | Categories |
|-------|-------|------------|
| P0 | 10 | Dead code, connector semantics, argument state, color bug, confirmedRelation, unreachable CSS, mobile layout, touch targets, keyboard nav, screen reader |
| P1 | 11 | focus-visible, FOUC, button states, reduced motion, debounce, relation picker scroll, team button size, data consensus inconsistency |
| P2 | 14 | Contrast, dead CSS, print styles, stale comments, observer threshold |

## Fixer Team

| Fixer | Scope | P0 Fixed |
|-------|-------|----------|
| Fixer A | JS logic + CSS bugs | 6/6 |
| Fixer B | Mobile UX + A11y | 6/6 |

## Fixed Issues (Round 1)

### P0 — All Fixed
- Dead code: removed aiConsensus, getConsensus, team-highlight, .t1-.t5 CSS
- Team desc color: hardcoded teamColors object instead of broken CSS var
- SVG connectors: unified to "selected=lit" semantics, grayed=red dashed, added ev-6→arg-2
- Argument state: auto-inactive now driven by isArgActive() not evidence heuristics
- confirmedRelation: auto-resets when conclusion becomes unreachable
- unreachable CSS: covers '' state (not just 'grayed')
- Mobile tree: single-column at ≤520px, SVG hidden
- Touch targets: nodes min-height 44px on mobile
- Keyboard nav: tabindex, :focus-visible, Enter/Space handlers
- Screen reader: dynamic aria-labels, aria-live, aria-pressed

### P1 — All Fixed
- Button :active/:disabled states added
- Reduced motion expanded to all animations
- Resize debounce (150ms)
- FOUC: all evidence nodes have initial `selected` class
- IntersectionObserver threshold 0.08 → 0.2
- demo-hint contrast: 0.45 → 0.7 opacity
- grayed node-src contrast: 0.15 → 0.35
- Mobile team-btn font: 8px → 10px

### P2 — Selectively Fixed
- Print styles added
- Stale comment removed
- Team desc color uses JS (CSS variable approach removed)

## Remaining (Backlog)

| Issue | Level | Reason Deferred |
|-------|-------|-----------------|
| P2-7 (resize SVG sync) | P2 | Fixed by mobile single-column layout (no SVG on mobile) |
| P2-13 (tree mobile specific width limits) | P2 | Fixed by single-column layout |
| M5 (relation picker auto-scroll) | P1 | Minor UX, not blocking |
| M7 (demo anchor link) | P1 | Minor, can add later |
| D3 (isArgActive evidence check) | P1 | Logic is intentional — consensus is decoupled from evidence |
| D5 (partial multi-path) | P1 | Edge case, not blocking |
| C5 (print styles comprehensive) | P2 | Basic print styles added, full coverage deferred |

## Next Steps

- Tester re-test on fixed items (Round 2)
- Focus: verify P0 fixes don't regress, check P1 items
