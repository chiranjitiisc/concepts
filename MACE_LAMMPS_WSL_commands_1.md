# MACE + LAMMPS on WSL — Command Cheat Sheet

**Machine:** TimeMachine (DESKTOP-4DUSOA5) · WSL2 Ubuntu 24.04 · RTX 5080
**Your WSL home:** `/home/time_machine` (shortcut: `~`)
**Conda env:** `mace` · **LAMMPS binary:** `~/lammps/build/lmp`
**Models:** `~/mace_models/`

Everything below is typed in the **WSL terminal** (Ubuntu). Paste with `Ctrl+Shift+V`
in Windows Terminal, or right-click in MobaXterm.

---

## 1. The 4 commands you need most

This is the whole workflow. Replace `NEWFOLDER` with your actual folder name.

```bash
# 1. Copy the folder from your Windows Desktop into WSL
cp -r "/mnt/c/Users/Time Machine/Desktop/NEWFOLDER" ~/

# 2. Go into the copy
cd ~/NEWFOLDER

# 3. Activate the environment (skip if your prompt already shows "(mace)")
conda activate mace

# 4. Run
~/lammps/build/lmp -k on g 1 -sf kk -pk kokkos newton on neigh half -in in.lmp
```

**Why copy instead of running from the Desktop?**
`/mnt/c/...` is the Windows disk seen through a translation layer. It is much slower for
writing dump/log/restart files. `~` is the native Linux disk. Always run from `~`.

---

## 2. Understanding the run command

```bash
~/lammps/build/lmp  -k on g 1  -sf kk  -pk kokkos newton on neigh half  -in in.lmp
```

| Piece | Meaning |
|---|---|
| `~/lammps/build/lmp` | the LAMMPS executable (not on PATH — see §9) |
| `-k on g 1` | turn Kokkos **on**, use **1 GPU** |
| `-sf kk` | use the Kokkos (GPU) version of each style |
| `newton on` | **required by MACE** |
| `neigh half` | **required because you used `newton on`** |
| `-in in.lmp` | your input script |

> **Remember this pair:** MACE needs `newton on`, and `newton on` needs `neigh half`.
> Kokkos defaults to `neigh full`, which only allows `newton off`. That mismatch is the
> error you hit first.

---

## 3. Copying files between Windows and WSL

Your Windows drives are always mounted under `/mnt`:

| Windows | In WSL |
|---|---|
| Desktop | `/mnt/c/Users/Time Machine/Desktop/` |
| Downloads | `/mnt/c/Users/Time Machine/Downloads/` |
| Documents | `/mnt/c/Users/Time Machine/Documents/` |
| `D:\` drive | `/mnt/d/` |

**Quote paths containing spaces** — `Time Machine` always needs quotes.

```bash
# Windows  ->  WSL   (copy a folder in)
cp -r "/mnt/c/Users/Time Machine/Desktop/myfolder" ~/

# WSL  ->  Windows   (copy results back out)
cp -r ~/myfolder "/mnt/c/Users/Time Machine/Desktop/"

