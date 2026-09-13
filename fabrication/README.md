# Fabrication & assembly — Silkscreen (silkscreen_pcb)

Generated with KiCad 9.0.6 `kicad-cli` from `silkscreen_pcb.kicad_pcb` / `.kicad_sch`.
2-layer board, 60 × 111 mm, 1 oz copper. 173 components (163 placed, 6 DNP; +5 mounting holes, +5 test pads).

## What's tracked vs. generated

Tracked in git:
```
fabrication/
  README.md            this file
  BOM.md / BOM.csv      full BOM (MPNs, DigiKey/LCSC links, LCSC alternates, pricing)
  assembly/
    bom_jlcpcb.csv      JLC BOM format (Comment, Designator, Footprint, LCSC Part #)
    cpl_jlcpcb.csv      JLC placement (Designator, Mid X, Mid Y, Layer, Rotation)
    cpl_all.csv         raw KiCad centroid, both sides
```

**Not tracked** (regenerate on demand — see commands): `gerbers/` and the gerber `.zip`.

## Regenerate the gerber + drill set

One standard set uploads to **JLCPCB, PCBWay and NextPCB**:

```bash
kicad-cli pcb export gerbers -o fabrication/gerbers/ \
  --layers F.Cu,B.Cu,F.Mask,B.Mask,F.SilkS,B.SilkS,F.Paste,B.Paste,Edge.Cuts \
  --no-protel-ext --subtract-soldermask silkscreen_pcb.kicad_pcb
kicad-cli pcb export drill -o fabrication/gerbers/ --format excellon \
  --drill-origin absolute --excellon-units mm --excellon-separate-th silkscreen_pcb.kicad_pcb
```

Then zip `fabrication/gerbers/` and upload it. Board: 2-layer, 1.6 mm, 1 oz Cu; outline on `Edge.Cuts`.

## Assembly

Regenerate the centroid:
```bash
kicad-cli pcb export pos -o fabrication/assembly/cpl_all.csv --format csv --units mm --side both silkscreen_pcb.kicad_pcb
```

- **JLCPCB:** `bom_jlcpcb.csv` + `cpl_jlcpcb.csv` are in JLC's format, but the **LCSC column is blank**
  and rotations are raw KiCad angles. For the *final* JLC order, prefer the KiCad **Fabrication
  Toolkit** plugin — it applies JLC's per-package rotation database and auto-fills LCSC numbers.
  These files are a correct generic centroid and a cross-check.
- **PCBWay / NextPCB:** upload the gerber zip + a centroid (`cpl_all.csv`) + `BOM.csv`. They quote
  against the manufacturer part numbers, so the MPN/DigiKey BOM is what they need. Confirm the
  THT split with them (J1 USB-C, J5 JST, J6 header, buttons are through-hole).

## Caveats
- DNP parts (`R43 R45 R58 R66 R72 R74`) are excluded from the assembly BOM/CPL.
- Centroid uses the KiCad page origin (same as the gerbers); Y is negative (KiCad Y-up
  convention) — normal, houses import it directly.
- Regenerate all of the above whenever the board changes.
