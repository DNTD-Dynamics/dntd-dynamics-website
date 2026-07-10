# DNTD Dynamics — Session Context
*June 24, 2026*

---

## Session Overview

Long multi-topic session covering RadarGuard software completion, PCB BOM development, firmware questions, repo cleanup, and git workflow. The repo is now in a clean, shippable state with the standalone pipeline feature-complete for the first buyer experience.

---

## RadarGuard Software — What Was Built Today

### New Files
| File | Location | What it does |
|------|----------|--------------|
| `presence_hold.py` | `src/` | STOP-triggered static presence monitor. Latches STOP when person walks in and stops moving. Uses micro-Doppler sway detection (0.02–0.25 m/s) to hold STOP indefinitely until person leaves. IDLE → HOLDING → RELEASING state machine. |
| `arm_config_gui.py` | `src/` | Arm configuration GUI. Takes physical joint measurements in mm, writes `joint_geometry`, `joint_names`, zone distances, and sensor mount directly into `dntd_mmwave_config.yaml` via surgical regex replacement — all 146 comment lines preserved. |

### Updated Files
| File | What changed |
|------|-------------|
| `src/main.py` | Fixed wrong reader class (UARTReader → MmwaveReader), fixed frame processing path, wired in `StaticPresenceHold`, wired in optional `BackgroundModel` (`--bg-learn`), preserved `--occupancy-hold`, fixed fake ZoneState objects (now uses `dataclasses.replace()`). |
| `src/zone_logic.py` | Fixed `static_filter_mps` no-op bug — velocity gate was stored but never applied inside `_classify_points()`. Now enforced at classifier level regardless of how it's called. |
| `README.md` | Updated standalone pipeline section (StaticPresenceHold in diagram, new parameters table, static person detection section), corrected roadmap checkboxes, added arm config GUI to adapting section. |

### Key Design Decisions

**Static person detection approach:**
True cardiac heartbeat detection requires a dedicated slow-chirp firmware profile (TI vital-signs demo) — a completely different operating mode that would break the safety pipeline's range/velocity performance. Instead, `presence_hold.py` uses micro-Doppler sway detection: the involuntary low-amplitude movement (0.02–0.25 m/s) a stationary person always generates. Achieves the same safety property without a firmware profile change.

**Surgical YAML writer:**
First version of `arm_config_gui.py` used `yaml.safe_load()` + `yaml.dump()` which destroyed all 146 comment lines in `dntd_mmwave_config.yaml`. Replaced with targeted regex replacement — each key located by exact indentation pattern, value replaced, everything else byte-for-byte identical.

**Static presence hold + occupancy hold are complementary:**
Occupancy hold (`--occupancy-hold`, 1.5s default) bridges the gap between when points drop below the velocity filter and when `StaticPresenceHold` has enough sway history. Both are active — they're not redundant.

---

## CAUTION Speed Ramp

Arduino patch document created (`CAUTION_SPEED_RAMP_PATCH.md`). Three edits to `radarguard_arm_controller.ino`:
1. Add `CAUTION_SPEED_DIVISOR` constant (default 2 = half speed)
2. Add `currentZone` tracking variable
3. Update serial command handler with CAUTION branch

Arduino `.ino` file not uploaded in this session — patch doc written as a diff guide rather than full rewrite to avoid introducing bugs in working ESP32 code.

---

## PCB BOM — Both Boards

### IWR6843AOP Board
- **PMIC:** LP87524JRNFRQ1 — available direct from TI.com (myTI account required). LP87524JRNFRQ1: ~2,947 in stock, $2.352/1ku cut tape. LP87524JRNFTQ1 (reel): ~22,520 in stock, $3.248/1ku. No purchase limit currently.
- **Level shifter:** TXS0104EPWR (TSSOP-14) out of stock. Switched to TXS0104EDR (SOIC-14) — pin-for-pin identical, different footprint only. ~3,644 in stock at TI.com, $0.338/1ku. Do not switch back to TSSOP when it returns — SOIC is the better long-term choice for this board.
- **Estimated BOM cost:** ~$36–40/board at 10 units. Assembled: ~$60–75/unit.

### IWRL6432AOP Board
- **Power topology:** BOM-optimized single 1.8V rail. TPS62840 buck converter (60nA IQ, battery-friendly). Internal 1.2V LDO — §7.6.6 decoupling is critical per TI E2E confirmation.
- **Estimated BOM cost:** ~$23–28/board at 10 units. Assembled: ~$45–60/unit.