# Copy a single file only
cp "/mnt/c/Users/Time Machine/Desktop/myfolder/in.lmp" ~/myfolder/
```

**With a mouse instead:** open Windows Explorer and paste this into the address bar —

```
\\wsl.localhost\Ubuntu-24.04\home\time_machine
```

That is your WSL home as a normal folder window. Drag and drop freely.
Or, from any WSL directory, run `explorer.exe .` to open it in Explorer.

---

## 4. Fixing the `/home/chiranjit/` path problem

Scripts written on the cluster point at `/home/chiranjit/`, which does not exist here.
Symptom:

```
FileNotFoundError: ... '/home/chiranjit/mace_models/mace-omat-0-small.model-mliap_lammps.pt'
ERROR: Running mliappy unified module failure.
```

**Best fix — do this once, then never think about it again:**

```bash
sudo ln -s /home/time_machine /home/chiranjit
```

This makes `/home/chiranjit/...` resolve to your real home, so cluster scripts run
unedited on both machines.

**Or fix one file at a time:**

```bash
sed -i 's|/home/chiranjit/|/home/time_machine/|g' in.lmp
```

Prints nothing when it works. Check it took effect:

```bash
grep -n "mace_models" in.lmp
```

`sed -i` edits the file permanently — you only do it once per file, not every run.
Copy the fixed file back to the Desktop so future copies are already clean.

---

## 5. Long runs — don't lose them when you close the terminal

A foreground run dies with the terminal. For real jobs:

```bash
nohup ~/lammps/build/lmp -k on g 1 -sf kk -pk kokkos newton on neigh half -in in.lmp > run.out 2>&1 &
```

> ### The `&` is the whole point
> **Without the trailing `&` the job is NOT in the background.** Your terminal stays
> blocked, no prompt comes back, and you only see:
> ```
> nohup: ignoring input and appending output to 'nohup.out'
> ```
> That message means it is running *in front of you*, not detached. Add the `&`.
>
> Also keep `> run.out 2>&1`, otherwise output goes to `nohup.out` instead.

A correct launch prints a job number and PID and returns your prompt immediately:

```
[1] 12345
```

**Write that PID down** — it is the easiest way to kill the job later.

Watch progress:

```bash
tail -f run.out        # Ctrl+C stops watching, NOT the run
tail -f log.lammps     # LAMMPS thermo output
```

Check it is still alive:

```bash
ps aux | grep lmp      # is the process there?
nvidia-smi             # is the GPU busy?
```

**WSL warnings for long jobs:**

- Windows going to **sleep suspends WSL** — the run freezes and resumes on wake.
  Set the power plan to *never sleep* before starting a long job.
- **`wsl --shutdown` kills everything immediately.** Never run it while a job is going.
- `run.out` catches Python/torch tracebacks (where MACE errors appear);
  `log.lammps` has the thermo data. Keep both.

---

## 6. Stopping / cancelling a run

**Which method you need depends on how you launched it.**

### If it is in the foreground (no `&`, no prompt showing)

```
Press Ctrl+C
```

That is all. Works whether or not you typed `nohup`.

### If it is genuinely in the background (you used `&`)

`Ctrl+C` will **not** work — the job is no longer attached to your terminal. Use any one of:

```bash
pkill -f "lmp .*in.lmp"        # simplest — kills by matching the command line
```

```bash
kill 12345                      # by the PID printed at launch
```

```bash
jobs                            # list background jobs in THIS terminal
kill %1                         # kill job number 1
```

### If you closed and reopened the terminal

`jobs` will be empty — the job list belongs to the old shell. Find the process again:

```bash
ps aux | grep lmp
```

The number in the **second column** is the PID. Then:

```bash
kill <that PID>
```

### If it refuses to die

```bash
kill -9 <PID>
```

Force-kill. Try plain `kill` first so LAMMPS can close its dump files cleanly —
`kill -9` can leave a truncated trajectory.

### Confirm it is gone

```bash
ps aux | grep lmp      # should show only the grep line itself
nvidia-smi             # GPU memory should drop back
```

---

## 7. Where the output files are, and getting them back to Windows

**LAMMPS writes into whatever folder you were standing in when you pressed Enter.**
If you ran from `~/test_runs`, everything is in `~/test_runs`.

List them, newest first:

```bash
ls -lat ~/test_runs
```

| File | What it is |
|---|---|
| `log.lammps` | thermo output, timings, all the physics — the main one |
| `nohup.out` or `run.out` | screen output + Python/torch tracebacks (MACE errors appear here) |
| `dump.*` / `*.lammpstrj` | trajectories, if `in.lmp` has a `dump` command |
| `restart.*` / `*.restart` | restart files, if `in.lmp` writes them |

To see exactly what your script was told to write:

```bash
grep -nE "dump|write_data|write_restart|log " ~/test_runs/in.lmp
```

> If a line writes to a subfolder like `dumps/traj.lammpstrj`, **that folder must already
> exist** — LAMMPS will not create it, and the run dies partway through.
> Make it first with `mkdir -p dumps`.

### Copying results back to the Desktop

```bash
# Whole folder, under a new name (safe — nothing gets overwritten)
cp -r ~/test_runs "/mnt/c/Users/Time Machine/Desktop/test_runs_results"
```

```bash
# Only the outputs, into the folder that is already on the Desktop
cp ~/test_runs/log.lammps ~/test_runs/run.out ~/test_runs/dump.* "/mnt/c/Users/Time Machine/Desktop/test_runs/"
```

```bash
# Overwrite the existing Desktop folder with the current WSL contents
cp -rf ~/test_runs/. "/mnt/c/Users/Time Machine/Desktop/test_runs/"
```

The `.` after `test_runs/` means *"the contents of"* — files land **inside** the existing
folder instead of creating `test_runs/test_runs`.

**Or drag them with a mouse:**

```bash
explorer.exe ~/test_runs
```

Opens the folder in Windows Explorer — handy when you are about to load a trajectory
into OVITO on the Windows side.

Confirm the copy arrived:

```bash
ls -la "/mnt/c/Users/Time Machine/Desktop/test_runs_results"
```

---

## 8. Useful checks

```bash
# Which env am I in? Where is python?
echo $CONDA_DEFAULT_ENV && which python

