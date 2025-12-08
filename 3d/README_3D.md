# 3D Models Documentation

This directory contains 3D STEP models for the Altium Lotus 3 project, including PCB assembly, enclosure, and individual component models.

## Contents

### Main Assembly Files

- **PCB_assembly.step** - Complete PCB assembly with all components positioned
  - PCB dimensions: 100mm × 75mm × 1.6mm (standard FR4)
  - Components positioned logically based on typical layout
  - Mounting holes: 4× M3 clearance holes at corners

- **enclosure.step** - Project enclosure with connector cutouts
  - Dimensions: 120mm × 80mm × 30mm (adjustable - see below)
  - Wall thickness: 2.5mm
  - Features:
    - NFC/BLE window on top (40mm × 30mm) for wireless communication
    - XLR 5-pin connector cutout (front right) for DMX
    - 2-pin screw terminal cutout (front left) for 0-10V
    - DALI 2-pin screw terminal cutout (back left)
    - Power barrel jack cutout (back center)
    - 4× mounting bosses with M3 threaded inserts
    - Ventilation slots on sides for heat dissipation

### Component Library (components_3d/)

All individual component STEP files are located in the `components_3d/` subdirectory:

#### Main ICs and Modules
- **ESP32-C6-WROOM.step** - ESP32-C6 WiFi/BLE module (18mm × 25.5mm × 3.2mm)
- **ICL1122-radar-module.step** - mmWave radar presence detection module (25mm × 25mm × 3mm)
- **VEML6035-ALS.step** - Ambient light sensor, SOT-23-6 package
- **MCP4725-DAC.step** - 12-bit DAC for 0-10V output, SOT-23-6 package
- **ISO3082-RS485.step** - RS-485 transceiver, SOIC-8 package
- **OPA192-opamp.step** - Precision op-amp for 0-10V scaling, SOT-23-5 package
- **TLV431-Vref.step** - Voltage reference, SOT-23-3 package

#### Power Components
- **CE5R5155CF-ZJ-supercap.step** - 1.5F/5.5V supercapacitor (D=11.5mm, H=5.5mm)

#### Connectors
- **screw-terminal-2pin-0-10V.step** - 2-pin screw terminal for 0-10V output
- **XLR-5pin-DMX.step** - 5-pin XLR connector for DMX512 communication
- **DALI-screw-terminal-2pin.step** - 2-pin screw terminal for DALI bus
- **power-barrel-jack.step** - DC power barrel jack connector

#### Hardware
- **standoff-M3-10mm.step** - M3 standoff, 10mm height (×4 for PCB mounting)
- **screw-M3-6mm.step** - M3 mounting screw, 6mm length

## Model Details and Sources

### Level of Detail

All models in this repository are **placeholder models** with accurate dimensional envelopes based on manufacturer datasheets. These placeholders are suitable for:
- Mechanical fit verification
- Enclosure design
- Clearance checking
- Assembly visualization

The models use simplified geometry (rectangular/cylindrical shapes) rather than detailed features.

### Official Manufacturer Models

For higher fidelity models, we recommend obtaining official STEP files from manufacturers:

| Component | Manufacturer | Official Model Search |
|-----------|--------------|----------------------|
| ESP32-C6-WROOM | Espressif | https://www.espressif.com/en/support/download/documents |
| ICL1122 | InnoSenT | Contact manufacturer for 3D models |
| VEML6035 | Vishay | https://www.vishay.com/ppg?84367 |
| MCP4725 | Microchip | https://www.microchip.com/en-us/product/MCP4725 |
| ISO3082 | Texas Instruments | https://www.ti.com/product/ISO3082 |
| OPA192 | Texas Instruments | https://www.ti.com/product/OPA192 |
| TLV431 | Texas Instruments | https://www.ti.com/product/TLV431 |
| CE5R5155CF-ZJ | Seiko | Contact manufacturer or search on component distributors |
| XLR Connectors | Neutrik, Amphenol | Check manufacturer 3D model libraries |

### Replacing Placeholder Models

To replace a placeholder with an official STEP model:

1. Download the official STEP file from the manufacturer
2. Replace the placeholder file in `components_3d/` with the same filename
3. Update the `source_of_3d_model` column in `BOM_3D.csv` with the manufacturer link
4. If needed, adjust component position in `PCB_assembly.step` by regenerating the assembly

## Import Instructions

