# Runbook — FUXA HMI: iPTran Leak Detection System

End-to-end procedure to run this FUXA deployment: start the application,
configure devices and tags, place the versioned iPTran schematic widgets,
bind them, test in the Lab, run as HMI, and stop it cleanly.

| | |
|---|---|
| Application | FUXA web SCADA/HMI (this repository) |
| Deployment | Docker Compose (primary), `npm start` (alternative) |
| Web UI / Editor / Lab | http://localhost:1881 / `/editor` / `/lab` |
| Default login | user `admin`, password `123456` (security currently off) |
| Current views | `iPTran_Overview` (mock) -> `iPTran_Schematic` (widget v2) -> `iPTran_Live` (widget v3) |
| Audience | Operator / Commissioning engineer |

---

## 1. Prerequisites

- Docker Engine + Docker Compose (primary path), or Node.js 18 + npm
- Chrome or Edge browser
- Repo checked out at `~/fuxahmi/FUXA`

## 2. Start the application

**Docker (primary).** `compose.yml` bind-mounts all state to the host, so data
survives container removal:

| Host directory | Container path | Contents |
|---|---|---|
| `FUXA/appdata` | `.../server/_appdata` | settings, project/views/devices, users |
| `FUXA/db` | `.../server/_db` | DAQ history (SQLite) |
| `FUXA/logs` | `.../server/_logs` | server and API logs |
| `FUXA/images` | `.../server/_images` | image resources |
| `FUXA/widgets` | `.../server/_widgets` | widget library (dynamicSVG group) |

```bash
cd ~/fuxahmi/FUXA
docker compose up -d
docker ps | grep fuxa          # expect: Up ... 0.0.0.0:1881->1881/tcp
```

Open http://localhost:1881. Startup takes a few seconds; check
`docker logs fuxa-fuxa-1` for `FUXA started!` and `WebServer is running`.

**npm (alternative).** Widget source of truth is `server/_widgets/`, served
directly (no `widgets/` staging needed):

```bash
cd FUXA/server
npm install        # first time only
npm start
```

## 3. Verify the widget library

Widgets must live in **subfolders** of `_widgets/` — the editor lists only
subfolders (`GET /api/resources/widgets`). Current Docker library:

```
FUXA/widgets/dynamicSVG/
├── README.md                  # revision history
├── iptran_schematic_v1.svg    # 4 pressure vars (static mock)
├── iptran_schematic_v2.svg    # 15 vars: flows, level, pumps, comm, alarms
├── iptran_schematic_v3.svg    # v2 + animated flow, alarm banner, trends, clock
└── iptran_schematic.svg       # alias of the latest revision
```

Quick checks (security off, so no auth needed):

```bash
curl http://localhost:1881/api/resources/widgets   # must list all 4 SVGs
curl -o /dev/null -w "%{http_code}\n" \
  http://localhost:1881/_widgets/dynamicSVG/iptran_schematic_v3.svg   # expect 200
```

## 4. Configure the device and tags (Editor -> Device panel)

For simulation/testing this project uses the built-in `FUXA Server` device
(type `FuxaServer`). For field deployment, create ModbusTCP/OPC UA/MQTT
devices for PLC-1/2/3 and use the same tag names.

Tags used by the bindings (types: number / boolean):

| Tag | Type | Process point |
|---|---|---|
| `StationA_PT01` | number | PT-01 intake pressure, bar |
| `StationB_PT02` | number | PT-02 inlet pressure, bar |
| `StationB_PT03` | number | PT-03 discharge pressure, bar |
| `StationC_PT04` | number | PT-04 reservoir pressure, bar |
| `StationA_FM_Exist` | number | existing Vedanta SCADA flow, m3/h |
| `StationB_FM01` | number | FM-01 flow, m3/h |
| `StationC_FM_Exist` | number | reservoir outlet flow, m3/h |
| `StationB_LT01` | number | LT-01 tank level, % |
| `PUMP_A_RUN` / `PUMP_B_RUN` | boolean | pump running feedback |
| `COMM_ONLINE` | boolean | edge link online (per-station tags: `COMM_A/B/C` recommended for real links) |
| `Cloud_Server_Status` | boolean | cloud connectivity |

Enable **DAQ** on tags that must be historized for charts/alarms.
Confirm the device is green/online before binding.

## 5. Create views and place widgets

Convention: **one view per widget revision**, older views stay untouched as
the progress trail.

1. Editor (gear icon) -> Views panel (left) -> create the view, e.g. `iPTran_Live`.
2. Open the **Lib Widgets** panel, expand **dynamicSVG**.
3. Click the widget **file entry** (e.g. `iptran_schematic_v3`) — this arms
   placement mode.
4. Click (or click-drag) on the canvas to drop it. Resize with the handles
   (v3 is taller than v2 — the status strip adds ~70 px).
5. Repeat for additional views if staging multiple revisions.

Placed widgets freeze their SVG content at placement time; a new revision
never changes an existing view.

## 6. Bind widget variables to tags

1. Select the placed widget on the canvas (Transform panel appears on the right).
2. Open its **Property** section (right sidebar) — one row per exported variable.
3. For each row, open the tag picker (**Tag Selection** dialog: choose device
   `FUXA Server`, then the tag), and bind:

