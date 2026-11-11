# BikeSense: phone-based bike route busyness sensing (CEE 54N)

Two phones on a bike (handlebar + frame near the seat) record sound features, acceleration, rotation and GPS. A laptop page turns those recordings into graphs, detected events, per-segment "busyness" scores and CSV files.

| File | Runs on | What it does |
|---|---|---|
| `bike-sensor.html` | each phone | Records sensors and autosaves on the phone. Exports a ZIP of CSVs. |
| `bike-analysis.html` | laptop | Loads ZIPs from one or both phones. Draws graphs and a map, detects stops, brakes and turns, checks against GPS, and exports results. Has a **demo data** button. |

## 1. Put the recorder on the phones (needs HTTPS)

Phones only allow the microphone and motion sensors on **https** pages. Pick one:

- **Netlify Drop (easiest):** go to <https://app.netlify.com/drop> and drag this folder onto it. You get an `https://….netlify.app` link. Open `…/bike-sensor.html` on each phone.
- **GitHub Pages:** push the folder to a repo, then Settings → Pages → deploy from branch. Open `https://<user>.github.io/<repo>/bike-sensor.html`.

On iPhone, use **Safari**. Allow *Motion & Orientation*, *Microphone* and *Location* when asked. If motion is blocked, go to Settings › Safari › Motion & Orientation Access.

The analysis page works from the same link, or locally by double-clicking `bike-analysis.html`. The map needs internet.

## 2. Ride protocol (do the same every time)

1. **Before leaving:** turn off auto-lock on both phones (Settings › Display › Auto-Lock › Never). Sensors stop when the screen locks.
2. Mount both phones **rigidly**. Put foam or a sock over the mic end of each phone to cut wind noise.
3. On each phone, fill in position, phone label (A/B), the side that faces forward, route, rider code and conditions.
4. Press **Start** on both phones. Keep the bike **completely still for 3 s**.
5. Press **📍 Sync bump** on both phones, then knock the frame once firmly. The analysis page uses this spike to line up the two clocks.
6. Ride. At stops (only when safe), tap the marker buttons: Crowd / Music / Traffic / Near-miss.
7. **Stop & save**, then **Download ZIP** or **Share…** (AirDrop or Google Drive) to the laptop.
8. If possible, have a teammate **film the ride** (third phone or action cam) for ground truth.

## 3. What gets recorded

`<session>_motion.csv` (every sensor event, about 60–100 Hz):

| Column | Meaning |
|---|---|
| `acc_x/y/z` | acceleration without gravity, phone frame (m/s²). x = right edge, y = top edge, z = out of the screen |
| `accg_x/y/z` | acceleration including gravity, the raw source for calibration |
| `rot_*_dps` | gyroscope rotation rates (°/s) |
| `a_fwd, a_lat, a_vert` | bike frame: forward (+ = speeding up), sideways (+ = right), vertical |
| `yaw_rate_dps` | turning rate, + = turning right |

`<session>_audio.csv` (one row per second; **no audio is stored**):

| Column | Meaning |
|---|---|
| `laeq_dbfs` / `laeq_cal_db` | A-weighted sound level (relative / with your calibration offset) |
| `l90_a_dbfs` | background level (quietest 10 % of the second) |
| `low_ratio / speech_ratio / high_ratio` | share of energy below 250 Hz (wind, engines) / 300–3400 Hz (voices) / 3.4–8 kHz |
| `speech_mod_2_8hz` | how much the voice band flickers at syllable rate (2–8 Hz). High for talking crowds. |
| `rhythm` | beat regularity at 60–200 BPM. High for music with a beat. |
| `flatness` | 0 = tonal (notes, music), 1 = noise-like (wind, crowd babble) |
| `sound_class` | rough rule-based label: quiet / wind_lowfreq / music_like / voices_like / mixed |

`<session>_gps.csv` (lat, lon, accuracy, speed, heading), `<session>_events.csv` (your markers and app events), and `<session>_metadata.csv/.json` (everything about the setup: phone, mount, calibration, sample rates, mic settings).

## 4. How the analysis answers your four questions

| Your signal | How it's measured | Independent check built in |
|---|---|---|
| Crowd noise vs music | Rule baseline on the sound features. All thresholds are editable. | Your Crowd/Music marker buttons (agreement table), plus video |
| Stopping/starting (you said X and Z) | **Forward** acceleration after rotating the phone axes into the bike frame. Hard brakes and starts are threshold crossings. Stops come from GPS speed. | GPS speed change vs. accelerometer (correlation r); vibration-only stop detector vs. GPS stops |
| Turns (you said Y) | Gyroscope yaw rate (better than accelerometer Y), with total turn angle | GPS heading change; sideways accel vs. speed × yaw (centripetal physics) |
| Busyness per route segment | Weighted mix of noise, % voices, stops/min and brakes/min. The weights are editable and exported. | Compare against video counts of people and cars per segment |

Exports include `analysis_settings.json` with every threshold and check value, so another team can reproduce your plots. The raw phone files are never modified.

## 5. Gaps in the plan and suggestions

1. **Phone X/Y/Z ≠ bike directions.** Which phone axis points forward depends on the mount, and the handlebar phone **turns when you steer**. Your plan maps Z to stopping/starting, but on most mounts Z is mostly up/down (road bumps). The tools rotate everything into forward / sideways / vertical, so use those columns. Use the **frame phone** for braking and turns. The handlebar phone mostly adds steering wobble (itself a possible "busyness" signal: more corrections in crowds).
2. **Wind noise will dominate the mic above ~15 km/h.** Use foam on the mics, watch `low_ratio`, and compare sound levels only at similar speeds (or at stops). Comparing the two phones also helps, because wind differs between mounts while crowd sound doesn't.
3. **Phone mics aren't calibrated sound meters.** Calibrate each phone once against a sound level meter and enter the offset. Otherwise compare only within a phone.
4. **Ground truth is the weakest part right now.** Marker buttons only cover the moments you noticed something. Video lets you count pedestrians, bikes and cars per segment. That gives a real baseline (counts vs. your score) and false-alarm rates (stretches with no crowd that were called "voices").
5. **Hold out data:** decide now which rides you use to tune thresholds and which you keep unseen for the final check (course requirement).
6. **Same route, different times:** for example, ride the same loop at a busy time and a quiet time (Fri 5 pm vs Sun 8 am). That's the cleanest test that the score tracks busyness rather than wind, speed or the rider.
7. **Privacy:** no audio is stored. GPS start and end points can reveal homes, so start recording away from residences, or trim the first and last 100 m before sharing.
8. **Rule-based limits** (be upfront about these): a regular rhythm in speech can look like a beat, loud traffic can mask voices, and indoor music through a window may not show a clear beat. Report the failed cases.

## Files in this folder

- `bike-sensor.html`: phone recorder
- `bike-analysis.html`: analysis and graphs (press *Load demo data* to try it; demo data is synthetic, never use it in results)
- `.claude/launch.json`: local test-server config (not needed by the phones)