### Importing into CAD Software

#### Altium Designer
1. Open your PCB project
2. Go to View → 3D View
3. Right-click on a component → Properties → Models
4. Add → Generic 3D → Browse to component STEP file
5. Adjust position/rotation as needed

#### KiCad
1. Open the PCB Editor
2. Select a footprint
3. Edit Footprint Properties → 3D Settings
4. Add 3D Shape → Browse to STEP file
5. Adjust scale, offset, and rotation

#### FreeCAD / CAD Software
1. File → Import → Select STEP file
2. Assembly can be imported as a complete unit
3. Individual components can be imported for custom assembly

#### Fusion 360
1. File → Upload → Select STEP file
2. Insert into Design → Browse to STEP file
3. Use "Insert into Current Design" for assemblies

### Viewing STEP Files

Free STEP file viewers:
- **FreeCAD** (Windows/Mac/Linux) - https://www.freecadweb.org/
- **eDrawings** (Windows/Mac) - https://www.edrawingsviewer.com/
- **Online STEP Viewer** - https://www.viewstl.com/ or https://www.sharecad.org/

## Adjusting Enclosure Dimensions

The enclosure can be modified to fit different requirements:

### Current Dimensions
- Outer: 120mm (W) × 80mm (D) × 30mm (H)
- Wall thickness: 2.5mm
- Inner clearance: ~115mm × 75mm × 27.5mm

### To Adjust Dimensions

Option 1: **Regenerate with Python Script**
1. Edit the `generate_enclosure.py` script in the project repository
2. Modify `outer_width`, `outer_depth`, `outer_height` variables
3. Run: `python3 generate_enclosure.py`
4. Replace `enclosure.step` with the newly generated file

Option 2: **Edit in CAD Software**
1. Import `enclosure.step` into your CAD software
2. Use parametric editing tools to adjust dimensions
3. Re-export as STEP

### Connector Position Adjustments

To move connector cutouts:
- Edit `generate_enclosure.py` script
- Modify the `moveTo(x, y)` coordinates for each connector cutout
- Regenerate the enclosure STEP file

## Technical Notes

### Design Decisions

- **ICL1122 Power**: Module powered at 3.3V (ESP32 native voltage)
- **Cable Length**: ICL1122 assumed <30cm cable. For longer runs (>1m), consider RS-485 level conversion
- **Op-Amp Selection**: OPA192 chosen for precision 0-10V output scaling
- **PCB Layout**: Logical component placement without actual .kicad_pcb file (approximation for mockup)

### Coordinate System

All STEP files use the following coordinate system:
- **X-axis**: Width (left-right)
- **Y-axis**: Depth (front-back)
- **Z-axis**: Height (bottom-up)
- **Origin**: Center of component/board/enclosure

### File Formats

- All models are in STEP AP214 format for maximum compatibility
- Units: millimeters (mm)
- No colors/textures (geometry only)

## Licenses and Attribution

### Placeholder Models
All placeholder models in this repository were created specifically for this project and are provided as-is for mechanical design reference.

License: These placeholder models are provided under the MIT License.

### Official Manufacturer Models
When using official STEP models from manufacturers:
- Check the manufacturer's license terms
- Manufacturer models are typically provided for design purposes only
- Redistribution may be restricted - refer to manufacturer's terms

### Usage Rights
The placeholder models can be used for:
- ✅ Internal design and development
- ✅ Manufacturing and assembly
- ✅ Documentation and presentations
- ✅ Derivative works within your project

## Validation and Accuracy

⚠️ **Important**: These are placeholder models with approximate dimensions.

Before finalizing mechanical design:
1. ✅ Verify all dimensions against manufacturer datasheets
2. ✅ Check component pin positions and orientations
3. ✅ Validate connector cutout sizes with actual parts
4. ✅ Test fit with physical prototypes
5. ✅ Replace placeholders with official models where possible

## Contributing

To improve these models:
1. Source official STEP files from manufacturers
2. Update `BOM_3D.csv` with official model links
3. Submit corrections for dimensional inaccuracies
4. Add additional components as needed

## Support

For questions or issues with 3D models:
- Review component datasheets for official dimensions
- Check manufacturer websites for official 3D models
- Open an issue in the project repository

---

**Last Updated**: 2025-12-08
**Model Format**: STEP AP214
**Units**: Millimeters
**Coordinate System**: Right-handed (X-Y-Z)
