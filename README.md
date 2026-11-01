# Arionix Attendance System

A desktop attendance system for Windows that recognises employees by their face. Point a webcam at the door, and every recognised person is checked in automatically. No cloud, no server, no account: everything runs and is stored on the local machine.

**Version:** 3.0.0  **Author:** Iliya Saiedi

## Features

- **Live face recognition**: attendance is recorded the moment a registered face appears in front of the camera.
- **Easy registration**: register an employee with a webcam capture or from a photo file.
- **Duplicate protection**: a face that is already registered cannot be registered a second time, and the same person is not recorded again within the cooldown period (default 60 seconds, also enforced across restarts).
- **Dashboard**: present today, total employees, last check-in, latest attendance table and an activity log.
- **Reports**: filter by date range (Today / Last 7 days / Last 30 days), search by name or ID, and see a **daily summary** with first scan, last scan, number of scans and time span per person per day.
- **CSV export**: UTF-8 with BOM so Excel shows Persian names correctly. Cells that look like spreadsheet formulas are neutralised.
- **Shamsi (Jalali) dates**: switch all displayed dates between Gregorian and Shamsi.
- **Modern interface**: dark and light themes, toast notifications, optional check-in sound.
- **Fully local**: SQLite database, no network access needed after installation.

## Requirements

- Windows 10 or 11 (64-bit)
- Python **3.10** (other versions may lack a prebuilt `dlib` wheel)
- A webcam (employees can still be registered from photos without one)

## Installation

The commands below use [uv](https://docs.astral.sh/uv/). Run them in PowerShell from the project folder.

```powershell
uv venv --python 3.10
.\.venv\Scripts\Activate.ps1
uv pip install -r requirements.txt
uv pip install face_recognition --no-deps
```

`face_recognition` is installed separately with `--no-deps` on purpose: its normal dependency `dlib` has no prebuilt Windows wheel and tries to compile from source (which needs Visual Studio C++ Build Tools). `requirements.txt` contains `dlib-bin` instead, which is the same library, prebuilt.

`requirements.txt`:

```
numpy
opencv-python<5
Pillow
setuptools<81
customtkinter
dlib-bin
face-recognition-models
Click
```

On a slow or unstable connection, raise the download timeout first: `$env:UV_HTTP_TIMEOUT = "300"`.

Check the installation:

```powershell
python -c "import dlib, face_recognition, cv2, customtkinter; print('OK')"
```

## Running

```powershell
python run.py
```

## Usage

1. **Register Employee**: enter the full name and a unique employee ID, then choose **Use camera** or **Choose photo…**. Exactly one clear, front-facing face must be visible.
2. **Start Attendance**: opens the live camera window. Known faces get a green box with the employee ID, unknown faces a red box.
3. **Employees**: search and delete employees. Deleting an employee also deletes all of their attendance records.
4. **Attendance Report**: choose a date range (`YYYY-MM-DD`), review the records and the daily summary, and export to CSV.
5. The sidebar lets you pick the camera, toggle dark mode, Shamsi dates and the check-in sound.

## Configuration

All settings are optional and are read from environment variables:

| Variable | Default | Description |
|---|---|---|
| `ARIONIX_DB` | `~/.arionix/arionix.db` | SQLite database file |
| `ARIONIX_TOLERANCE` | `0.5` | Maximum face distance for a match (0.1 to 1.0). Lower is stricter. |
| `ARIONIX_COOLDOWN` | `60` | Minimum seconds between two records of the same person |
| `ARIONIX_SCALE` | `0.5` | Frame downscale used for face detection (0.1 to 1.0). Lower is faster. |
| `ARIONIX_MODEL` | `hog` | Detector: `hog` (CPU, fast) or `cnn` (more accurate, needs a GPU) |
| `ARIONIX_MAX_CAMERAS` | `4` | How many camera indexes to probe at start-up |

Example (PowerShell):

```powershell
$env:ARIONIX_TOLERANCE = "0.45"
$env:ARIONIX_COOLDOWN = "300"
python run.py
```

## How it works

1. A camera thread always keeps the newest frame, so a slow processor never builds up delay.
2. A worker thread detects faces on a downscaled copy of the frame (dlib HOG) and computes a 128-dimensional encoding for each face on the full-resolution frame.
3. Each encoding is compared with the registered encodings by Euclidean distance. The closest one within the tolerance is a match.
4. A match is written to the database unless the person was already recorded within the cooldown.

dlib is not thread-safe, so every call into it goes through a lock.

## Project structure

```
run.py                 Entry point
requirements.txt
arionix/
  __main__.py          Command-line / start-up logic
  gui.py               CustomTkinter interface
  service.py           Application logic (no UI code)
  faces.py             Face detection, encoding and matching
  camera.py            Background frame grabbing and frame processing
  database.py          SQLite storage layer
  config.py            Settings from environment variables
  jalali.py            Gregorian to Shamsi date conversion
  assets/              Application icon
tests/                 Tests
tools/                 Helper scripts
docs/                  Documentation and screenshots
```

## Building a Windows executable

```powershell
uv pip install pyinstaller
pyinstaller --noconfirm --clean --windowed `
  --name Arionix `
  --icon arionix\assets\icon.ico `
  --add-data "arionix\assets;arionix\assets" `
  --collect-data customtkinter `
  --collect-data face_recognition_models `
  --hidden-import pkg_resources `
  run.py
```

The program is created in `dist\Arionix\`. Distribute the **whole folder**, not just the `.exe`. If the program does not start on another machine, install the Microsoft Visual C++ Redistributable (x64). If it closes without a message, rebuild without `--windowed` to see the error in a console.

The database lives in the user's home folder, not next to the program, so a new machine starts empty unless that folder is copied.

## Troubleshooting

| Problem | Solution |
|---|---|
| `Failed building wheel for dlib` | Use `dlib-bin` as described in Installation. Do not run `pip install face_recognition` with dependencies. |
| `externally-managed-environment` | The wrong Python is being used. Activate the project's `.venv` first. |
| No camera found | Check **Settings → Privacy & security → Camera** (allow desktop apps), and close other apps that use the camera. |
| `WARN ... can't be used to capture by index` in the console | Harmless. The program probes camera indexes that do not exist. |
| `pkg_resources is deprecated` warning | Harmless as long as `setuptools<81` is installed. |
| Employees disappeared after renaming the project | The database path changed. Copy the old database file to the new `ARIONIX_DB` location. |
| Face not recognised or confused with someone else | Lower `ARIONIX_TOLERANCE` to be stricter, or register with a clearer, well-lit photo. |

## Privacy and security

- Face images are never stored. Only the 128-number encoding of each registered face is saved.
- Encodings are stored as raw `float64` bytes, never with `pickle`, so a tampered database cannot execute code when loaded.
- Face data is biometric data. Make sure you have the consent of the people you register and comply with the privacy rules that apply to you.
