# 📦 Sample — TR-Vision-BasicExercises

---

## 🧠 Remarks and limitations

- This is an **unofficial, unsupported** quickstart sample project. It is an excerpt of the TF7xxx TwinCAT Vision training course offered by [Beckhoff Automation BV](https://www.beckhoff.be). The sample code is provided as-is under the Zero-Clause BSD license.
- This quick start does not replace any official documentation or training. Please refer to the local Beckhoff subsidiary for the complete training session.
- Minimum TwinCAT version: **3.1.4026.15**
- Requires **TF7xxx | TC3 Vision** (>= 4.0.7.4) to be installed
- Requires **TwinCAT HMI** (>= 1.14.3) for exercise 03_Hmi
- The project uses `FB_VN_FileSourceControl` by default (file source). To use a GigE Vision camera, uncomment the `FB_VN_GevCameraControl` or `FB_VN_SimpleCameraControl` line in `MAIN_VISION` and comment out the file source line.
- Sample images are provided in `Documentation/00_Images/`

---

## 🔧 Functionality Overview

The repository contains three progressive exercises, each building on the previous one. Every exercise ships a **Project** folder (editable source), a **Solution** folder (packaged `.tszip`), and a **Templates** folder (starting point).

### `01_Base` — Foundation

Sets up the basic TwinCAT Vision project structure: camera control state machine, image acquisition, and image pipeline (original, work, result, displayable copies). The vision algorithm section is left empty for the trainee to fill in.

### `02_Analysis` — Image Analysis

Extends the base with a complete vision pipeline:

1. Color range check (`F_VN_CheckColorRange`) to threshold by RGB bounds
2. Contour detection (`F_VN_FindContours`) on the thresholded image
3. Contour iteration and filtering by area (> 100 000 px)
4. For each qualifying contour: draw outline, compute center of mass, find enclosing rectangle, and label with text

### `03_Hmi` — HMI Integration

Same PLC logic as 02_Analysis, plus a **TcHmi_Vision** HMI project (`TcHmi_Vision.hmiproj`) that demonstrates the TwinCAT Vision HMI control for live image display in a web-based dashboard.

### `MAIN_VISION`

The central program in every exercise. It drives the camera through a state machine based on `ETcVnCameraState`:

| State | Action |
|-------|--------|
| `TCVN_CS_INITIAL` / `OPENING` / `OPENED` / `STARTACQUISITION` | Start acquisition |
| `TCVN_CS_ACQUIRING` | Get image, run vision pipeline, produce displayable results |
| `TCVN_CS_ERROR` | Reset the camera controller |

Key variables:

| I/O | Name | Type | Description |
|-----|------|------|-------------|
| VAR | `fbControl` | `FB_VN_FileSourceControl` | Camera / file source controller |
| VAR | `ipImage` | `ITcVnImage` | Original acquired image |
| VAR | `ipImageWork` | `ITcVnImage` | Working copy for algorithms |
| VAR | `ipImageResult` | `ITcVnImage` | Result image for drawing overlays |
| VAR | `ipImageDisp` | `ITcVnDisplayableImage` | Displayable original (ADS Image Watch) |
| VAR | `ipImageWorkDisp` | `ITcVnDisplayableImage` | Displayable work image (ADS Image Watch) |
| VAR | `ipImageResultDisp` | `ITcVnDisplayableImage` | Displayable result image (ADS Image Watch) |
| VAR | `tWDStop` | `DINT` | Watchdog stop time (default 38 000 us) |

### Task architecture

| Task | Cycle | Priority | Runs |
|------|-------|----------|------|
| PlcTask | 10 ms | 20 | `MAIN` |
| VisionTask | 10 ms | 1 | `MAIN_VISION` |

---

## 🧪 Example

Minimal vision program that acquires images from a file source and applies a color range check (from exercise 02_Analysis):

```iecst
PROGRAM MAIN_VISION
VAR
    eControlState   : ETcVnCameraState;
    fbControl       : FB_VN_FileSourceControl;
    hr              : HRESULT;
    ipImage         : ITcVnImage;
    ipImageDisp     : ITcVnDisplayableImage;
    ipImageWork     : ITcVnImage;
    ipImageWorkDisp : ITcVnDisplayableImage;
    aLowerBounds    : TcVnVector4_LREAL := [25, 25, 80, 0];
    aUpperBounds    : TcVnVector4_LREAL := [100, 100, 160];
END_VAR

eControlState := fbControl.GetState();

CASE eControlState OF
    TCVN_CS_INITIAL, TCVN_CS_OPENING, TCVN_CS_OPENED, TCVN_CS_STARTACQUISITION:
        hr := fbControl.StartAcquisition();

    TCVN_CS_ACQUIRING:
        hr := fbControl.GetCurrentImage(ipImage);
        IF SUCCEEDED(hr) AND ipImage <> 0 THEN
            hr := F_VN_CopyImage(ipImage, ipImageWork, hr);
            hr := F_VN_TransformIntoDisplayableImage(ipImage, ipImageDisp, S_OK);
            // Apply color range check
            hr := F_VN_CheckColorRange(ipImageWork, ipImageWork, aLowerBounds, aUpperBounds, hr);
            hr := F_VN_CopyIntoDisplayableImage(ipImageWork, ipImageWorkDisp, hr);
        END_IF

    TCVN_CS_ERROR:
        fbControl.Reset();
END_CASE
```

To switch from file source to a live GigE Vision camera, replace `FB_VN_FileSourceControl` with `FB_VN_GevCameraControl` and configure the camera connection in the TwinCAT System Manager.

---

## 🧠 Notes

- The `VisionTask` runs at priority **1** (highest) to ensure deterministic image processing. The `PlcTask` runs at priority **20** for general PLC logic.
- After calling `F_VN_TransformIntoDisplayableImage`, the source `ITcVnImage` interface is no longer valid. Use `F_VN_CopyIntoDisplayableImage` when you still need the source image for further processing.
- The watchdog variables (`hrWD`, `tWDStop`, `tRest`, etc.) are declared but not actively used in the sample code — they are prepared for trainees to implement watchdog monitoring.
- Contour filtering in exercise 02_Analysis uses a threshold of 100 000 pixels. Adjust this value to match your image resolution and object sizes.
- The `F_VN_ConvertColorSpace` call is commented out. Uncomment it (and comment out the `F_VN_CopyImage` line above it) when working with Bayer-encoded camera images instead of RGB file sources.

---

## 🔢 Additional information

**Required libraries**

| Library | Vendor | Purpose |
|---------|--------|---------|
| Tc2_Standard | Beckhoff Automation GmbH | Standard PLC library |
| Tc2_System | Beckhoff Automation GmbH | System functions |
| Tc3_Module | Beckhoff Automation GmbH | TwinCAT module base (system library) |
| Tc3_Vision | Beckhoff Automation GmbH | TwinCAT Vision functions and types |

**Minimum TwinCAT version**: 3.1.4026.15

**License**: [BSD Zero Clause](LICENSE.md)
