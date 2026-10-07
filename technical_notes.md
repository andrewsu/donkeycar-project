# Donkey Car: Technical Notes & Plan

These notes came out of an initial research conversation (2026-10-07). There are two goals:

1. A learning exercise in RL and "autoresearch"-style agentic optimization, using the simulator.
2. A hardware project with my son: building a physical Donkey Car.

**Confidence labels:**
- **[verified]**: I checked the source code or docs.
- **[reported]**: from docs or READMEs that I haven't reproduced.
- **[inference]**: my reasoning, not yet checked.

---

## 1. The stack

- **Donkeycar** ([autorope/donkeycar](https://github.com/autorope/donkeycar)): the car software (drive loop, data recording to "tubs," training, autopilot). The same code runs on the Pi and in the sim.
- **Donkey sim / sdsandbox** ([tawnkramer/sdsandbox](https://github.com/tawnkramer/sdsandbox)): a Unity simulator. Prebuilt binaries for Linux, Windows and macOS are on the gym-donkeycar releases page.
- **gym-donkeycar** ([tawnkramer/gym-donkeycar](https://github.com/tawnkramer/gym-donkeycar)): a Python wrapper that exposes the sim as an environment for RL libraries. It talks to the sim over TCP on port 9091. **[verified]** It moved from OpenAI Gym to Gymnasium in Jan 2026; the last commit was Mar 2026.
- **Tracks [reported]:** about 10: generated_road, generated_track, warehouse, sparkfun_avc, mini_monaco, warren, thunderhill, circuit_launch, roboracingleague, waveshare.

### Two ways to train

| | Imitation learning (behavior cloning) | Reinforcement learning |
|---|---|---|
| Data | You drive; the car records (image, steering, throttle) | The agent explores by itself |
| Tooling | `donkey train` (standard Donkeycar) | gym-donkeycar + an RL library (SB3, CleanRL, …) |
| Works on the real car? | Yes, the reliable path | Needs sim-to-real transfer; hard |
| Role in this project | The kid's win; the baseline | My research playground |

---

## 2. Facts that matter for the RL design

### The sim runs freely in real time; there's no lockstep stepping **[verified]**
`DonkeyEnv.step()` sends the control command and then just **waits for the next telemetry frame** (`donkey_sim.py` `observe()` waits for `time_received` to change, polling every 1 ms). The sim never pauses for Python. Consequences:
- **Throughput is capped at about the sim's frame rate**, roughly 20 Hz in the docs' screenshot **[reported]**. A 5-minute budget is only about 6,000 steps.
- **If the Python side is slow** (gradient updates in the middle of an episode, slow preprocessing), the car keeps driving on its last command and frames are missed. Raffin's approach does the SAC gradient updates *between* episodes for this reason **[inference: I remember this from his blog/code; re-check]**.
- Results depend on timing, which adds noise on top of seed variance.

### Reward and episode-end are pluggable **[verified]**
`env.set_reward_fn(fn)` and `env.set_episode_over_fn(fn)` exist. The info dict includes `cte` (cross-track error), `pos`, speed and lap info. So reward shaping can live in your own code without forking the package. Other config: `max_cte` (the off-track threshold) and `frame_skip`.

### Headless mode: not supported in the current version **[verified]**
- The earliest gym-donkeycar (v1.0.0, 2019) had `headless=True`, which launched the sim with Unity's `-nographics -batchmode`. That option was removed by Dec 2019. The current launcher passes only `--port`, `--host` and `-logFile`.
- **[inference]** It was probably removed because Unity's `-nographics` skips GPU initialization, so cameras can't render, and camera images are the whole observation.
- **Practical route on Linux:** run the sim under a virtual X display (`xvfb-run`) or on a real desktop session, with the window minimized. **[inference]** Make sure Xvfb uses GPU rendering; software rendering will be slow.

### Parallel sims: unclear, probably fragile **[verified that it's unresolved]**
- Each sim instance accepts a `--port` argument, so multiple instances on different ports is the intended design.
- But the `--multi` flag in gym-donkeycar's `examples/reinforcement_learning/ppo_train.py` is parsed and then **never used**.
- An **open, unanswered issue** ([autorope/donkeycar#1214](https://github.com/autorope/donkeycar/issues/1214), Aug 2025) reports 3 sims on ports 9001–9003 crashing with `Connection reset by peer` (on macOS).
- I found no write-up of anyone successfully running SB3 `SubprocVecEnv` across several Donkey sims.
- **Treat parallelism as something to test, not something to rely on.** Test it early (week 1–2). If it fails, the fallback is running *independent experiments* in parallel (one sim per experiment, which is what autoresearch needs anyway), rather than one experiment across several sims.
- Separately, one sim can host **multiple cars**: the virtual race league runs several clients in one sim. That's for racing, not for speeding up RL.

### GPU: a beefier GPU helps less than you'd think **[inference]**
- **The bottleneck is real-time sim stepping, not compute.** Observations are 120×160, and the policy networks are small (especially on autoencoder latents). A mid-range NVIDIA card has plenty of headroom for a single sim and its training.
- **Where more hardware *does* help:**
  - running several sims and experiments at once: more CPU cores, RAM and VRAM, since each Unity instance renders
  - training the autoencoder or CNN on images
  - native Linux instead of WSL2: no Windows↔WSL networking or GUI-forwarding layer, and Xvfb works
- **Recommendation:** for the autoresearch phase, a dedicated **native Linux box** with a decent NVIDIA GPU (≥12 GB) and many cores beats a top-tier GPU. Measure how much one sim uses (CPU, GPU, VRAM) before buying or moving anything.

---

## 3. Where the performance levers are (beyond picking a library)

The choice of library barely matters. The choice of algorithm matters some: SAC is usually more sample-efficient than PPO for continuous control. The real levers, roughly in order of impact:

1. **Observation representation.** Raw pixels are slow to learn from. A pretrained (V)AE gives a compact latent, and SAC on the latent is the Raffin approach: smooth driving in about 5–20 minutes **[reported]**. Also: cropping out the sky, stacking frames, including past actions in the observation.
2. **Reward design.** The default is roughly "centered and fast," and it gets exploited: weaving, crawling, corner-cutting. Add a penalty for jerky steering and reward progress along the track.
3. **Action space.** Limit the steering rate, smooth actions, bound the throttle, or learn steering only with fixed throttle.
4. **Episode structure.** Termination threshold (`max_cte`), random start points, timeouts.
5. **Hyperparameters.** Learning rate, γ, buffer and batch size, SAC entropy coefficient, network size. Optuna (built into RL Zoo) can search these.
6. **Sim-to-real.** Domain randomization (lighting, textures, camera parameters) and noise. Choose a representation that transfers to real images.

**Seed variance is large in RL.** Any comparison needs several seeds.

---

## 4. Autoresearch design (Karpathy-style)

Karpathy's [autoresearch](https://github.com/karpathy/autoresearch) works like this: an agent edits only `train.py`, each run gets a fixed 5-minute budget, there's one frozen metric (val_bpb), and the agent keeps or discards each change, for about 100 experiments overnight. `prepare.py` (data and evaluation) is frozen, and `program.md` holds the human's instructions.

### Mapping to this project

| autoresearch | Donkey version |
|---|---|
| `prepare.py` (frozen) | `harness.py`: sim launch and restart, env wrappers that define the *evaluation*, the scoring function, held-out tracks |
| `train.py` (agent edits) | Training reward, preprocessing/AE, action wrappers, algorithm and hyperparameters |
| `program.md` | Research directions and constraints |
| val_bpb | e.g. mean lap time over N eval laps on **held-out** track(s), with a large penalty per off-track event; averaged over ≥2–3 seeds |

### Things that must be designed in from the start
- **Frozen evaluation, separate from the training reward.** Otherwise the agent improves the score by gaming the reward.
- **Noise control.** Use multiple seeds, a minimum-improvement margin, and re-run apparent winners. Measure the noise floor first: run the same config with ~5 seeds before starting the loop.
- **Held-out tracks**, so it doesn't overfit to one track.
- **Robust harness.** Timeouts, automatic sim restarts, detection of a hung TCP connection. One sim crash at 2 a.m. shouldn't end the night.
- **Time budget.** 5 minutes is probably too short. Start at 10–20 minutes per experiment with the AE+SAC setup, so roughly 3–6 experiments per hour per sim.

---

## 5. Rough arc for the first ~2 months

These are milestones, not a tutorial. The kid track and the me track run in parallel and meet in weeks 6–8.

### Weeks 1–2: Get everything running
- **Me:** Install the sim and Donkeycar on the target machine (decide WSL2 vs. native Linux now). Drive the sim manually. Run the stock imitation-learning loop (record, `donkey train`, autopilot) to confirm the stack works end to end. Run a raw gym-donkeycar random-action loop and measure the actual step rate (Hz).
- **Me:** Test parallel sims: 2 instances on different ports. Test headless with Xvfb. Record the results here.
- **Kid:** Drive the sim (gamepad), customize the car, train his own imitation model from his own driving and watch it drive. *This is the hook.*
- **Together:** Decide on the physical car kit and order parts. (Checklist: chassis, Pi 4/5 or Jetson, camera, PCA9685 or equivalent, buck converter, battery, and if LiPo, a LiPo-safe charging bag.)

### Weeks 3–4: Baseline RL in the sim, build the car
- **Me:** Run SB3 SAC on gym-donkeycar with a hand-written reward; expect it to fail in some instructive way. Then reproduce the Raffin approach (AE + SAC + action smoothing) on one track, as the hand-tuned baseline. Log lap time, off-track rate and steering jerk.
- **Kid:** Build the car: mechanical assembly, wiring, power, then calibrate steering and throttle (`donkey calibrate`).
- **Together:** "Reward design game" sessions in the sim: he proposes a reward rule and you both watch what the car learns (and how it cheats).

### Weeks 5–6: Evaluation harness, first real-car autopilot
- **Me:** Build the frozen harness (scoring, held-out track, seeds, timeouts and restarts). Measure the noise floor. **Only then** write `program.md` and run the first short autoresearch loop (a few hours), and read every experiment it ran.
- **Kid:** Tape a track at home, record driving data on the real car, train, and get **real autopilot laps**. This milestone deliberately doesn't depend on the RL work.

### Weeks 7–8: Overnight loops and sim-to-real
- **Me:** Run overnight autoresearch loops. Check whether the "improvements" survive re-runs on fresh seeds. Write up which levers actually mattered.
- **Together:** Sim-to-real experiment: make a sim track look like the home track (or randomize the domain), then try the best sim policy on the real car. Expect partial failure; *why* it fails is the interesting part.
- **Optional:** Find a DIY Robocars meetup or virtual race as a goal.

---

## 6. Open questions to resolve early
- [ ] Machine: stay on WSL2 or move to a native Linux box? (Decide after measuring one sim's resource use.)
- [ ] Does running more than one sim instance actually work? (See issue #1214.)
- [ ] Does Xvfb rendering give usable FPS?
- [ ] What's the real step rate, and how much does Python-side latency drop frames?
- [ ] Which Donkeycar version and model format (the docs' `.h5` examples may be outdated; recent PRs moved the Pi to TFLite)?
- [ ] Which car platform for the kid build?

## Sources
- Donkey simulator docs: https://docs.donkeycar.com/guide/deep_learning/simulator/
- gym-donkeycar: https://github.com/tawnkramer/gym-donkeycar (code read directly: `envs/donkey_proc.py`, `envs/donkey_env.py`, `envs/donkey_sim.py`, `examples/reinforcement_learning/ppo_train.py`, plus git history of `donkey_proc.py`)
- Multi-sim issue: https://github.com/autorope/donkeycar/issues/1214
- Raffin, learning-to-drive-in-5-minutes: https://github.com/araffin/learning-to-drive-in-5-minutes
- Karpathy autoresearch overview: https://www.theneuron.ai/explainer-articles/andrej-karpathys-autoresearch-tiny-repo-big-implications/
