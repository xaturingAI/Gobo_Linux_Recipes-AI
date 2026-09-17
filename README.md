These are Recipes for Gobo Linux 

ALL OF THESE WHERE MADE BY AI AMD TESTED BY ME THE (HUMAN) 

AI WHERE USED TO MAKE so I have chosen not upload to main GoBo Recipes  Unless the MAIN solo Dev are okay with intell then this will remain on my own GitHub page 





















# Recipes/Plasma6-core — KDE Plasma 6.7 desktop stack (single flat dir)

Home for EVERY recipe needed to build KDE Plasma 6.7 (Qt 6.10 + KDE Frameworks
6.26 + Plasma 6.7.x + Gear 26.08) as the second full desktop on the ISO.

This is a FLAT directory — there are no tier subdirectories. Every recipe lives
directly here as `Recipes/Plasma6-core/<Program>/<ver>/Recipe`, mirroring the
`Recipes/Pantheon/` convention (one directory per program, a version
subdirectory per recipe). The four build phases (Qt6, KF6, Plasma6, Gear) are
tracked as ORDER inside `build.sh`'s `step_plasma`, not as directories.

Full build strategy, module ordering and wiring live in `PLASMA-6.7.md` at the
repo root (mirrors `QT5.15-MIGRATION.md`). This file defines the layout only.

## Layout

```
Recipes/Plasma6-core/
├── qtbase/                 Qt 6.10.x module (program-per-module, one recipe)
├── qtshadertools/
├── qtdeclarative/
├── qt5compat/
├── qtsvg/
├── qtwayland/
├── qttools/
├── qtimageformats/
├── KCoreAddons/            KDE Frameworks 6.26.0 module
├── KConfig/
├── KI18n/
├── ...
├── kwayland/               Plasma 6.7.4 module (kwin, workspace, desktop, ...)
├── kwin/
├── plasma-workspace/
├── plasma-desktop/
├── ...
├── Dolphin/                Gear 26.08.0 app
├── Konsole/
└── ...
```

The prefix/program naming tells you which phase a recipe belongs to for the
ordering in `step_plasma`; every recipe also pins its Qt6 prefix (`/Programs/Qt6`).

## Conventions (every recipe)

- `recipe_type=cmake` (Qt6/KF6/all Plasma modules use CMake + Ninja).
- Qt recipes install to `Programs/Qt6/<ver>` (NOT `/Programs/Qt`): full
  coexistence with the SDDM-era 5.15.2 tree, since the two have incompatible
  binary ABIs and libQt5*/libQt6* SONAMEs.
- KF6/Plasma recipes reuse the ECM cmake dance from
  `Recipes/Extra-CMake-Modules/5.115.0/Recipe`: `-DCMAKE_PREFIX_PATH=/System/Index`
  `-DCMAKE_INSTALL_LIBDIR=lib` `-DCMAKE_SKIP_RPATH=True`
  `-DCMAKE_BUILD_TYPE=Release`, tests/docs/examples off wherever the module
  exposes that toggle (BUILD_TESTING=OFF; -DBUILD_DOCS... etc.).
- Every recipe documents in its header (a) which upstream dep it needs from
  the ISO/core/pantheon trees and (b) the KF6/Plasma modules it hard-requires,
  so `CheckDependencies` failures are greppable rather than guessable.
- Plasma 6.7 is the last X11-capable release and boots Wayland-first; kwin's
  Wayland backend is the target. Do NOT disable Wayland in any Qt6/kwin recipe.
- The ISO and all Plasma packages are systemd-FREE: link elogind, never
  libsystemd. Any recipe that `find_package(PkgConfig)`s a `systemd` feature
  must have it disabled or the dep dropped.

## build.sh wiring

`step_sync` syncs this whole flat dir in one glob loop:
`for r in /mnt/repo/Recipes/Plasma6-core/*/; do rm -rf "Recipes/$(basename $r)";
cp -a "$r" "Recipes/$(basename $r)"; done`. A new `step_plasma` step iterates
the topo order from `PLASMA-6.7.md` (`compile`/`package` per recipe) across the
four phases in one flat list. See the plan doc.
