# MangoHud — Super I/O fan support (fork changes)

This is a fork of [MangoHud](https://github.com/flightlessmango/MangoHud) that adds
the ability to read **chassis / CPU / system fan RPM from Super I/O chips** via the
Linux `hwmon` interface, and to display them in the overlay.

This file documents *only* what differs from upstream MangoHud. For everything else,
see the main [`README.md`](README.md).

---

## Why

Stock MangoHud can only show two kinds of fan:

- `gpu_fan` — the GPU's own fan (AMD RPM, or NVIDIA percentage)
- `fan` — the **Steam Deck** APU fan only

It has no way to read the **motherboard fan headers** (CPU fan, case fans, etc.).
On a desktop those are wired to a Super I/O chip (Nuvoton, ITE, Fintek, …) which the
kernel exposes through `hwmon`. This fork teaches the existing `fan` element to read
those chips, with optional manual sensor selection for multi-fan boards.

---

## What changed

| Area | Upstream | This fork |
|------|----------|-----------|
| `fan` element | Steam Deck only | Steam Deck **and** auto-detected Super I/O chip |
| Sensor selection | n/a | New `fan_custom_sensor` option to pick exact chip + input(s) |
| Multiple fans | n/a | Multiple labelled fans via `fan_custom_sensor` |

### Touched source files
- `src/overlay.cpp` — `find_active_fan_input()` helper and the rewritten `update_fan()`
  auto-detection (Steam Deck → Super I/O).
- `src/overlay_params.cpp` / `.h` — `fan_custom_sensor` parameter and its parser
  (`parse_fan_custom_sensor`).
- `src/hud_elements.cpp` / `.h`, `src/overlay.h` — overlay plumbing for rendering the
  resolved fan(s).

---

## Usage

Config lives in `~/.config/MangoHud/MangoHud.conf`.

### 1. Auto-detect (simple case)

```ini
fan
```

`update_fan()` resolves the sensor **once** on first frame, then refreshes its RPM
every update. Auto-detect order:

1. **Steam Deck** — an hwmon named `steamdeck_hwmon` (uses `fan1_input`).
2. **Super I/O** — the first hwmon whose `name` starts with a known prefix:

   | Vendor   | Prefixes matched                        |
   |----------|-----------------------------------------|
   | Nuvoton  | `nct61`, `nct67`, `nct77` (incl. nct6799) |
   | ITE      | `it86`, `it87`                          |
   | Fintek   | `f7188`, `f8000`                        |
   | Winbond  | `w836`, `w8362`, `w8377`                |
   | SMSC     | `smsc`, `sch311`                        |

   Within that chip it picks the **first spinning fan** — the lowest-numbered
   `fanN_input` reporting a nonzero RPM (`find_active_fan_input()`).

> **Caveat:** auto-detect only finds one fan, and only one that is *already spinning*
> when the overlay starts. If the fan you want is idle at launch, or you have several
> fans, use `fan_custom_sensor` below.

### 2. Manual selection (recommended for desktops / multi-fan)

```ini
# single fan
fan_custom_sensor=nct6799,fan2_input

# multiple fans, with labels
fan_custom_sensor=nct6799,fan2_input,CPU;nct6799,fan7_input,GPU
```

Format — one or more entries separated by `;`:

```
chip,input[,label]
```

- **chip** — matched against the hwmon `name` file (e.g. `nct6799`).
- **input** — the raw sysfs filename (e.g. `fan2_input`).
- **label** — optional display label. Defaults to `FAN` for a single sensor, or
  `FAN1`, `FAN2`, … when several are configured.

When `fan_custom_sensor` is set, auto-detection is skipped and one overlay entry is
created per configured sensor.

---

## Finding your chip name and inputs

List the hwmon chips and their live fan readings:

```sh
for d in /sys/class/hwmon/hwmon*; do
  n=$(cat "$d/name" 2>/dev/null)
  echo "== $n  ($d)"
  for f in "$d"/fan*_input; do
    [ -e "$f" ] && echo "   $(basename "$f") -> $(cat "$f") rpm"
  done
done
```

Use the printed `name` as **chip** and the `fanN_input` filename as **input** in
`fan_custom_sensor`.

If no Super I/O chip shows up, load the right kernel module (e.g.
`sudo modprobe nct6775` for Nuvoton) — and run `sudo sensors-detect` from
`lm_sensors` once if you haven't.

---

## Troubleshooting

- **Nothing shows:** run with `MANGOHUD_CONFIG=fan` and check the debug log —
  `update_fan()` emits `fan: using ...` / `fan: no fan sensor found` via `SPDLOG_DEBUG`
  (build/run with debug logging enabled).
- **Wrong fan picked:** auto-detect grabbed the first spinning fan — switch to an
  explicit `fan_custom_sensor`.
- **RPM reads `-1`:** the sysfs path returned empty or non-numeric; double-check the
  exact `fanN_input` filename for that chip.