| Variable | Tag | | Variable | Tag |
|---|---|---|---|---|
| `_ps_stationA_pt` | `StationA_PT01` | | `_pn_stationB_level` | `StationB_LT01` |
| `_pn_stationA_flow` | `StationA_FM_Exist` | | `_ps_stationC_pt` | `StationC_PT04` |
| `_ps_stationB_pt2` | `StationB_PT02` | | `_pn_stationC_flow` | `StationC_FM_Exist` |
| `_pn_stationB_flow` | `StationB_FM01` | | `_pb_pumpA_run` | `PUMP_A_RUN` |
| `_ps_stationB_pt3` | `StationB_PT03` | | `_pb_pumpB_run` | `PUMP_B_RUN` |
| `_pb_commA/B/C` | `COMM_ONLINE` (all three) | | `_pb_pumpA_cmd`, `_pb_pumpB_cmd` | leave UNBOUND |

   Notes:
   - Bindings are per placed element — each view's widget needs its own.
   - The `_cmd` variables write outward via `postValue()`; only bind them to
     **writable** command tags on a real PLC, never to the feedback (`_run`) tags.
   - Alarm ranges are built into the widget: pressure 0.5–6.0 bar,
     flow 0–2000 m3/h, level 20–90%. Values outside range turn red (and in v3
     raise the alarm banner).
4. Click **OK** to commit the dialog — until then edits are not persisted.
5. Click the **save icon -> Save Project**.

## 7. Save the project

- FUXA auto-saves: devices/tags/bindings save on dialog confirmation, views
  save when switching views or leaving the editor.
- **Save Project** (save-icon menu) forces a full save.
- **Save Project As...** exports the whole project as a JSON file — use this
  for backups and moving projects between instances.
- **Open Project** restores from such a JSON export.

## 8. Test in the Lab, then run as HMI

**Lab** (http://localhost:1881/lab):

1. Select the target view in the **editor** first — the Lab renders the
   editor's last-selected view (remembered per browser).
2. In the Lab, click the red **Test** button to open the value tester.
3. Drive values and verify v3 behavior: flow > 0 animates the pipeline dashes
   (speed scales with rate), out-of-range values turn red and raise the
   blinking alarm banner and ALARMS chip, boolean tags flip pump colors and
   comm dots (comm false dims the station and shows LINK DOWN), sparklines
   track the last 40 pressure samples, clock and LIVE/STALE indicator run.

**Home / production view** (http://localhost:1881): pick the view from the
top-bar dropdown; live values update per device polling interval (default
3000 ms). Expected initial state with the FUXA Server simulator: flows
1240.0, level 85.2%, pumps green, comm dots green, pressures 0.00 shown red
(zero is below the 0.5 bar alarm limit — expected until real PLCs feed values).

## 9. Stop, restart, and reset

```bash
cd ~/fuxahmi/FUXA
docker compose stop      # stop, keep container (quick restart)
docker compose down      # stop and remove container (data survives in bind mounts)
docker compose up -d     # start again
```

npm variant: stop with `Ctrl+C` (graceful shutdown flushes SQLite DAQ files).

Full reset (destroys project/views/settings — export a project JSON first):
stop the application, then delete `FUXA/appdata/`, `FUXA/db/`, `FUXA/logs/`.

## 10. Troubleshooting

| Symptom | Cause / fix |
|---|---|
| Widget not listed in the widget library | SVG must be inside a subfolder of `_widgets/` (e.g. `dynamicSVG/`); for Docker it must be in `FUXA/widgets/<group>/`. Refresh the editor. |
| Docker container `Up` but 1881 unreachable | Port publish failed (often created while another container held 1881) — `docker compose up -d --force-recreate`. |
| Old/different widget renders on a new placement | Browser cached a previous copy — Ctrl+F5; if needed clear site data (F12 -> Application -> Clear site data). |
| Lab shows the wrong view | The Lab renders the editor's last-selected view. Select the view in the editor, then open `/lab` (refresh the tab if it was already open). |
| Placed widget does not upgrade to a new revision | Placed widgets freeze content at placement. Place the new revision file on a new view (preferred), or delete + re-place + re-bind. |
| `401 Unauthorized` on widget panel | Widgets API requires admin — sign in as admin (when security is enabled). |
| Values stuck at defaults | Device offline, tag not bound, or binding row not committed — click **OK** in the dialog, then Save Project. |
| `EADDRINUSE` on startup | Port 1881 occupied — `docker compose down` any other FUXA container first, or use another port. |
| Pumps change color when only clicked | `_pb_*_cmd` bound to the `_run` feedback tag — unbind the cmd rows; commands must go to writable tags only. |
| Server log `data-overload!` at high tag rates | Known DAQ write-path limitation; don't raise polling above ~1 Hz with DAQ until the queued-batch design is implemented. |

## 11. Files reference

```
FUXA/
├── compose.yml                            # Docker deployment (bind mounts incl. widgets)
├── runbook.md                             # this file
├── appdata/  db/  logs/  images/          # container state (bind mounts)
├── widgets/dynamicSVG/                    # widget library served to the container
│   ├── README.md                          # revision history + release procedure
│   └── iptran_schematic*.svg              # v1 / v2 / v3 + latest alias
├── requirements.txt                       # Python deps (MkDocs docs site only)
├── docs/HowTo-Widgets.md                  # FUXA widget/SVG scripting standard
└── server/
    ├── main.js                            # entry point (npm variant)
    ├── settings.default.js                # defaults -> copied into _appdata on first run
    ├── _widgets/dynamicSVG/               # widget source of truth (npm variant)
    └── runtime/
        ├── devices/device.js              # device state machine, polling intervals
        └── storage/daqstorage.js          # DAQ storage (SQLite/Influx/QuestDB)
```

Project data lives in `appdata/project.fuxap.db` (views, devices, bindings);
the widget revision placed on each view can be verified by checking whether
its saved element content contains the v3 root marker (`iptran-widget-root`).