### FCC Part 15.255 Board-Level Notes
- Supply noise is the #1 board-level FCC risk — 1dB increase in 1.8V ripple = 1dB increase in RX spur level (TI datasheet §7.2)
- LP87524 switches at 4MHz — route switcher traces on inner layers, keep short
- `PMIC_CLKOUT` → `PMIC CLKIN` sync trace: provision it on the PCB even if not enabled by default. Enables PMIC switching sync to radar clock if pre-compliance shows spurs near band edges
- With AOP, antenna design is TI's responsibility — board-level concern is power supply noise coupling into LO path
- UART lines at 921600 baud: ESD array before connector, no UART traces near AOP antenna keepout zone
- Crystal harmonic at 40×N MHz: keep traces <5mm, guarded with ground stitching

### SOP Pin Decision
Hard-wire for functional mode on v1 (SOP0 pull-up 1.8V, SOP1/SOP2 pull-down). Expose pads on PCB for bench flash recovery. No switches needed — you flash every board before it ships. V2: add MOSFET on SOP2 for software-controlled field firmware updates.

### Host Connector (direct Jetson GPIO, no CP2105)
3 signal wires: UART1-TX, UART1-RX, UART2-TX (RX only — sensor never receives on data port). Plus NRESET, 3.3V power, GND. Level shifter required: Jetson GPIO = 3.3V, sensor UART = 1.8V. Recommended: 8-pin JST-SH 1.0mm locking connector.

---

## Firmware / Flash

**What's on every IWR6843AOP:** `out_of_box_6843_aop.bin`, firmware version 03.06.00.00. Flashed via TI UniFlash in SOP2 mode. This is the only thing on the chip itself — the chirp profile (`profile_AOP.cfg`) is NOT firmware, it's sent over CLI UART at runtime every boot.

**Production workflow:** Flash chip before shipping. Buyer never needs UniFlash or SOP switches. Chirp profile delivered via private kit repo access.

---

## Chirp Profile Strategy

### Public repo (`configs/profile_AOP.cfg`)
Example placeholder — uses conservative TI defaults. Gets the sensor streaming valid TLV frames for hardware bring-up verification. Comments explain what the production profile changes in each section without giving away values. Will actually produce CLEAR/CAUTION/STOP transitions.

### Production profile (private)
Stays local on Jetson only, protected by `.gitignore`. Backed up to private Gitea `dntd-mmwave` repo on NucBox (192.168.254.117:3000, ShireFolk account). Kit buyers get access to private repo.

**Security note:** Hiding values in the example file (white text, prompt injection) is not a viable protection strategy — LLMs read raw text regardless of visual formatting. The `.gitignore` + private repo approach is the correct mechanism.

---

## Git / Repo Cleanup

### .gitignore rules added
```
configs/profile_AOP.cfg
*.bak
__pycache__/
*.pyc
*.pyo
.DS_Store
```

### Sequence that was executed
1. `git pull --rebase` — resolved two conflicts (README merge, profile_AOP.cfg delete/modify)
2. `git rm configs/profile_AOP.cfg.bak` and `git rm src/main.py.bak` — removed backup files that slipped into the commit
3. `git rm -r --cached src/__pycache__` — removed pycache that had been tracked since early in the project
4. All cleanup committed and pushed — repo now clean

### Profile gitignore sequence (correct order)
```bash
# 1. Add ignore rule FIRST
echo "configs/profile_AOP.cfg" >> .gitignore

# 2. Stop tracking current file (doesn't delete local copy)
git rm --cached configs/profile_AOP.cfg

# 3. Commit
git add .gitignore
git commit -m "stop tracking production chirp profile"
git push

# 4. Back up real file somewhere safe
cp configs/profile_AOP.cfg ~/profile_AOP_REAL_BACKUP.cfg

# 5. Put example file in place
cp ~/Downloads/profile_AOP_example.cfg configs/profile_AOP.cfg

# 6. Force-add example (overrides ignore rule, one time only)
git add -f configs/profile_AOP.cfg
git commit -m "add example chirp profile for public repo"
git push

# 7. Restore real file
cp ~/profile_AOP_REAL_BACKUP.cfg configs/profile_AOP.cfg
```

