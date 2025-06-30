# FujiFilm.pm Module Outline

**ExifTool Version:** 13.26
**Module Version:** 1.96
**Document Version:** 1.0
**Last Updated:** 2025-06-30

## Overview

FujiFilm.pm handles reading and writing of FujiFilm maker notes in EXIF data and FujiFilm RAW (RAF) images. This module supports extensive camera-specific metadata including advanced face detection, video metadata, camera settings, and proprietary RAW format processing for FujiFilm cameras from the FinePix series through modern X-series and GFX cameras.

## Module Structure

### Tag Tables (13 total)

- **Main** - Primary FujiFilm maker notes tags (165+ tags)
- **PrioritySettings** - AF and exposure priority configuration
- **FocusSettings** - Autofocus system configuration
- **AFCSettings** - AF-C (continuous autofocus) settings
- **DriveSettings** - Drive mode and burst settings
- **FaceRecInfo** - Face recognition database information
- **RAFHeader** - RAF file header metadata
- **RAF** - RAF directory tags for raw image data
- **RAFData** - Additional RAF format data
- **IFD** - FujiIFD directory processing
- **FFMV** - FujiFilm movie format tags
- **MOV** - QuickTime movie tags for FujiFilm
- **MRAW** - M-RAW format header processing

### Key Data Structures

- **%testedRAF** - Lookup table of tested RAF version numbers mapped to camera models
- **%faceCategories** - Face recognition category definitions (Partner, Family, Friend)
- **Tag format distributions** - 137 hex format tags, 28 string format tags

## Processing Architecture

### 1. Main Processing Flow

Entry point through standard ExifTool maker notes processing, with specialized RAF file processing via ProcessRAF() function. The module handles both embedded JPEG preview extraction and proprietary RAF directory parsing.

### 2. Special Processing Functions

- **ProcessFujiDir()** - Processes FujiFilm RAF directories with custom binary format
- **ProcessFaceRec()** - Extracts face recognition database information including names, birthdays, and categories
- **ProcessMRAW()** - Handles M-RAW format headers for specific camera models
- **ProcessRAF()** - Main RAF file processor managing multiple directory types and embedded JPEG previews

## Key Features

### Advanced Face Detection System
- Comprehensive face element detection (Face, Eyes, Body, Head, Animal features)
- Support for various subject types: humans, animals, birds, vehicles, aircraft
- Face recognition database with personal information (names, birthdays, categories)
- Face positioning data in full-resolution coordinates

### RAF Raw Format Support
- Complete RAF file reading and writing capability
- Version tracking for tested RAF formats across camera models
- Support for M-RAW compressed format
- Raw image dimension and cropping information
- X-Trans sensor layout data (36-byte pattern)
- White balance coefficient arrays for different lighting conditions

### Video Metadata Processing  
- Frame rate, dimensions, and compression settings
- Video codec information (Log GOP, All Intra)
- High-speed recording detection
- Movie format integration (FFMV/MOV)

### Camera-Specific Features
- Internal serial number decoding with manufacture date extraction
- Film simulation modes (Provia, Velvia, Astia, Classic Chrome, Acros, etc.)
- Dynamic range settings (Auto, 100%, 230%, 400%)
- Image stabilization detection (Optical, Sensor-shift, Digital, IBIS/OIS)
- Scene recognition (Portrait, Landscape, Night Scene, Macro)

## Special Patterns

### RAF Directory Processing Pattern
Custom binary format parsing with big-endian byte order, variable-length entries identified by tag/length pairs, and automatic format detection for unknown tags.

### Face Recognition Integration
Sophisticated face data extraction combining detection coordinates, element classification, and personal database information with category-based organization.

### Multi-Format Support Pattern
Unified handling of JPEG maker notes, RAF raw files, and video formats through format-specific tag tables and processing functions.

## Usage Notes

- RAF files contain both proprietary metadata and embedded JPEG previews
- Face recognition requires compatible camera models with database functionality
- M-RAW format support varies by camera model and firmware version
- Internal serial numbers contain embedded manufacture dates and camera body numbers
- Video tags are only present in movie files from capable camera models

## Debugging Tips

- Use `-v` option to see RAF directory parsing details
- Check RAF version against %testedRAF lookup for write compatibility
- Face recognition issues often relate to database corruption or unsupported models
- M-RAW processing failures typically indicate short or corrupted headers
- RAF pointer errors usually indicate file corruption or unsupported format variations
