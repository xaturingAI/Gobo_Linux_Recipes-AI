# Pantheon Stack Port — Recipe Tracker

elementary OS 8 Pantheon desktop stack for the GoboLinux gaming LiveCD.
Foundation is **GNOME 48** (Mutter 48.7, GLib 2.84.4, GTK 4.20.0 — bumped from
4.18.6 so libadwaita 1.8.0's GTK >= 4.19.4 hard requirement is met). The ISO is
sysvinit-based — **no systemd anywhere**: every session-stack package is pinned
so it builds with `systemd=false` (or the equivalent). Builds run via
`sudo /home/nohearth/gobo-build/run-build.sh all`; the order below is wired into
`step_pantheon()` in `/home/nohearth/gobo-build/build.sh` (a topo sort of each
recipe's `Resources/Dependencies`).

## 1. GNOME 48 foundation (26 recipes, 7 layers)

| Layer | Recipes |
|---|---|
| GLib core | GLib 2.84.4 |
| Text stack | HarfBuzz 11.4.0 → Fribidi 1.0.16 → Pango 1.56.1 |
| GLib satellites | JSON-GLib 1.10.0 → GdkPixbuf 2.42.12 → Graphene 1.10.8 |
| Math / input | LCMS2 2.19.1 → Libei 1.3.901 → Libdisplay-Info 0.3.0 |
| Settings / infra | GSettings-Desktop-Schemas 48.0 → Sysprof-Capture 48.0 → Libwacom 2.16.0 |
| Graphics / logind | **LibInput 1.31.3** → **Elogind 257.16** → **LibGLVnd 1.7.0** → **LLVM 19.1.7** → **Mesa 25.3.6** |
| Session infra | LibPipewire 1.4.0 → LibCanberra 0.30 → **LibGudev 238** |
| GTK / compositor | Colord 1.4.8 → Gnome-Desktop 44.5 → GTK4 4.20.0 (introspection ON) → LibYaml 0.2.5 → LibXmlb 0.3.29 → AppStream 1.0.6 → **Vala 0.56.19** → LibAdwaita 1.8.0 (introspection+vapi ON) → **Mutter 48.7** |

Build order (as in `step_pantheon()`):

```
GLib 2.84.4 → HarfBuzz 11.4.0 → Fribidi 1.0.16 → JSON-GLib 1.10.0 → Pango 1.56.1
→ GdkPixbuf 2.42.12 → Graphene 1.10.8 → LCMS2 2.19.1 → Libei 1.3.901
→ Libdisplay-Info 0.3.0 → GSettings-Desktop-Schemas 48.0 → Sysprof-Capture 48.0
→ Libwacom 2.16.0 → LibInput 1.31.3 → Elogind 257.16 → LibGLVnd 1.7.0
→ LLVM 19.1.7 → Mesa 25.3.6 → LibPipewire 1.4.0 → LibCanberra 0.30 → LibGudev 238
→ Colord 1.4.8 → IsoCodes 4.17.0 → LibSeccomp 2.6.1 → **ATK 2.36.0 → GTK+ 3.24.43**
→ Gnome-Desktop 44.5 → GTK4 4.20.0 → LibYaml 0.2.5 → LibXmlb 0.3.29
→ AppStream 1.0.6 → Vala 0.56.19 → LibAdwaita 1.8.0 → Mutter 48.7
```

**ATK + GTK+ are deliberately early** (before Gnome-Desktop, the first gtk+-3.0
consumer): GTK3's meson hard-requires `atk >= 2.35.1` and falls back to the
bundled atk subproject when the ISO's 2.34.1 fails, corrupting the generated
`gtk+-3.0.pc` `Requires:` line with a stringified `dep<id>`. Building ATK 2.36.0
then GTK+ 3.24.43 here guarantees every later consumer sees a clean pc; it also
provides `gdk-wayland-3.0.pc` early for the whole GTK3 stack. GTK3 is
independent of GTK4/Mutter, so nothing forbids the early placement.

Key ordering constraints:

- **GTK4 introspection + LibAdwaita vapi** (for valac): every GTK4 elementary
  app does `--pkg libadwaita-1`, which is only generated from libadwaita's GIR
  (vapigen, `gnome.generate_vapi`). libadwaita's GIR `includes: ['Gio-2.0',
  'Gtk-4.0']`, so GTK4 must be rebuilt with `-Dintrospection=enabled`
  (Gtk-4.0.gir) AND Vala must be built BEFORE LibAdwaita (vapigen). Both GTK4
  and LibAdwaita are rebuilt in place (`compile_keep`, tarball wiped) once
  their recipes flip those options on.

- **LibInput 1.31.3** (bump from ISO 1.26.2): Mutter 48.7 native backend needs
  libinput ≥ 1.27. Now `-Dlibwacom=true` (LibWacom 2.16.0 is in the foundation).
- **Elogind 257.16**: logind for the sysvinit ISO — Mutter hard-requires a
  libsystemd *or* libelogind. Built with `-Ddbus=enabled -Dpam=enabled`, docs/
  nss/userdb/varlink/polkit/selinux/audit/acl all disabled. Daemon startup is
  wired into the Gobo boot via `Resources/Tasks/Elogind` (symlinked into
  `/System/Tasks` by SymlinkProgram) plus a BootUp append done by `build.sh` —
  a recipe PostInstall cannot write outside the program tree (Run_PostInstall
  runs in a UnionSandbox that discards such writes). PAM headers from the
  ISO's Linux-PAM are linked into `/System/Index/include/security` by
  `build.sh` (the ISO program never linked them, and elogind's `pam_elogind`
  module needs them).
- **LibGLVnd 1.7.0** (bump from ISO 1.2.0): Mesa 25.3.6 with `-Dglvnd=enabled`
  requires libglvnd ≥ 1.3.2, and the NVIDIA driver ships glvnd ICDs
  (`egl_vendor.d/10_nvidia.json`), so the dispatch architecture is kept.
- **LLVM 19.1.7 → Mesa 25.3.6**: Mesa needs LLVM ≥ 18 (radeonsi/llvmpipe) and
  supplies `gbm.pc ≥ 21.3` for Mutter native. LLVM built as a shared dylib with
  X86+AMDGPU targets. Mesa drivers: llvmpipe + radeonsi, Vulkan off (NVIDIA
  provides its own), `-Dglvnd=enabled`. Build-time dep **python-mako** is
  installed via Alien PIP3 at the top of step_pantheon.
- **LibGudev 238 before Mutter**: Mutter requires `gudev-1.0 >= 238`. The ISO's
  Eudev ships `udev.pc`/`libudev.pc` but **no gudev.pc** → LibGudev is a recipe.
- **LibCanberra gtk3 enabled** (`--enable-gtk3`): gsd 48.1
  color/media-keys/power plugins hard-require `libcanberra-gtk3.pc`.
- LCMS2 is **2.19.1** (BUILD.md says 2.17.0 — stale).
- UPower, Colord and gsd all need `LibGudev`; gsd additionally needs
  `libgweather4`, `geocode-glib`, `geoclue`, `libwacom`, `libcanberra-gtk3`.

## 2. Session stack (14 recipes, no libsystemd)

```
LibNghttp2 1.70.0 → Libsoup3 3.6.6 → GweatherLocations 2026.2 → Gcr3 3.41.2
→ GeocodeGlib 3.26.4 → Libgweather4 4.6.0 → Geoclue 2.8.1 → UPower 1.90.9
→ Polkit 126 → GnomeKeyring 48.0 → GnomeSettingsDaemon 48.1 → GnomeSession 45.0
→ DesktopFileUtils 0.27 → SessionSettings 8.1.0
```

| Recipe | Notes |
|---|---|
| LibNghttp2 1.70.0 | configure `--enable-lib-only` (+ `--disable-dependency-tracking`) |
| Libsoup3 3.6.6 | gssapi/ntlm/brotli/introspection/vapi/docs off |
| GweatherLocations 2026.2 | data-only; GitHub `GNOME/gweather-locations` tag tarball |
| Gcr3 3.41.2 | `-Dgtk=false -Dssh_agent=false -Dsystemd=disabled` → gck-1 + gcr-base-3 |
| GeocodeGlib 3.26.4 | `-Dsoup2=false` |
| Libgweather4 4.6.0 | gtk_doc/introspection/vala/tests off |
| Geoclue 2.8.1 | 3g/cdma/modem-gps/nmea/compass/demo-agent off; `-Dsystemd-system-unit-dir=no` |
| UPower 1.90.9 | `-Dsystemdsystemunitdir=no -Dpolkit=disabled -Didevice=disabled -Dos_backend=linux` |
| Polkit 126 | **full polkitd build** now that Duktape 2.7.0 is on the ISO: `-Dlibs-only=false`, `-Dsession_tracking=elogind`, `-Dauthfw=shadow`; emits polkit-gobject-1.pc AND /usr/lib/polkit-1/polkitd (built AFTER Duktape) |
| GnomeKeyring 48.0 | `-Dsystemd=disabled -Dssh-agent=false -Dpam=false`; links gck-1 + gcr-base-3 |
| GnomeSettingsDaemon 48.1 | `do_patch()` sed-drops the two Linux hard asserts; `-Dsystemd=false -Delogind=false -Dnetwork_manager=false -Drfkill=false -Dsmartcard=false -Dusb-protection=false -Dwayland=false -Dwwan=false -Dgcr3=false`; alsa/gudev/cups/colord true. Autostart .desktop for all 16 plugins generated by plugins/meson.build |
| GnomeSession 45.0 | **pinned 45.0** (46+ requires libsystemd); `-Dsystemd=false -Dsystemd_session=disable -Dsystemd_journal=false -Dconsolekit=false` |
| DesktopFileUtils 0.27 | provides `desktop-file-install` used by session-settings |
| SessionSettings 8.1.0 | `do_patch()` drops onboard+orca; `-Dsystemd=false -Ddetect-program-prefixes=false -Dfallback-session=gala`; needs gsd.pc + desktop-file-install + gnome-keyring-daemon at build time |

## 3. elementary libraries + themes (support recipes)

| Recipe | Version | Notes |
|---|---|---|
| ElementaryIconTheme | 8.2.0 | theme data |
| ElementaryWallpapers | 8.0.0 | wallpaper data |
| GtkThemeElementary | 8.2.2 | GTK theme |
| SoundThemeElementary | 1.1.0 | sound data |
| Contractor | 0.3.5 | GTK3 contract API |
| Granite | 6.2.0 | **GTK3** — Wingpanel/switchboard-plugs-era lib |
| Granite7 | 7.8.1 | **GTK4** — Gala/Switchboard/GTK4 apps; built `-Ddemo=false` (demo needs shumate-1.0, not shipped) |
| Libgee | 0.20.8 | Vala collections |
| Libhandy | 1.8.3 | GTK3 adaptive widgets (PantheonFiles) |
| Vala | 0.56.19 | compiler (needed by every app). Built EARLY (before LibAdwaita): libadwaita's vapi needs vapigen |
| ATK | 2.36.0 | final standalone ATK (meson, introspection ON). ISO ships 2.34.1 but GTK+ 3.24.43 hard-requires `atk >= 2.35.1`; built in the foundation, immediately before GTK+ 3.24.43 and before any gtk+-3.0 consumer (see §1) |

## 3.5 Restored-app dependency tiers (26 recipes, full desktop scope)

The 7 apps dropped from the original plan (Calendar, Mail, Tasks, Code, Photos,
SettingsDaemon, CapnetAssist) are restored **with their full dependency chains**.
Their meson deps resolve against the index pcs, so the app recipes themselves are
unchanged; all new work is these support recipes. Ordered as wired in
`step_pantheon()` after the flatpak chain:

```
# Code tier
LibGit2 1.8.4 (cmake, -DUSE_SSH=OFF -DUSE_HTTPS=OpenSSL) → LibGit2-Glib 1.2.0
GtkSourceView 4.8.4   (meson, gir+vapi on)
LibPeas 2.0.4         (meson, pure-GLib 2.x; gjs/lua51/python3 off)

# WebKit tier (GTK3 4.1 API for Mail, GTK4 6.0 API for CapnetAssist)
LibHyphen 2.8.8       (configure; webkit FindHyphen resolves /usr->Index, no pc)
Unifdef 2.12          (plain makefile; install_variables=(prefix=$target))
WebKit2GTK 2.46.5     (cmake, USE_GTK4=OFF -> webkit2gtk-4.1; USE_GBM=OFF,
                       gamepad/introspection/docs/minibrowser OFF)
WebKitGTK 2.46.5      (cmake, USE_GTK4=ON -> webkitgtk-6.0; same flags)
  — one tarball, two program trees; NO libwpe/wpebackend-fdo needed (2.46 GTK
    port has zero wpe references); GStreamer 1.26 + plugins-base + good + libav
    above (playback codecs).

# Media playback tier (added for actual playback on the desktop ISO)
GStreamer 1.26.1 + Gst-Plugins-Base 1.26.1 (dev pcs for Photos/WebKit/Videos)
Gst-Plugins-Good 1.26.1   (meson; flac/mpg123/jpeg/png/gdk-pixbuf/cairo/
                           gtk3/ximagesrc/soup(libsoup3) on; taglib/vpx/pulse/
                           jack/lame/oss/shout2/v4l2/qt off — ISO's TagLib is
                           static-only with a pc missing -lz, so its taglib
                           plugin fails the final link; only tag WRITING is
                           lost, id3demux read-side is separate)
Gst-LibAV 1.26.1          (meson; links the ISO's OWN FFmpeg 4.2.2 — libavcodec
                           58.54.100 etc. satisfy gst-libav's declared minimums,
                           so the meson 'FFmpeg' fallback subproject never
                           fetches; gives H.264/AAC/MPEG/VP8/VP9/WebM decode,
                           so WebM plays without libvpx. do_patch guards the 4
                           unguarded AV_CODEC_ID_VVC refs (H.266 is FFmpeg >=
                           6.0; ISO has 4.2.2, same guard style as upstream's
                           QOI))
  — build order: core → plugins-base → good → libav, right after plugins-base
    and before VTE.

# EDS / clutter tier
LibIcal 3.0.19        (cmake, ICAL_GLIB=ON -> libical-glib >= 3.0.7 for EDS;
                       ISO only ships libical 2.x)
Cogl 1.22.8 → Clutter 1.26.4 → ClutterGtk 1.8.4 → LibChamplain 0.12.21
EvolutionDataServer 3.54.3 (cmake; GTK on, goa/weather/canberra/tests off,
                       introspection off, vala bindings on -> libedataserver-ui
                       + .vapi for Calendar/Mail/Tasks)
Folks 0.15.9          (meson; eds_backend only, telepathy/bluez/ofono off —
                       dbus-glib-1.pc absent on ISO)

# Photos tier
LibExif 0.6.24 → LibRaw 0.21.3 → LibGphoto2 2.5.31 → Exiv2 0.28.5 → GExiv2 0.14.3
  — libexif/LibRaw must use the release/data tarballs that ship the generated
    `configure` (github tag archives don't). Exiv2 with BROTLI/INIH off
    (no libbrotli on ISO). Photos' `dependency('gexiv2-0.16','gexiv2')` falls
    back to gexiv2.pc (0.14.3). LibGphoto2 links LibExif (order matters).

# SettingsDaemon tier
Jansson 2.14 (cmake) → PackageKit 1.3.6 (meson, -Dpackaging_backend=dummy;
                 jansson >= 2.8 is hard-required and absent on ISO)
LibJcat 0.2.3 (meson, gpg+pkcs7 via gpgme from flatpak chain, ed25519 off)
→ Fwupd 1.9.27 (meson; no systemd/elogind/lvfs/passim/bluez/cbor/consolekit;
                 -Defi_binary=false; xmlb 0.3.29 already built for AppStream)
```

Key facts:

- **WebKit 2.46.5 = one API per build**: `USE_GTK4=OFF` → `webkit2gtk-4.1.pc`
  (Mail), `USE_GTK4=ON` → `webkitgtk-6.0.pc` (CapnetAssist). Same tarball, two
  `--version`-suffixed program trees; index repoints per-pc.
- **`-DUSE_GBM=OFF`** for webkit (libgbm.pc missing on ISO; libdrm stays ON),
  `-DENABLE_GAMEPAD=OFF` (no libmanette), `-DENABLE_INTROSPECTION=OFF` on the
  6.0 build only (4.1 build keeps it ON for Mail's gir).
- **Code**: needs `vte-2.91.pc` (GTK3) → VTE recipe now builds both backends
  (`-Dgtk3=true -Dgtk4=true`); `libvala-0.56` present on ISO (valac 0.56.19),
  no LibVala build needed.
- **Stage archives**: tarballs are seeded into `/Data/Compile/Archives` under
  the recipe-url basenames (github refs tags: `v2.8.8.tar.gz`, `v1.8.4.tar.gz`,
  `v0.28.5.tar.gz`, `0.2.3.tar.gz`, `1.9.27.tar.gz`, `v1.3.6.tar.gz`,
  `v2.14.tar.gz`) so every tier builds offline — `stage_archives()` in
  `build.sh` (called by `step_sync`) copies them from `/mnt/gobo-build`.
- **ATK 2.36.0 must precede GTK+ 3.24.43**: GTK3's meson `dependency('atk',
  '>= 2.35.1')` falls back to the bundled `atk` subproject when the ISO's
  2.34.1 fails the check, and the subproject dependency stringifies as
  `dep<id>` into the generated `gtk+-3.0.pc` `Requires:` line — breaking every
  downstream pkg-config consumer (first symptom: VTE). Building ATK 2.36.0
  makes GTK+ use the system atk.pc and emit clean pcs. If this ever recurs,
  rebuild GTK+ 3.24.43 with **`compile_keep`** (not plain compile — the plain
  re-install's safe-copy dangles `/usr/include/gtk-3.0` and meson install dies
  ENOENT) after ATK is on the index.

## 4. elementary OS 8 apps (63 recipes)

Build order (as in `step_pantheon()`), split GTK3/Granite6 vs GTK4/Granite7:

- **Data/defaults**: PantheonDefaultSettings 8.1.1 (gsettings data, glib only)
- **GTK3 + Granite6 apps** (before Wingpanel, after Granite): PantheonCalendar
  8.0.2, PantheonCode 8.3.2, PantheonFiles 7.3.2, PantheonGeoclue2Agent 1.0.6,
  PantheonMail 8.0.1, PantheonPhotos 8.0.2, PantheonSettingsDaemon 8.5.0,
  PantheonSideload 6.3.1, PantheonTasks 6.3.3, PantheonVideos 8.0.2
- **GTK4 + Granite7 apps**: Granite7 → PantheonCalculator 8.0.1 (src/meson.build
  does `dependency('granite-7')` unconditionally), PantheonDefaultSettings 8.1.1
  (gsettings data; pre_build drops the unconditional accountsservice subdir —
  AccountsService not on the ISO, nothing references it), PantheonNotifications
  8.1.2 (gtk4-wayland),
  PantheonWayland 1.1.0, CapnetAssist 8.0.2,
  PantheonCamera 8.0.2, PantheonMusic 8.1.0, PantheonOnboarding 8.1.0,
  PantheonPolkitAgent 8.1.0, PantheonScreenshot 8.0.4, PantheonShortcutOverlay
  8.1.0, PantheonTerminal 8.1.0
- **DROPPED: LightdmPantheonGreeter 8.1.2** — its src/meson.build requires
  `gdk-wayland-3.0` (needs a GTK+ wayland-backend rebuild) AND
  `liblightdm-gobject-1 >= 1.30.0` (no LightDM on the ISO). The DM is SDDM (its
  own QML greeter), so the pantheon greeter would never run. The GTK+ rebuild
  IS in the plan anyway: Wingpanel 8.0.4 hard-requires `gdk-wayland-3.0`
  (GTK+ 3.24.43 rebuilt with x11+wayland).
- **Shell**: Wingpanel 8.0.4 (GTK3) → PantheonApplicationsMenu 8.0.4 + 10
  WingpanelIndicators; Gala 8.5.1 (Mutter+Granite7); Switchboard 8.0.3 → 21
  SwitchboardPlug-*

Key ordering:

- **Gala 8.5.1** after Mutter + Granite7 + LibGee + Gnome-Desktop; built with
  `-Dsystemd=false` (uses elogind at runtime).
- **Wingpanel before its indicators**; **Switchboard before its plugs**.
- **PantheonPolkitAgent** after Polkit; **PantheonGeoclue2Agent** after Geoclue.

## 5. Cross-recipe dependency tree (session stack)

```
gsd 48.1 ─┬─ gnome-desktop-3.0, gsettings-desktop-schemas, gtk+-3.0, pango,
          │  libcanberra-gtk3, libpulse-mainloop-glib, libnotify (ISO)
          ├─ gweather4 (─ gweather-locations, geocode-glib(─ libsoup3), libxml2)
          ├─ geocode-glib-2.0 / libgeoclue-2.0 (─ Geoclue ─ gudev)
          ├─ upower-glib (─ UPower ─ gudev, eudev)
          ├─ polkit-gobject-1 (─ Polkit libs-only)
          ├─ gudev-1.0, libwacom (─ LibGudev, Libwacom)
          ├─ cups, colord (ISO + Colord)
          └─ (rfkill/nw/wwan/smartcard/wayland plugins DISABLED)
gnome-session 45.0 (─ gtk+-3.0, gnome-desktop, gsettings)   [no libsystemd]
session-settings 8.1.0 (─ gsd.pc, desktop-file-install, gnome-keyring-daemon, vala)
Mutter 48.7 (─ elogind, mesa ≥ 21.3, gudev >= 238, libdisplay-info, libei,
              libinput ≥ 1.27, libwacom, colord, gnome-desktop)
```

## 6. Status

- [x] 26 foundation recipes authored & md5/size verified (added LibInput 1.31.3,
      Elogind 257.16, LibGLVnd 1.7.0, LLVM 19.1.7, Mesa 25.3.6)
- [x] 15 session-stack recipes authored & md5/size verified
- [x] 63 elementary + 4 support recipes authored & md5/size verified
- [x] 26 restored-app support recipes authored & md5/size verified (§3.5: webkit
      ×2, EDS/clutter tier, code tier, photos tier, settings-daemon tier, plus
      LibIcal/Jansson/LibHyphen/Unifdef); all 26 tarballs staged+verified; app
      recipes unchanged (deps resolve via index pcs)
- [x] `step_sync` loops + `stage_archives()` wired; `step_pantheon()` tier order
      + restored apps inserted; `bash -n` clean
- [x] VTE recipe now dual-backend (`-Dgtk3=true -Dgtk4=true`) for Code's vte-2.91
- [x] VTE built `-Dicu=false`: the ISO's LibICU4C 65.1 is a reduced build lacking
      `ucnv_clone` (VTE's unguarded icu-glue.cc uses it); icu=false drops the
      ICU sources and VTE uses libc/glib charset decoding instead
- [x] VTE 0.80.1 built + packaged with the icu fix (build now passes it)
- [x] Gst-Plugins-Good 1.26.1 + Gst-LibAV 1.26.1 recipes added (playback codecs:
      MP3/FLAC/WAV + libsoup3 from good; H.264/AAC/VP8/VP9/WebM via gst-libav
      linked to the ISO's own FFmpeg 4.2.2); both wired into `step_sync`,
      `stage_archives`, and `step_pantheon` right after Gst-Plugins-Base
- [x] Gcr4 4.3.1 recipe: `-Dintrospection=enabled` → `-Dintrospection=true`
      (gcr-4's option is a meson BOOLEAN, not a feature; `enabled` errored)
- [x] Gst-Plugins-Good 1.26.1 built + packaged (taglib disabled)
- [x] Gst-LibAV 1.26.1 built + packaged (VVC do_patch worked — linked the ISO's
      FFmpeg 4.2.2)
- [x] Gcr4 recipe: `-Dsystemd=false` → `-Dsystemd=disabled` (gcr-4's `systemd`
      is a FEATURE, not boolean; introspection/vapi/ssh_agent are boolean so
      their `true` values are correct; crypto defaults to libgcrypt — on ISO)
- [x] GTK4 4.20.0 gir failure diagnosed: `[1140/1249] Generating
      gtk/Gdk-4.0.gir` — `Couldn't find include 'Pango-1.0.gir'`. Index
      Pango-1.0.gir → Pango/1.44.7 (Feb-2020-era gir, GoboLinux 017 base) —
      that stale-tree dependency intermittently becomes unreachable during a
      rebuild (07:37 GTK4 build OK; 07:53 rebuild failed right after the
      GLib/Graphene/Libei in-place rebuilds at 07:47-48). Root files all
      resolve NOW (verified in-chroot with g-ir-scanner 1.84's own
      `os.path.exists`); failure was a transient broken symlink chain.
      **Fix: Pango recipe `-Dintrospection=disabled` → `-Dintrospection=enabled`**
      so Pango 1.56.1 ships its own fresh girs (built by g-ir-scanner 1.84,
      consistent with the rebuilt GLib 2.82.4 / Graphene stack), and
      RefreshIndex repoints Pango-1.0.gir/PangoCairo-1.0.gir away from 1.44.7.
- [x] Pango 1.56.1 in-place rebuild via `compile_keep` (build.sh:666): plain
      `compile`'s Pre_Installation_Preparation safe-copy dangled
      `/usr/include/pango-1.0` and meson install died ENOENT on
      `pango-enum-types.h` (same failure as GTK+ 3.24.43); `--keep` leaves the
      tree live and overlays the new introspection-enabled install
- [x] **Run `run-build.sh sync` FIRST** — chroot recipes were stale (Pango
      `-Dintrospection=disabled`, Gcr4 `-Dsystemd=false`); `step_sync` copies
      `/mnt/repo/Recipes` → chroot and runs `stage_archives`. `build.sh` edits
      take effect live (bind-mount); **`Recipes/` edits do NOT until `sync`**.
- [x] **GTK4 4.20.0 BUILT** (11:26, gir step passed!) after Pango introspection
      flip + sync. Pango 1.56.1 now ships its own girs
      (`Pango-1.0.gir` 1016547 B, scanner 1.84) and RefreshIndex repointed
      `Pango-1.0.gir`/`PangoCairo-1.0.gir` → 1.56.1 (links mtime 11:20).
      Run also rebuilt GLib/Graphene/Libei/Gnome-Desktop/LibAdwaita fine.
- [x] **Gcr4 4.3.1 built** with `-Dgpg_path=/bin/true` (skips the vestigial
      `find_program('gpg2','gpg')`; `GPG_EXECUTABLE` never referenced in source).
- [x] **Gpgme estream_t fix** (`export CPPFLAGS="$CPPFLAGS -DGPGRT_ENABLE_ES_MACROS"`):
      gpg-error.h 1.52 typedefs `estream_t` only under that macro but uses it
      unconditionally at :1727; old assuan.h 2.5.3 doesn't set it. Compile
      then succeeded fully.
- [x] **Gpgme bindings off** (`--enable-languages=`): install-time C++ relink
      (`libgpgmepp`) dies — ld can't open `/Programs/GCC/14.2.0/lib/crti.o`
      (base-ISO GCC tree only shipped `lib/gcc/`; path resolved via the
      gcc-version index symlink). No consumer needs gpgmepp/gpgme-qt.
- [x] **Gpgme recipe fixed (3rd issue)**: a recipe edit accidentally dropped
      the `compile_version/url/file_size/file_md5/dir/recipe_type` lines, so
      Compile died "Missing URL or repository information". Variables
      restored; `url` verified via `source` test. **Lesson: after editing a
      recipe, re-`source` it in bash and confirm `url` (or `git`) is set.**
- [x] **LibGpg-Error header patched (estream_t unconditional)**: LibOstree
      failed on the same `unknown type name 'estream_t'` as gpgme (its code
      includes gpg-error.h directly). Root cause is systemic: 1.52 guards
      `estream_t` behind `GPGRT_ENABLE_ES_MACROS` yet uses it unconditionally
      at :1727, and ISO-era libassuan 2.5.3 includes gpg-error.h without the
      macro. Rather than per-recipe CPPFLAGS, fixed at the single source:
      LibGpg-Error recipe `do_patch()` strips the guard in
      `src/gpg-error.h.in` (+ generated copies) so `estream_t` is always
      defined — matching upstream 1.37 and the header's own usage; `es_*`
      macros stay opt-in. Idempotent (tested). build.sh:899 now
      `rm tarball + compile_keep` so LibGpg-Error rebuilds every run.
      Also covers Flatpak (includes gpgme.h) and any future consumer.
- [x] **Flatpak pyparsing unblocked**: 1.16.2's vendored variant-schema-compiler
      (generates flatpak-variant-* headers, not optional) does
      `from pyparsing import *`; the ISO Python lacks it and the chroot has NO
      network (all tarballs are staged host-side via `stage_archives`), so
      `pip install` inside the chroot won't work. Vendored the pure-python
      wheel `pyparsing-3.3.2-py3-none-any.whl` into
      `Recipes/Flatpak/1.16.2/` (rides into the chroot via `step_sync`'s
      `cp -a`); build.sh now runs
      `python3 -m pip install --no-index --no-deps <wheel>` right before
      `compile Flatpak` unless `import pyparsing` already succeeds (idempotent).
- [x] **LibGit2 pinned 1.7.2 (down from 1.8.4)**: Flatpak + LibGit2 built; then
      LibGit2-Glib 1.2.0 failed: `git_error_set_str` implicit declaration.
      Root cause: libgit2 1.8 moved the `git_error_*` setters out of public
      `git2/errors.h` into deprecated `git2/sys/errors.h` (not included by
      git2.h), but ggit calls them under `#if LIBGIT2_VER_MAJOR > 0`. 1.7.2
      keeps them public (errors.h:166). New recipe `Recipes/LibGit2/1.7.2`
      (md5 fa4f30e5 / 7548186); 1.8.4 recipe removed; build.sh wipes the 1.8.4
      tree + tarball before compiling 1.7.2 so the index refresh resolves
      git2.h → 1.7.2 (only remaining candidate). ggit needs >= 0.25.0.
- [x] **Unifdef 2.12 release tarball**: LibGit2 tier fully built (LibGit2-Glib,
      GtkSourceView, LibPeas all OK); Unifdef failed in `scripts/reversion.sh`
      — "Your copy of unifdef is incomplete". reversion.sh exits 1 unless
      `version.sh` exists or `.git` is present; the github tag archive has
      neither (version.sh is gitignored). Switched the recipe to the author's
      release tarball (dotat.at/prog/unifdef/unifdef-2.12.tar.gz) which ships
      pre-generated version.sh + version.h (md5 b225312c / 87091, dir
      unifdef-2.12). stage_archives now force-refreshes the same-named stale
      github archive in the chroot (previously skipped by the `[ ! -f ]` guard).
- [x] **cmake recipes: `configure_options` → `cmake_options`**: WebKit2GTK failed
      at configure — "Please choose which WebKit port to build". Root cause:
      Gobo's cmake build type reads `cmake_options` (BuildType_cmake:
      `Combine_Arrays config cmake_options userconfigureoptions`), but the
      recipes used `configure_options`, so -DPORT=GTK etc. were silently
      dropped. Renamed the array in all 7 cmake recipes: WebKit2GTK, WebKitGTK,
      EvolutionDataServer, LibGit2, LibIcal, Exiv2, Jansson (Vulkan-*/ECM/SDDM/
      LLVM already used cmake_options). LibGit2 1.7.2 had built without options
      (no HTTPS, non-Release) — build.sh now wipes its tarball + compile_keep
      rebuilds it with USE_HTTPS=OpenSSL + Release; headers unchanged so
      LibGit2-Glib stays as-is.
- [x] **Ruby 3.3.12 recipe added**: WebKit configure failed "Could NOT find
      Ruby (missing: Ruby_EXECUTABLE ...) (Required is at least version 2.5)".
      WebKitCommon.cmake:199-205 hard-requires a Ruby >= 2.5 interpreter (JSC
      codegen); ISO ships none. New `Recipes/Ruby/3.3.12/Recipe` (autotools,
      cache.ruby-lang.org tarball md5 c6c276876f7f7fa1d40e47347928e9bd,
      `--disable-install-doc*`; PrepareProgram passes --prefix=$target). Wired
      into step_sync (before WebKit2GTK), stage_archives (ruby-3.3.12.tar.gz),
      and build.sh `compile Ruby 3.3.12 && package Ruby 3.3.12 &&` before the
      WebKit tier.
- [x] **Ruby `makefile='GNUmakefile'`**: first build died at install —
      "Nothing to be done for 'all'" then "No rule to make target 'install'".
      Gobo's configure/makefile build types run `make -f Makefile`, but Ruby's
      real targets (`all: $(SHOWFLAGS) main docs` common.mk:329, `install:
      install-nodoc` common.mk:468) live in common.mk, pulled in ONLY via
      GNUmakefile (`-include uncommon.mk` -> `defs/gmake.mk`); the generated
      Makefile has an empty `all:` and no install target. `make -f
      GNUmakefile` == upstream `make` (GNU make prefers GNUmakefile anyway).
- [x] **WebKit `-DENABLE_SPELLCHECK=OFF`**: configure died at
      OptionsGTK.cmake:373 "Enchant is needed for ENABLE_SPELLCHECK" (no
      enchant-2 on the ISO). Spellcheck is ON by default for the GTK port and
      no consumer in the stack uses webkit spellcheck (Code uses its own
      spellchecker). Audited every other hard dep too: Cairo, LibGcrypt 1.8.5,
      Libtasn1 4.16, HarfBuzz(+icu), ICU, JPEG, Epoxy 1.5.4, LibXml2, PNG,
      SQLite3, Unifdef, ZLIB, WebP, LibXslt, libsoup3, GTK(3+4), libsecret,
      X11/Wayland all present; speech synthesis defaults OFF (Flite skipped).
- [x] **WebKit extra OFF flags**: after spellcheck, configure failed at
      OptionsGTK.cmake:403 "libjxl is required for USE_JPEGXL". Audited all
      ON-by-default options; added to both WebKit recipes:
      `-DENABLE_PDFJS=OFF` (pdf.js would be downloaded at build time — no
      ThirdParty/ dir in the tarball, chroot is offline), `-DUSE_JPEGXL=OFF`,
      `-DUSE_AVIF=OFF`, `-DUSE_WOFF2=OFF`, `-DUSE_LIBBACKTRACE=OFF` (libs not
      on the ISO). Verified present: USE_LCMS (LCMS2), ENABLE_JOURNALD_LOG
      (libelogind.pc), ENABLE_BUBBLEWRAP_SANDBOX (runtime-only), gperf,
      OpenMP (GCC builtin), all unconditional pcs.
- [x] **`-DUSE_GSTREAMER_TRANSCODER=OFF`** (both WebKit recipes): configure
      failed at GStreamerChecks.cmake:51 "GStreamerTranscoder >= 1.20 is
      needed for USE_GSTREAMER_TRANSCODER". gst-transcoder is a separate
      project (gstreamer-transcoder-1.0.pc not built); flag defaults PUBLIC ON
      and MediaRecorder defaults ON, so the check fatals. MediaRecorder's only
      real GTK impl is behind USE(GSTREAMER_TRANSCODER), so the feature is
      inert without it. ENABLE_WEB_RTC defaults OFF (=
      ENABLE_EXPERIMENTAL_FEATURES=OFF), so gstreamer-webrtc (gst-plugins-bad,
      not built) is never probed.
- [x] **HarfBuzz 11.4.0 rebuilt WITH ICU**: WebKit configure reached the
      Generate step but died `HarfBuzz::ICU` / `HarfBuzz_ICU_INCLUDE_DIR-
      NOTFOUND`. Root cause: the first HarfBuzz 11.4.0 build used
      `-Dicu=disabled` (recipe wrongly claimed "ISO doesn't ship it") — so the
      11.4.0 tree has no harfbuzz-icu.pc, and WebKit's FindHarfBuzz (REQUIRED
      COMPONENTS ICU) fell back to the stale ISO 2.6.4 harfbuzz-icu.pc
      (prefix=/usr, ABI-mismatched lib). ISO actually ships LibICU4C 65.1
      (icu-uc/icu-i18n pcs). Recipe now `-Dicu=enabled`; build.sh wipes the
      no-ICU tarball + compile_keep rebuild.
