# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repository is

This is an **Obsidian documentation vault** (`OBSIDIAN.tareas`), not a code project. It is
the "engineering brain" for **ECOREX - Sistema de Tareas**: the planned rebuild on **.NET 10**
of a task/BPMN-workflow/dynamic-forms/rules manager that currently runs as a legacy
WebForms/VB.NET app (Bitcode's `GestionMovil` / `MotherData` platform).

There is no build, test, or lint step here. Deliverables are Markdown notes plus a couple of
self-contained HTML prototypes. The actual destination source code lives in a **separate repo**:
`https://github.com/alexandercuartas665/EcorexV.git`.

The vault schema mirrors a sibling vault, **CUBOT.nails**, which is the destination backbone
(Clean Architecture, dual DAL) that ECOREX is being cloned from and adapted.

## The one convention that governs almost every note: ORIGEN vs DESTINO

Every technical note deliberately separates two systems and must keep them distinct:

- **DESTINO** (what is being built): ASP.NET Core 10 / EF Core 10 / Blazor / dual DAL
  (PostgreSQL **or** SQL Server) / real multi-tenant (`TenantId` + `HasQueryFilter` + RLS).
  This is the primary subject — notes describe *what to build*.
- **ORIGEN** (the legacy system, kept as migration reference / ETL blueprint): WebForms +
  VB.NET + `MotherData`, multi-tenant by a `SUCURSAL` column. Preserved to describe *what the
  legacy did*, never as the target design.

When editing or adding a note, preserve this framing. Do not blur legacy behavior into the
destination spec. Section headers commonly use a `D#`/`O#` prefix scheme (D = destino, O = origen).

## Layout (read `00 - INDICE.md` first — it is the master index)

```
00 - INDICE.md                     Master index + reading paths by role. Start here.
01. Requerimiento/                 Spec by "capas" (layers) + Prototipo
    Capa 0 Vision General/         Vision, shell, cross-cutting engines
    Capa 1 Gestion de tenant/      Multi-tenant admin
    Capa 2 Tareas y Proyectos/     Operational core (task/project specs + UI)
    Capa 3 Flujos de Tareas BPMN/  Workflow engine (AdmWorkflow -> WorkflowEngine, .bpmn)
    Capa 4 Constructor de Formularios/  EAV form builder (reverse-engineered `Documental/`)
    Capa 5 Librerias Base/         MotherData DAL + shared functions
    Capa 6 Modulos Ecorex Especificaciones/  5 module specs, each with an HTML prototype
    Capa 7 Agentes de IA/          AI layer (assistant + config copilot)
    Prototipo/                     DEFINITIVE visual prototype (.NET 10 target look)
02. Inventario de modulos/         Inventory, module map, E-R model, detected SQL tables
03. Hoja de Ruta desarrollo/       Build roadmap, startup prompt, tech-debt backlog
04. Notas para desarrollador/      Data handling, security/auth, ADRs
05. Pruebas/                       Testing strategy (destino) + legacy validation (origen)
06. Deploy/                        Docker/hybrid, CI/CD, backup & DR
10. Historias de Usuario/          4 personas + end-to-end narrative
```

## Authoring conventions (follow these when writing notes)

- **ASCII only.** New notes are written in plain ASCII — no decorative Unicode box-drawing,
  arrows, bullets, or emojis. Use `=`, `-`, `|`, `+`, `#`, `*`, `-->`, `[OK]`, `[ERROR]`.
  This is enforced by the `/ascii-only` skill and applies to any script files (`.ps1`, `.bat`,
  `.sh`, `.cmd`) as well — including string literals.
- **Language is Spanish.** All prose is in Spanish (accents are used in prose despite the ASCII
  rule for structural characters). Match the surrounding note.
- **Wikilinks by filename only.** Links are `[[Nombre del archivo]]` (no folder paths), which
  keeps them immune to file moves. Use `[[Nombre|Alias]]` for display text. A link target that
  does not yet exist is acceptable (marks a note to write).
- **YAML frontmatter.** Notes carry structured frontmatter (`tipo`, `modulo_codigo`,
  `tabla_principal`, `estado`, `url_prod`, `stack_destino`, etc.). When adding a note, mirror
  the frontmatter shape of a peer note in the same layer.
- **Keep `00 - INDICE.md` in sync.** It has a per-layer link listing, a "Novedades (YYYY-MM-DD)"
  changelog, and header counters (`total_docs`, `nivel_completitud`). Update the relevant
  section when you add/rename/remove a note. Each folder also tends to have its own `00 - *.md`
  index; keep those consistent too.
- **Convert relative dates to absolute** in notes and changelog entries.

## Commit conventions

Commits follow `docs(<scope>): <summary>` in Spanish, e.g. `docs(tareas): ...`, `docs(capa4): ...`,
`docs(cred): ...`. Work is organized into numbered "Olas" (waves) and "PRE-#" prerequisites; the
history reads as a running log of which wave/prerequisite was completed.

## Security-sensitive content (do not leak into the public repo)

- Real credentials live **outside** the repo. `04. Notas para desarrollador/Notas para
  desarrollador Alexander.md` is gitignored precisely because it holds a real credential — never
  re-add it or paste its contents elsewhere.
- Notes about the legacy code annotate serious security flaws (embedded Azure secret in
  `cl_Transcripcion_audio.vb`, SQL by string concatenation, an unauthenticated AI endpoint with
  `CORS *`, open reflection = de-facto RCE). These are documented as risks **not to inherit** —
  describe them, do not reproduce secrets verbatim, and do not carry the patterns into destino specs.

## Prototype assets

`01. Requerimiento/Prototipo/` holds the definitive target look: `ECOREX.dc.html` + `support.js`
are the editable Claude Design source; `capturas/` and `screenshots/` are official per-screen
shots; `conceptos-claude-design/proto_*.html` are per-spec concept pages referenced by Capa 6.
`uploads/` and `.thumbnail` are gitignored Claude Design artifacts — do not version them.
