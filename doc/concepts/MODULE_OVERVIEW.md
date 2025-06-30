# ExifTool Module Overview

**ExifTool Version:** 13.26  
**Document Version:** 1.0  
**Last Updated:** 2025-06-29

This document provides a categorized overview of ExifTool library modules to help developers quickly understand the codebase structure.

## Module Categories

### Camera/Manufacturer Modules (Maker Notes)

These modules handle proprietary metadata formats from camera manufacturers. They are among the most complex and frequently updated.

| Module       | Size  | Complexity | Description                             |
| ------------ | ----- | ---------- | --------------------------------------- |
| Canon.pm     | 405KB | High       | Canon maker notes, extensive tag tables |
| Nikon.pm     | 556KB | Very High  | Nikon maker notes, most complex module  |
| Sony.pm      | 471KB | Very High  | Sony maker notes, rapidly evolving      |
| Pentax.pm    | 231KB | High       | Pentax/Ricoh maker notes                |
| Olympus.pm   | 148KB | High       | Olympus maker notes                     |
| Panasonic.pm | 116KB | Medium     | Panasonic/Lumix maker notes             |
| FujiFilm.pm  | 78KB  | Medium     | Fujifilm maker notes                    |
| Samsung.pm   | 71KB  | Medium     | Samsung maker notes                     |
| Kodak.pm     | 123KB | Medium     | Kodak maker notes                       |
| Casio.pm     | 81KB  | Medium     | Casio maker notes                       |

**Supplementary modules:**

- CanonCustom.pm - Canon custom functions
- CanonVRD.pm - Canon VRD recipe data
- CanonRaw.pm - Canon CRW/CIFF format
- NikonCustom.pm - Nikon custom settings
- NikonCapture.pm - Nikon Capture editing
- NikonSettings.pm - Additional Nikon settings

### File Format Modules

Handle specific file formats and containers.

**Major formats:**
| Module | Size | Purpose |
|--------|------|---------|
| QuickTime.pm | 455KB | MOV/MP4 container, very complex |
| XMP.pm | 192KB | Adobe XMP metadata |
| PDF.pm | 100KB | PDF metadata |
| DICOM.pm | 250KB | Medical imaging format |
| Exif.pm | 268KB | Core EXIF/TIFF handler |
| JPEG.pm | 32KB | JPEG markers and segments |
| PNG.pm | 104KB | PNG chunks |

**Media containers:**

- MXF.pm (258KB) - Material eXchange Format
- RIFF.pm (88KB) - AVI and related formats
- Matroska.pm (49KB) - MKV/WebM format
- ASF.pm (35KB) - Windows Media format

**Raw formats:**

- DNG.pm - Adobe Digital Negative
- PanasonicRaw.pm - Panasonic RW2
- MinoltaRaw.pm - Minolta MRW
- SigmaRaw.pm - Sigma X3F

### Metadata Standard Modules

| Module         | Purpose                         |
| -------------- | ------------------------------- |
| IPTC.pm        | IPTC/IIM metadata standard      |
| ICC_Profile.pm | Color management profiles       |
| DarwinCore.pm  | Biodiversity metadata           |
| MWG.pm         | Metadata Working Group mappings |

### Infrastructure/Utility Modules

| Module            | Size  | Purpose                          |
| ----------------- | ----- | -------------------------------- |
| TagLookup.pm      | 740KB | Auto-generated tag lookup tables |
| BuildTagLookup.pm | 128KB | Generates TagLookup.pm           |
| Writer.pl         | 295KB | Core writing logic               |
| Validate.pm       | 27KB  | Tag validation functions         |
| Fixup.pm          | 14KB  | Offset correction utilities      |
| Charset.pm        | 17KB  | Character encoding handler       |

### Special Processing Scripts

These .pl files handle complex operations:

- WriteExif.pl - EXIF writing logic
- WriteQuickTime.pl - QuickTime atom writing
- QuickTimeStream.pl - QuickTime stream processing
- XMP2.pl - XMP structure handling

## Module Complexity Indicators

**Very High Complexity (>400KB):**

- Nikon.pm, Sony.pm, Canon.pm, QuickTime.pm
- Require deep understanding of proprietary formats
- Frequent updates with new camera models

**High Complexity (200-400KB):**

- NikonCustom.pm, Pentax.pm, Exif.pm, DICOM.pm, MXF.pm
- Complex structures but stable formats

**Medium Complexity (50-200KB):**

- Most camera modules, standard formats
- Well-documented structures

**Low Complexity (<50KB):**

- Simple formats, minimal processing
- Good starting points for understanding patterns

## Key Dependencies

1. **All modules** depend on the main ExifTool.pm
2. **Camera modules** typically use:
   - Exif.pm for IFD processing
   - Common functions from ExifTool.pm
3. **Format modules** may use:
   - Appropriate metadata modules (XMP, IPTC, Exif)
   - Charset.pm for encoding
4. **Binary formats** often use:
   - BinaryData tables
   - Fixup.pm for offset handling

## Update Frequency

**Frequently Updated (with each camera release):**

- Canon.pm, Nikon.pm, Sony.pm, Olympus.pm, Panasonic.pm

**Occasionally Updated (format changes):**

- QuickTime.pm, XMP.pm, PDF.pm

**Rarely Updated (stable formats):**

- JPEG.pm, PNG.pm, GIF.pm, BMP.pm

## Where to Start

For understanding ExifTool's architecture:

1. Read lib/Image/ExifTool/README first
2. Study BMP.pm or GIF.pm for simple format examples
3. Look at MakerNotes.pm for maker note detection
4. Examine Canon.pm for complex maker note handling
5. Review XMP.pm for structured metadata
