# Third-party notices

The skills added in kit v1.1.0 are **original texts written for this kit**. Their process ideas (workflow shape, anti-rationalization tables, verification gates, decision scales) are *inspired by* the MIT-licensed projects below. No text or code was copied verbatim, and nothing from CC-BY-SA material (for example `trailofbits/skills`) was used.

Each affected `SKILL.md` records its sources in `metadata.inspired_by` and in a closing "Inspired by" section.

## Inspirations (MIT)

| Project | Used as inspiration for | Skills |
|---|---|---|
| [addyosmani/agent-skills](https://github.com/addyosmani/agent-skills) | Phase-based workflow, anti-rationalization tables, verification gates, scope discipline, Google engineering practices | `shp-tdd`, `shp-debugging`, `shp-incremental-build`, `shp-doubt-check`, `shp-api-design`, `shp-observability`, `shp-adr`, `shp-simplify`, `shp-router`, `shp-go-review` (extended) |
| [obra/superpowers](https://github.com/obra/superpowers) | One-hypothesis-at-a-time debugging | `shp-debugging` |
| [fidelisakilan/design-pattern-skills](https://github.com/fidelisakilan/design-pattern-skills) | Reuse / introduce / refactor first / no pattern decision, Rule of Three | `shp-patterns-select` |
| [jaeleeps/gof-patterns](https://github.com/jaeleeps/gof-patterns) | Appropriate / Over-engineered / Misapplied / Degraded verdict scale | `shp-patterns-select`, `shp-go-review` (audit mode) |
| [samber/cc-skills-golang](https://github.com/samber/cc-skills-golang) | Topic list: enums, `init()` and globals, panic vs error | `shp-patterns-creational` (extended), `shp-go-style` steering |

## Other references (not licensed material, cited as sources)

Learn Go with Tests, ThreeDotsLabs Wild Workouts, Google Engineering Practices, Software Engineering at Google, Michael Nygard's ADR article, MADR, the Go and AWS documentation. Links are in each skill's "Sources" section and in `docs/references.md`.

## MIT license notice (applies to the inspirations above)

Copyright (c) the respective authors of the projects listed above.

Permission is hereby granted, free of charge, to any person obtaining a copy of this software and associated documentation files (the "Software"), to deal in the Software without restriction, including without limitation the rights to use, copy, modify, merge, publish, distribute, sublicense, and/or sell copies of the Software, and to permit persons to whom the Software is furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.
