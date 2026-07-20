# OBS Studio: WS_EX_TOOLWINDOW capture bypass (patch guide)

This repository contains a patch guide and pre-compiled binaries to bypass the window filtering logic in OBS Studio. 

By default, OBS Studio filters out windows with the `WS_EX_TOOLWINDOW` extended style (such as transparent overlays, click-through cheat menus, and utility widgets) to prevent them from cluttering the "Window Capture" source selection menu. This repository demonstrates how to patch `obs.dll` to force OBS to recognize and capture these windows.

---

## ⚠️ Critical warning: compatibility
* **Do NOT use pre-compiled `obs.dll` binaries from different versions of OBS Studio.** DLL files are strictly version-dependent. Replacing your DLL with a version that does not match your exact OBS Studio installation will cause the application to crash on startup.
* The pre-patched binary provided in this repository is **ONLY** compatible with **OBS Studio v32.1.2 (64-bit)**.
* If you are using a different version of OBS, use the **Manual Patching Guide** below to patch your own DLL. It takes less than 2 minutes.

---

## Supported versions (pre-compiled)
| OBS Studio version | Architecture | Status | File location |
|--------------------|--------------|--------|---------------|
| `v32.1.2`          | `x64`        | Tested & working | `bin/64bit/obs.dll` |

---

## Manual patching guide (for any OBS Studio version)

If you updated OBS or are using a different version, you can easily apply this patch yourself using a disassembler like **IDA Pro**.

### Technical details of the filter
Inside the OBS core (`obs.dll`), the window validation function evaluates window styles before allowing them into the capture pipeline:
```cpp
// Logic inside window-helpers.c (check_window_valid)
if (ex_styles & WS_EX_TOOLWINDOW)
    return false;
```
In the compiled binary, this translates to checking the `0x80` bit (Sign bit of the lowest byte of the return value from `GetWindowLongPtrW(hwnd, -20)`). We want to neutralize the conditional jump that triggers on this bit.

---

### Method A: fast hex patch (recommended)

1. Open your `obs.dll` (located in `obs-studio/bin/64bit/`) in any hex editor (e.g., **HxD**).
2. Search for the following byte sequence (pattern):
   `84 C0 78 ?? 0F BA E6 1E`
   *(In OBS v32.1.2, the sequence is: `84 C0 78 1D 0F BA E6 1E`)*
3. **What these bytes represent:**
   * `84 C0` — `test al, al`
   * `78 XX` — `js short loc_XXXXXXXX` (jump if sign — this is the `WS_EX_TOOLWINDOW` check)
   * `0F BA E6 1E` — `bt esi, 1Eh` (this is the subsequent `WS_CHILD` check, do NOT corrupt it)
4. Change the `78 ??` (or `78 1D`) to **`90 90`** (two `NOP` instructions).
5. Save the file.

---

### Method B: manual patch in IDA Pro

#### Via IDA Pro:
1. Load `obs.dll` into IDA Pro.
2. Locate the window validation function (look for xrefs to `GetWindowLongPtrW` with index `0FFFFFFECh` / `-20`).
3. Locate the block containing:
   ```assembly
   call    cs:GetWindowLongPtrW
   test    al, al
   js      short loc_XXXXXXXX   ; <--- This is the target instruction
   bt      esi, 1Eh
   ```
4. Select the `js short ...` instruction.
5. Go to **Edit -> Patch program -> Change byte...**
6. Change the first two bytes to **`90 90`** (NOPs).
7. Apply patches to the file: **Edit -> Patch program -> Apply patches to input file...**

---

## Post-patch configuration in OBS Studio
Oncepatched, OBS will now see `WS_EX_TOOLWINDOW` windows.

1. Launch OBS Studio.
2. Create a **Window Capture** source.
3. Change the **Capture Method** to **Windows 10 (1903 and up)** (WGC) — this is required to render transparency/alpha channels of `WS_EX_LAYERED` windows properly.
4. Select your overlay/toolwindow from the list. It will now be captured flawlessly.