# Is the GPU visible to torch?
python -c "import torch; print(torch.cuda.is_available(), torch.cuda.get_device_name(0))"

# GPU usage / memory, live
watch -n 1 nvidia-smi

# What models do I have?
ls -lh ~/mace_models/

# Does LAMMPS have the styles I need?
~/lammps/build/lmp -h | tr ' ' '\n' | grep -xE "mliap|mliap/kk|d3"

# Where am I? What is here?
pwd && ls -la
```

---

## 9. One-time conveniences (highly recommended)

**Put `lmp` on your PATH** so you can type `lmp` instead of `~/lammps/build/lmp`:

```bash
echo 'export PATH="$HOME/lammps/build:$PATH"' >> ~/.bashrc && source ~/.bashrc
```

**Make a short alias** for the long GPU command line:

```bash
echo "alias lmpgpu='$HOME/lammps/build/lmp -k on g 1 -sf kk -pk kokkos newton on neigh half'" >> ~/.bashrc && source ~/.bashrc
```

After that, a run is simply:

```bash
lmpgpu -in in.lmp
```

and a background run:

```bash
nohup lmpgpu -in in.lmp > run.out 2>&1 &
```

**Auto-activate the mace env** in every new terminal:

```bash
echo 'conda activate mace' >> ~/.bashrc
```

---

## 10. Converting a MACE model for LAMMPS

Use the **mliap** format. It applies cuEquivariance automatically — there is no flag,
and no need to ask for it.

```bash
conda activate mace
mace_create_lammps_model ~/mace_models/YOUR_MODEL.model --format=mliap --dtype=float64
```

Produces `YOUR_MODEL.model-mliap_lammps.pt` next to the original. Point your
`pair_style` at that file:

```
pair_style hybrid/overlay mliap unified /home/time_machine/mace_models/YOUR_MODEL.model-mliap_lammps.pt 0 d3 3214 1428 damp_bj pbe
```

> Do **not** use `--format=libtorch` if you want speed — that path goes through
> TorchScript and gets no cuEquivariance acceleration.

Models stay in `~/mace_models/` — one central copy, referenced by absolute path.
Do not duplicate them into each run folder.

---

## 11. Errors you have already met, and their fixes

| Error / symptom | Cause | Fix |
|---|---|---|
| `Must use 'newton off' with KOKKOS package option 'neigh full'` | Kokkos defaults to full neighbour lists | add `neigh half` → `-pk kokkos newton on neigh half` |
| `FileNotFoundError: /home/chiranjit/...` | path from another machine | §4 — symlink once, or `sed -i` the file |
| `No such file or directory: mnt/c/...` | missing leading `/` | use `/mnt/c/...` — the slash matters |
| `nohup: ignoring input and appending output to 'nohup.out'` **and no prompt returns** | forgot the trailing `&` — job is in the foreground | `Ctrl+C`, then relaunch with `> run.out 2>&1 &` |
| `ModuleNotFoundError` / model won't load | wrong conda env | `conda activate mace` |
| Run dies partway when writing a dump | dump folder does not exist | `mkdir -p dumps` before running |
| `Failed to get GPU information from pynvml ... RTX A6000` | WSL's NVML shim; MACE falls back to defaults | **harmless** — affects only batch-size hints, never physics |
| `torch.jit.script is deprecated`, `weights_only` warnings | e3nn 0.4.4 on torch 2.11 | **harmless**, ignore |

---

## 12. Quick reference — Linux basics

```bash
cd ~                  # go to home (/home/time_machine)
cd ~/test_runs        # go to a folder
cd ..                 # up one level
pwd                   # where am I?
ls -la                # list everything, with sizes and dates
ls -lat               # same, newest first
mkdir -p ~/runs/test  # make a folder (and parents)
cp file1 file2        # copy a file
cp -r dir1 dir2       # copy a folder (-r = recursive)
mv old new            # rename or move
rm file               # delete a file      (no undo!)
rm -r folder          # delete a folder    (no undo!)
cat in.lmp            # print a whole file
head -20 in.lmp       # first 20 lines
tail -20 log.lammps   # last 20 lines
tail -f log.lammps    # follow a file as it grows (Ctrl+C to stop watching)
grep -n "pair_style" in.lmp   # find a line, with line number
nano in.lmp           # edit a file (Ctrl+O save, Ctrl+X exit)
du -sh ~/test_runs    # how big is this folder?
df -h                 # disk space
ps aux | grep lmp     # find running LAMMPS processes
kill <PID>            # stop a process
```

---

*Cheat sheet for MACE + LAMMPS + cuEquivariance (ML-IAP unified) on WSL2 · 8 September 2026*
