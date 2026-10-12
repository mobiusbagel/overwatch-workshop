# Overwatch Workshop

Custom game modes built in the Overwatch Workshop.

## Modes

### Items & Inventory (in build)
Health-pack gameplay replaced with an item/power-up system: map spawners
dispense held items, players use them on demand from a 1-slot inventory, and
deaths drop the held item as contestable loot — teammates get a first pickup
window, then it's free-for-all, then it expires.

- Build doc: `modes/items-and-inventory/build-doc.md` (full rule spec, variables,
  tuning constants, testing checklist)
- Script text: `modes/items-and-inventory/workshop-script.txt` (paste a
  "Copy Settings" export here)
- Share code: TBD
- Status: build doc complete, in-editor build in progress

## Tools

### Placer
Companion preset for placing item spawners: join a Skirmish lobby, get
auto-assigned Jetpack Cat, fly the map dropping pins, export the coordinates
for the build doc. Spec lives in the Items & Inventory build doc
("Placer tool" section).

- Script text: `tools/placer/workshop-script.txt`

## Repo layout

```
README.md
modes/<mode-name>/
  build-doc.md          design + full Workshop rule spec
  workshop-script.txt   "Copy Settings" text export (durable backup)
tools/<tool-name>/
  workshop-script.txt
```

## Using a mode

- **Share code** (easiest for players): custom game lobby → Settings → Import
  Code → enter the code.
- **Script text**: copy the contents of `workshop-script.txt` → lobby Settings →
  Workshop → Import Settings → paste.
