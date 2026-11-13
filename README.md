# rov-pilot-kit

The minimum needed to run path-following controllers and a camera-driven LLM pilot on a
BlueROV2 from the topside laptop. Pure Python on top of ROS 2 and pymavlink.

**Status:** tested in simulation and against a fake autopilot. Do the bench check before
any controller runs. One hand near SPACE at all times.

Full procedure, pool layout and stop table: [INSTRUCTIONS.md](INSTRUCTIONS.md).

## Get it

Inside the BlueROV-Tools folder, so the ROS 2 container sees it and the GUI still works:
```bash
cd <path to>/BlueROV-Tools
git clone https://github.com/sumanthk123/rov-pilot-kit.git pilot_kit
echo pilot_kit/ >> .git/info/exclude
```

## Before anything moves

1. GUI: **Launch All**, then stop **Dead Reckoning**. Do **not** click **Connect** in the
   Teleop panel. GUI teleop and this kit cannot share the vehicle link.
2. Vehicle in **MANUAL** mode.
3. Three ROS 2 terminals, each: `cd .../BlueROV-Tools/pilot_kit`

**Terminal A, dead reckoning**
```bash
ros2 run bluerov_dr dead_reckon --ros-args -p estimator_mode:=legacy_integrator
```

**Terminal B, stick converter. Holds the emergency stop. Keep it in focus.**
```bash
python3 cmdvel_to_manual.py --depth-mode passthrough --fence=-0.4,2.9,-0.4,0.8   # pipe line course
python3 cmdvel_to_manual.py --depth-mode passthrough                             # rectangle course
```
Keys: `a` arm · **SPACE** disarm and latch stop · `c` clear stop · `i k` jog fwd/back · `j l` jog yaw left/right

**Bench check, ROV held at the wall:** `a`, tap `i` (must push forward), tap `l` (nose must
swing clockwise seen from above), SPACE (must disarm), `c`. Wrong direction: stop here.

## Run a controller

Place the ROV on its start mark, nose along the course. Arm with `a` in Terminal B. Then:

**Terminal C**
```bash
./run_one.sh <law> <course> <rep>        # e.g.  ./run_one.sh los line 1
```
| law | | course | |
|---|---|---|---|
| `p` | proportional heading | `line` | start 0.3 m RIGHT of the pipe, parallel to it. 2.4 m pass |
| `pursuit` | pure pursuit | `rect` | 2.0 m ahead, 0.6 m left, back. Two laps |
| `los` | line-of-sight | | |
| `ilos` | integral line-of-sight | | |
| `fuzzy_lab` | the lab's fuzzy tracker | | |

It resets dead reckoning, records a bag, runs, and stops by itself. Output: `runs/<name>/`.
Shorter pipe: `LINE_LENGTH=1.8 ./run_one.sh los line 1`. Different depth: `DEPTH=-0.4 ...`.

Suggested order, whole list once per repeat:
```
line:  los  p  fuzzy_lab  pursuit  ilos     (rep 1, 2, 3)
rect:  p  pursuit  los  ilos  fuzzy_lab     (rep 1, 2)
```
Afterwards: `python3 score_runs.py runs/`

## LLM pilot (optional, needs internet)

```bash
python3 -m pip install openai
echo 'OPENAI_KEY=...' > .env && chmod 600 .env
python3 llm_pilot.py --check                                   # one test call, no motion
```
Goal that works (stop on something visible, never on a distance):
```bash
GOAL="Follow the dark pipe lying on the pool floor to its far end, staying over it or right beside it. Stop when the far end of the pipe reaches the bottom edge of the frame or passes out of view below it."
```
Three stages, in order. Stop at the first one that looks wrong.
```bash
python3 llm_pilot.py --frames <folder of images> --goal "$GOAL" --out llm_runs/offline1   # 1. recorded frames, no ROV
python3 llm_pilot.py --dry-run --goal "$GOAL" --out llm_runs/dry1                         # 2. live camera, ROV does not move
python3 llm_pilot.py --depth-target -0.3 --goal "$GOAL" --out llm_runs/pipe1              # 3. live
```
For stages 2 and 3 run Terminal B with `--max-run 300 --fence=-0.6,3.0,-0.8,0.8`.

## Files

| File | Purpose |
|---|---|
| `cmdvel_to_manual.py` | velocity command to stick command. Arm, stop, jog, limits, geofence |
| `course_runner.py` | runs one steering law on `/deadreckon/odom`, publishes `/cmd_vel` |
| `tracker.py` | the five steering laws and the course logic. No ROS |
| `run_one.sh` | one recorded run: reset, bag, run, stop |
| `llm_pilot.py` | camera frame to model to one small action, step by step |
| `score_runs.py` | time, path length and tracking error per run |
| `test_tracker.py` | offline self-check: `python3 test_tracker.py` |
