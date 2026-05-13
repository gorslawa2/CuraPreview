# Snapmaker G-Code Writer Plugin for Cura 5.x

## Overview

This plugin is a custom G-code writer for **Snapmaker 3D printers** that integrates with **Ultimaker Cura 5.x** (including version 5.11 and later). It adds Snapmaker-specific headers and embeds a **preview thumbnail** directly into the G-code file, enabling the printer's display to show a visual preview of the model before printing.

---

## Table of Contents

- [Features](#features)
- [Compatibility](#compatibility)
- [Installation](#installation)
- [Usage](#usage)
- [Technical Details](#technical-details)
  - [API Compatibility Fix](#1-api-compatibility-fix-def-write)
  - [Thumbnail Generation Fix](#2-thumbnail-generation-fix-snapshot-returned-none)
- [How the Thumbnail Generation Works](#how-the-thumbnail-generation-works)
- [Troubleshooting](#troubleshooting)
- [File Structure](#file-structure)
- [Contributing](#contributing)
- [License](#license)

---

## Features

- ✅ **Snapmaker-Specific Headers**: Automatically adds required metadata headers that Snapmaker printers need to properly interpret the G-code file.
- ✅ **Embedded Thumbnail Preview**: Generates and embeds a PNG thumbnail (300x300 or 200x200 pixels) in Base64 format directly into the G-code file.
- ✅ **Cura 5.x Compatibility**: Fully updated to work with Cura 5.x API changes, including Qt6 integration.
- ✅ **Multiple Header Versions**: Supports both legacy (v0) and modern (v1) Snapmaker header formats, automatically detecting the correct version based on the selected printer model.
- ✅ **Multi-Extruder Support**: Correctly handles multi-extruder configurations and includes extruder-specific information in the headers.
- ✅ **Robust Error Handling**: Includes retry mechanisms and fallback options to ensure thumbnail generation succeeds even under challenging conditions.

---

## Compatibility

### Supported Software
- **Ultimaker Cura**: Version 5.0 and above (tested on 5.11)
- **Operating Systems**: Windows, macOS, Linux (any system that supports Cura 5.x)

### Supported Printers
- Snapmaker A150
- Snapmaker A250
- Snapmaker A350
- Snapmaker 2.0 models (with appropriate header version support)
- Other Snapmaker machines configured in the plugin's machine database

---

## Installation

### Step 1: Download the Plugin

Clone this repository or download the plugin files to your local machine:

```bash
git clone <repository-url>
```

Or manually copy the `gcode_writer` folder.

### Step 2: Locate Cura's Plugin Directory

The plugin needs to be placed in Cura's plugin directory. The location varies by operating system:

- **Windows**: `%APPDATA%\cura\5.x\plugins\`
- **macOS**: `~/Library/Application Support/cura/5.x/plugins/`
- **Linux**: `~/.config/cura/5.x/plugins/`

Replace `5.x` with your specific Cura version (e.g., `5.11`).

### Step 3: Install the Plugin

1. Copy the `gcode_writer` folder into the plugins directory.
2. Ensure the folder structure looks like this:
   ```
   plugins/
   └── gcode_writer/
       ├── __init__.py
       ├── SnapmakerGCodeWriter.py
       └── config.py (if applicable)
   ```

### Step 4: Restart Cura

Close and reopen Ultimaker Cura to load the new plugin.

### Step 5: Verify Installation

1. Open Cura and go to **Marketplace** → **Installed Plugins**.
2. Look for the Snapmaker G-Code Writer in the list.
3. If it doesn't appear, check Cura's log file for errors (usually located in the same directory as the plugins folder).

---

## Usage

### Basic Workflow

1. **Load Your Model**: Import your 3D model (STL, OBJ, etc.) into Cura.
2. **Configure Print Settings**: Set up your print parameters as usual.
3. **Slice the Model**: Click the **Slice** button to generate the G-code.
4. **Export G-code**: 
   - Go to **File** → **Save to File...**
   - Choose the desired location and filename.
   - The plugin will automatically add the Snapmaker headers and thumbnail.

### Verifying the Thumbnail

After exporting:

1. Transfer the G-code file to your Snapmaker printer (via USB, SD card, or Wi-Fi).
2. Open the file on the printer's touchscreen interface.
3. You should see a visual preview of your model instead of a generic icon.

---

## Technical Details

This plugin addresses two critical issues that arose when migrating from Cura 4.x to Cura 5.x:

### 1. API Compatibility Fix (`def write`)

#### The Problem
In Cura 5.x, the base class `MeshWriter` changed the signature of the `write` method. The new signature includes additional parameters (`mime_type` and `**kwargs`) that were not present in earlier versions.

**Old Signature (Cura 4.x):**
```python
def write(self, stream, node, mode=FileWriter.OutputMode.BinaryMode) -> None:
```

**New Signature (Cura 5.x):**
```python
def write(self, stream, node, mode=FileWriter.OutputMode.BinaryMode, mime_type=None, **kwargs) -> bool
```

If the plugin doesn't match this signature, Python raises an argument mismatch error, causing the export to fail.

#### The Solution
We updated the `write` method in `SnapmakerGCodeWriter.py`:

```python
def write(self, stream, node, mode=FileWriter.OutputMode.BinaryMode, mime_type=None, **kwargs) -> bool:
    """Writes the G-code for the entire scene to a stream."""
    
    if mode != MeshWriter.OutputMode.TextMode:
        Logger.log("e", "GCodeWriter does not support non-text mode.")
        self.setInformation(catalog.i18nc("@error:not supported", "GCodeWriter does not support non-text mode."))
        return False
    
    # ... rest of the implementation
```

**Key Changes:**
- Added `mime_type=None` parameter for compatibility.
- Added `**kwargs` to catch any future arguments Cura developers might introduce.
- Changed return type from `None` to `bool` to explicitly indicate success (`True`) or failure (`False`).

---

### 2. Thumbnail Generation Fix (`Snapshot returned None`)

#### The Problem
Cura 5.x uses **Qt6** for its user interface and rendering engine. The thumbnail generation process (`Snapshot.snapshot`) is asynchronous and depends on the OpenGL rendering pipeline.

When the plugin requests a snapshot "immediately" during the save operation, the rendering engine often hasn't finished drawing the scene, or it's in a blocked state. This results in:
- `None` being returned instead of an image.
- Empty or missing thumbnails in the exported G-code.
- Log messages like `Snapshot returned None`.

#### The Solution
We completely rewrote the `__generateThumbnail` method to implement a **retry mechanism with forced UI updates**.

---

## How the Thumbnail Generation Works

The new thumbnail generation process works as follows:

### Step 1: Import Required Modules

```python
import os
import time
from PyQt6.QtCore import QBuffer, QCoreApplication
from UM.Application import Application
from cura.Snapshot import Snapshot
```

- `QCoreApplication`: Allows us to force Qt to process pending events (UI updates).
- `time`: Provides sleep functionality for timing control.
- `Snapshot`: Cura's built-in screenshot utility.

### Step 2: Force Scene Initialization

Before attempting to capture the thumbnail, we ensure the scene is properly initialized:

```python
scene = Application.getInstance().getController().getScene()
```

### Step 3: Retry Loop (The "Paparazzi" Approach)

Instead of making a single snapshot request, we implement a loop that makes up to **20 attempts**:

```python
image = None
for attempt in range(20):
    QCoreApplication.processEvents()  # Force UI update NOW
    time.sleep(0.05)                  # Wait 50ms for GPU to render
    
    # Try to capture 300x300 thumbnail (Snapmaker standard)
    image = Snapshot.snapshot(300, 300)
    
    if image:  # Success! Exit the loop
        break
```

**Why This Works:**
- `QCoreApplication.processEvents()`: Forces the Qt event loop to process all pending events, including window repaints and OpenGL rendering commands. Without this, Cura might be "stuck" in the save operation and never update the viewport.
- `time.sleep(0.05)`: Gives the graphics card time to complete the rendering. 50ms is short enough to keep the operation fast but long enough for most frames to render.
- **20 attempts**: Provides up to 1 second total (20 × 50ms) for the scene to render, which is more than sufficient for most models.

### Step 4: Fallback Option

If all 20 attempts with 300×300 resolution fail (rare, but possible with certain GPU drivers), we try a smaller resolution:

```python
if not image:
    image = Snapshot.snapshot(200, 200)  # Fallback to smaller size
```

Some graphics drivers have alignment requirements (e.g., textures must be multiples of 16 or 64 pixels). A smaller size may bypass these issues.

### Step 5: Validate the Image

Even if an image object is returned, it might be invalid (zero-size or corrupted):

```python
if not image or image.isNull():
    return ""  # Return empty string to avoid corrupting the G-code
```

### Step 6: Encode to Base64

Once we have a valid image:

```python
buffer = QBuffer()
buffer.open(QBuffer.OpenModeFlag.ReadWrite)
image.save(buffer, "PNG")
base64_bytes = base64.b64encode(buffer.data())
base64_message = base64_bytes.decode("ascii")
buffer.close()

return "data:image/png;base64," + base64_message
```

The thumbnail is encoded as a Base64 string and prefixed with the data URI scheme, which is the format expected by Snapmaker printers.

### Step 7: Exception Handling

All operations are wrapped in a try-except block to prevent crashes:

```python
except Exception as e:
    Logger.logException("w", "Failed to create thumbnail for G-code")
    return ""
```

---

## Troubleshooting

### Issue: "GCodeWriter does not support non-text mode"

**Cause**: The plugin only supports text mode output.

**Solution**: Ensure you're exporting in the correct format. This error should not occur during normal usage.

---

### Issue: "Please prepare G-code before exporting"

**Cause**: The scene hasn't been sliced yet, or the G-code dictionary is missing.

**Solution**: 
1. Click the **Slice** button before saving.
2. Make sure your model is properly loaded and positioned.

---

### Issue: No Thumbnail Appears on Printer

**Possible Causes**:
1. **Insufficient Rendering Time**: The model is very complex and takes longer than 1 second to render.
2. **GPU Driver Issues**: Outdated or incompatible graphics drivers.
3. **Headless Mode**: Running Cura without a display server (e.g., via SSH without X11 forwarding).

**Solutions**:
1. **Increase Retry Attempts**: Edit `SnapmakerGCodeWriter.py` and increase the range in the retry loop:
   ```python
   for attempt in range(30):  # Increase from 20 to 30
   ```
2. **Update Graphics Drivers**: Ensure your GPU drivers are up to date.
3. **Use Display Server**: If running headlessly, use a virtual display server like Xvfb.
4. **Check Logs**: Enable debug logging to see detailed error messages.

---

### Issue: Plugin Not Loading

**Possible Causes**:
1. Incorrect installation path.
2. Missing dependencies.
3. Version incompatibility.

**Solutions**:
1. Verify the plugin is in the correct directory (see [Installation](#installation)).
2. Check Cura's log file for error messages.
3. Ensure you're using Cura 5.x (this plugin is not compatible with Cura 4.x).

---

### Debug Logging

To enable detailed logging, add the following to your Cura configuration or modify the plugin code temporarily:

```python
debug_log_path = "/path/to/debug_log.txt"

try:
    with open(debug_log_path, "a") as f:
        f.write(f"Thumbnail generated successfully. Length: {len(base64_message)}\n")
except:
    pass
```

This will append thumbnail generation status to a log file for troubleshooting.

---

## File Structure

```
gcode_writer/
├── __init__.py                    # Plugin initialization and registration
├── SnapmakerGCodeWriter.py        # Main G-code writer implementation
├── SnapmakerGCodeWriter_BACKUP.py # Backup of original file (for reference)
├── config.py                      # Configuration and machine definitions (if applicable)
├── editCode.md                    # Detailed editing guide (Russian language)
├── debug_log.txt                  # Debug log file (generated at runtime)
└── work_log.txt                   # Work log (generated during development)
```

### Key Files

- **`SnapmakerGCodeWriter.py`**: The core plugin file containing all the logic for writing G-code and generating thumbnails.
- **`__init__.py`**: Registers the plugin with Cura and makes it discoverable.
- **`editCode.md`**: Detailed documentation of the fixes applied (in Russian).

---

## Contributing

Contributions are welcome! If you encounter issues or have suggestions for improvements:

1. **Fork the Repository**: Create your own fork of the project.
2. **Create a Branch**: Make your changes in a dedicated branch.
3. **Test Thoroughly**: Ensure your changes work across different Cura versions and printer models.
4. **Submit a Pull Request**: Describe your changes and the problem they solve.

### Development Guidelines

- Follow PEP 8 style guidelines for Python code.
- Add comments for complex logic.
- Test on multiple Cura versions if possible.
- Update documentation to reflect any changes.

---

## License

This project is provided as-is for the benefit of the Snapmaker and Cura communities. Please respect the licensing terms of Ultimaker Cura and Snapmaker when using this plugin.

---

## Acknowledgments

- **Ultimaker**: For developing Cura and maintaining an extensible plugin architecture.
- **Snapmaker**: For creating accessible 3D printers and supporting custom G-code formats.
- **Community Contributors**: For identifying issues and testing fixes across different hardware configurations.

---

## Changelog

### Version 1.0 (Current)
- ✅ Fixed API compatibility with Cura 5.x (`write` method signature).
- ✅ Implemented robust thumbnail generation with retry mechanism.
- ✅ Added fallback resolution for problematic GPU drivers.
- ✅ Enhanced error handling and logging.
- ✅ Support for multiple Snapmaker header versions (v0 and v1).
- ✅ Multi-extruder configuration support.

---

## Support

For questions, issues, or feature requests:
- Check the [Troubleshooting](#troubleshooting) section.
- Review the `editCode.md` file for detailed technical notes.
- Contact the maintainer or open an issue on the repository.

---

**Happy Printing! 🖨️**