- [ ] Remaining: WebKit tier → EDS tier → Photos tier →
      SettingsDaemon tier → 7 restored apps
- [x] ATK 2.36.0 recipe added (meson; ISO 2.34.1 fails GTK+ 3.24.43's `atk >=
      2.35.1` check → subproject fallback corrupted gtk+-3.0.pc); **ATK + GTK+
      3.24.43 moved into the foundation before Gnome-Desktop** (the first
      gtk+-3.0 consumer) so a corrupt pc on the index can never break the
      rebuild chain; GTK+ 3.24.43 built **in place via `compile_keep`** (a
      plain re-compile's safe-copy dangles the `/usr/include/gtk-3.0` symlink
      and meson install dies ENOENT on gdkenumtypes.h — same failure the
      GTK4/LibAdwaita rebuilds already hit), tarball dropped to force the run
- [ ] First full `sudo /home/nohearth/gobo-build/run-build.sh all` pantheon build

## 7. Follow-up (not part of the desktop session stack)

Flatpak: Bubblewrap done; Polkit libs + Libsoup3 done. Remaining: OSTree,
Flatpak, XDG-Desktop-Portal (+ libportal), DConf, libseccomp, gpgme,
libappstream-glib, and the polkitd (now built via Duktape) Flatpak for install
auth. AppImage already works via ISO FUSE.