**Key behavior:** After `.gitignore` + `git rm --cached`, `git pull` will NEVER overwrite the local file. The ignore rule protects it from both push and pull. Only a fresh `git clone` on a new machine gets the example — which is correct behavior.

---

## Private Gitea Backup

**Gitea:** NucBox K6 at `192.168.254.117:3000`, Docker container, SSH port 222, username ShireFolk  
**Existing repo:** `dntd-mmwave` — use this for private files  
**Push method:** HTTP (SSH auth to Gitea still unresolved from earlier sessions — needs `docker ps | grep gitea` on NucBox to confirm port mapping)

To push real profile to Gitea:
```bash
git clone http://192.168.254.117:3000/ShireFolk/dntd-mmwave.git ~/mmwave-private
cd ~/mmwave-private
cp ~/mmwave/configs/profile_AOP.cfg .
git add profile_AOP.cfg
git commit -m "production chirp profile - validated arm-mounted config"
git push
```

---

## Post-Session Git Cleanup (completed after context first written)

- `src/classifer.py` renamed to `src/classifier.py` via `git mv` — import in `dntd_mmwave_safety_node.py` line 47 updated from `from classifer import` to `from classifier import`. Old misspelled file explicitly removed with `git rm src/classifer.py` after rename left both files tracked.
- `__pycache__/`, `*.bak`, `*.pyc`, `*.pyo`, `.DS_Store` added to `.gitignore`
- `configs/profile_AOP.cfg` protected by `.gitignore` — example placeholder in repo, real file local only
- Real `profile_AOP.cfg` now backed up in three locations: Jetson local, Gitea `dntd-mmwave` private repo (192.168.254.117:3000/ShireFolk/dntd-mmwave), MSI desktop

**URL:** https://github.com/DNTD-Dynamics/RadarGuard-mmwave-cobot-safety-system

### src/ contents
- `arm_config_gui.py` — new today
- `arm_controller_gui.py` — existing (runtime arm control, different from config GUI)
- `arm_controller_node.py` — existing
- `background_model.py` — existing
- `classifer.py` — existing (note: filename typo is in repo, leave as-is)
- `cluster.py` — existing
- `dntd_mmwave_driver_node.py` — existing
- `dntd_mmwave_launch.py` — existing
- `dntd_mmwave_safety_node.py` — existing
- `fake_joint_states.py` — existing
- `live_angle.py` — existing
- `main.py` — updated today
- `presence_hold.py` — new today
- `raw_dump.py` — existing
- `swept_volume.py` — existing
- `tlv_parser.py` — existing
- `uart_reader.py` — existing
- `zone_logic.py` — updated today

### configs/ contents
- `dntd_mmwave_config.yaml` — unchanged
- `dntd_mmwave_driver_config.yaml` — unchanged
- `profile_AOP.cfg` — example placeholder (NOT the production-tuned version)

---

## Immediate Next Steps

1. ~~**Push real profile_AOP.cfg to Gitea**~~ ✅ Done — backed up to Gitea `dntd-mmwave` repo AND MSI desktop
2. ~~**Drop example profile_AOP.cfg onto Jetson**~~ ✅ Done — example version now in `configs/` on Jetson
3. **Arduino CAUTION patch** — follow `CAUTION_SPEED_RAMP_PATCH.md`, three edits to `.ino`
4. **EB300 kinematic chain** — measure link lengths from printed parts, fill in `joint_geometry` in config using `arm_config_gui.py`
5. **ROS2 end-to-end test** — run full stack with `fake_joint_states.py`, validate pipeline runs without crashing

---

## Outputs from This Session

| File | Purpose |
|------|---------|
| `README.md` | Updated public repo README |
| `main.py` | Updated standalone pipeline |
| `zone_logic.py` | Static filter bug fix |
| `presence_hold.py` | New static presence hold module |
| `arm_config_gui.py` | New arm configuration GUI |
| `CAUTION_SPEED_RAMP_PATCH.md` | Arduino patch instructions |
| `profile_AOP.cfg` | Example chirp profile for public repo |
| `DNTD_RadarGuard_PCB_BOM.xlsx` | Dual-tab BOM (IWR6843AOP + IWRL6432AOP) with FCC notes |
| `RadarGuard-mmwave-cobot-safety-system.zip` | Full repo snapshot with all files in correct locations |
