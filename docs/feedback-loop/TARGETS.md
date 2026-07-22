# Test Targets — Detector Team Landing Page

**URL**: https://onezion12344.github.io/DetectorTeam/
**Type**: Static single-page HTML (no build step, no JS framework)
**Source**: `index.html` in repo root

## Core Journeys

1. **Framework understanding**: Page load → read hero → understand Data→Evidence→Arguments→Conclusions framework
2. **Demo interaction**: Scroll to demo → click evidence nodes → observe argument/conclusion changes
3. **Team presets**: Click team buttons (自定义/法院/Team A/Team B) → observe state changes
4. **Relation confirmation**: When multi-mapping → relation picker appears → click to confirm path
5. **Information architecture**: Read pipeline → principles → research → CTA links work

## Key UI Elements

- Evidence nodes (6): click to cycle 采信/排除/待定
- Argument nodes (3): click to cycle AI共识/采信/排除
- Team preset buttons (4)
- Data layer (collapsible)
- AI consensus panel
- Reasoning panel
- Relation picker (conditional)
- SVG connector lines
- Scroll reveal animations

## Personas

1. **Desktop user**: 中文 native, technical, 1920px viewport, interested in AI/legal
2. **Mobile user**: 中文 native, non-technical, 390px viewport, curious about the case
