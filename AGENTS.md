Repository guidance for Codex agents working in `D:\retro\unpack`.

Purpose:
- This repository contains the `unpack` NitrOS-9 application, written in Basic09.
- Its goal is to unpack packed Basic09 modules back into Basic09 source code.
- When analyzing, modifying, or explaining this codebase, treat the local manuals below as primary references for Basic09 and NitrOS-9 behavior.

Primary reference documents:
1. `D:\retro\unpack\reference\Basic09 Programming Language Reference Manual_Microware.md`
2. `D:\retro\unpack\reference\Basic09 Programming Language Reference Manual - NitrOS9 EOU.md`
3. `D:\retro\unpack\reference\NitrOS9 EOU Technical Reference.md`

Instructions:
1. Before making assumptions about Basic09 syntax, semantics, runtime behavior, procedures, data types, arrays, strings, or module structure, consult the two Basic09 reference manuals.
2. Before making assumptions about NitrOS-9 module layout, system conventions, file handling, paths, memory, or OS-level behavior, consult the NitrOS-9 technical reference.
3. Use repository walkthrough files and implementation files together with the manuals:
   - `*-walkthrough.md` files explain current reverse-engineering intent.
   - `*.B09` files define actual program behavior.
4. If repository code and a walkthrough disagree, prefer the `*.B09` source, then validate interpretation against the manuals.
5. If the two Basic09 manuals differ, note the difference explicitly and avoid silent assumptions.
6. Mark speculation clearly when the manuals and code do not fully resolve behavior.

Recommended reading order for unfamiliar tasks:
1. `README.md`
2. Relevant `*-walkthrough.md`
3. Corresponding `*.B09` source files
4. The applicable manual sections in `reference\`

Expected working style:
- Be precise and minimal.
- Use the manuals as authoritative context for understanding Basic09 constructs and NitrOS-9 details in this project.
- Cite the relevant manual file when a conclusion depends on documented behavior rather than code inspection alone.
