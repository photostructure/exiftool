# DJI.pm Module Outline

**ExifTool Version:** 13.26  
**Module Version:** 1.14  
**Document Version:** 1.0  
**Last Updated:** 2025-06-30

## Overview

The DJI module extracts metadata from DJI drone cameras, covering traditional maker notes, thermal imaging parameters, XMP metadata, beauty/glamour settings, and modern protobuf-format timed metadata. It supports a wide range of DJI devices from early Phantom drones to modern Mavic, Matrice, Osmo Action, and Avata models. The module handles complex thermal imaging data and implements sophisticated protobuf parsing for device-specific protocols.

## Module Structure

### Tag Tables (12 total)

**Core Formats:**
- `Main` - Traditional maker notes (early Phantom drones)
- `Info` - Debug info format with bracket-delimited data
- `XMP` - XMP drone-dji namespace tags for location and orientation
- `Glamour` - Beauty/selfie enhancement settings (semicolon-delimited)

**Thermal Imaging:**
- `ThermalParams` - ZH20T thermal parameters (APP4 format)
- `ThermalParams2` - M3T, H20N, M2EA, M30T thermal parameters
- `ThermalParams3` - Alternative M30T thermal parameters

**Protobuf Format:**
- `Protobuf` - Main protobuf metadata table with protocol-specific tags
- `DroneInfo` - Drone orientation data (roll/pitch/yaw)
- `GimbalInfo` - Gimbal orientation data (pitch/roll/yaw)
- `FrameInfo` - Video frame information (width/height/rate)
- `GPSInfo` - GPS coordinate and unit information

### Key Data Structures

**Protocol Support:**
- `%knownProtocol` - Validated protobuf protocols by device:
  - dvtm_ac203.proto (Osmo Action 4)
  - dvtm_ac204.proto (Osmo Action 5)
  - dvtm_AVATA2.proto (Avata 2)
  - dvtm_wm265e.proto (Mavic 3)
  - dvtm_pm320.proto (Matrice 30)
  - dvtm_Mini4_Pro.proto (Mini 4 Pro)

**Conversion Utilities:**
- `%convFloat2` - Two-decimal float formatting template
- Complex thermal parameter structures with multiple magic numbers
- Protocol-specific tag hierarchies using field number notation

## Processing Architecture

### 1. Main Processing Flow

**Multi-Format Support:**
- Traditional EXIF maker notes processing
- APP4 thermal parameter extraction
- XMP namespace processing
- Protobuf binary format parsing
- Text-based settings parsing

**Format Detection:**
- Automatic protocol detection for protobuf data
- Magic number validation for thermal parameters
- Format-specific processing pipelines
- Unknown protocol warning system

### 2. Special Processing Functions

**ProcessDJIInfo($$$)**
- Parses bracket-delimited debug information format
- Uses regex pattern matching: `\[(.*?)\](?=(\[|$))`
- Handles key-value pairs separated by colons
- Supports both string and binary value types
- Dynamic tag creation with MakeTagInfo

**ProcessSettings($$$)**
- Parses semicolon-separated beauty/glamour settings
- Uses key=value format parsing
- Handles DJI selfie enhancement parameters
- Creates tags dynamically for unknown settings

## Key Features

### 1. Comprehensive Device Support

**Classic Drones:**
- DJI Phantom series (maker notes format)
- Basic orientation and speed data
- Camera gimbal positioning

**Modern Drones:**
- Mavic 3, Matrice 30, Mini 4 Pro
- Avata 2, Osmo Action 4/5
- Real-time sensor data via protobuf

### 2. Thermal Imaging Capabilities

**Professional Thermal Cameras:**
- ZH20T, M3T, H20N, M2EA, M30T support
- Thermal calibration parameters (K1-K4, KF, B1-B2)
- Environmental compensation (humidity, distance, emissivity)
- Temperature measurement coefficients

**Thermal Data Processing:**
- Multiple thermal parameter formats
- Magic number validation (0xaa551206, 0x55aa1206, 0xaa553800)
- Device-specific parameter layouts
- Ambient and reflection temperature handling

### 3. Advanced Protobuf Integration

**Protocol-Specific Processing:**
- Device-specific .proto file support
- Hierarchical tag ID system (protocol_field-subfield-subsubfield)
- Format specification per tag (float, rational, unsigned, etc.)
- Unknown protocol warning system

**Temporal Metadata:**
- Frame-synchronized sensor data
- GPS coordinates with time correlation
- Camera settings per frame
- Accelerometer and orientation data

## Special Patterns

### 1. Protobuf Tag Hierarchy

DJI uses a sophisticated tag naming system:
```
Format: protocol_field-subfield-subsubfield
Example: dvtm_wm265e_3-2-2-1 (Mavic 3, ISO setting)
- dvtm_wm265e: Protocol identifier
- 3-2-2-1: Hierarchical field numbers
```

**Device-Specific Tags:**
- Each drone model has unique protocol file
- Field numbers vary between device types
- Common data types across protocols
- Missing tags default to value 0

### 2. Thermal Parameter Processing

**Multi-Format Support:**
- ThermalParams: ZH20T format with specific magic numbers
- ThermalParams2: M3T format with float parameters
- ThermalParams3: Alternative M30T format with scaled integers

**Calibration Data:**
- K-coefficients for thermal calibration
- B-coefficients for brightness adjustment
- Environmental compensation parameters
- Device identification strings

### 3. XMP Integration

**Drone-Specific Namespace:**
- Custom drone-dji XMP namespace
- GPS coordinate handling with DMS conversion
- Misspelled tag support (GpsLongtitude vs GpsLongitude)
- RTK (Real-Time Kinematic) GPS support

## Usage Notes

### Device Support

**Supported Models:**
- **Phantom Series:** Early models with maker notes
- **Mavic Series:** Mavic 3 with protobuf metadata
- **Matrice Series:** Matrice 30 professional drones
- **Mini Series:** Mini 4 Pro consumer drones
- **Osmo Series:** Action 4/5 action cameras
- **Avata Series:** Avata 2 FPV drones

**Thermal Imaging:**
- ZH20T, M3T, H20N, M2EA, M30T
- Professional thermal imaging payloads
- RJPEG format support

### Metadata Types

**Flight Data:**
- GPS coordinates and altitude (absolute/relative)
- Drone orientation (roll/pitch/yaw)
- Flight speed (X/Y/Z components)
- Gimbal positioning

**Camera Settings:**
- ISO, shutter speed, f-number
- Color temperature, digital zoom
- Frame dimensions and rate
- White balance and exposure

**Environmental Data:**
- Thermal imaging parameters
- Temperature and humidity
- Object distance and emissivity
- RTK GPS accuracy data

## Debugging Tips

### Common Issues

**Protocol Detection:**
- Unknown protocol warnings indicate new firmware
- Check %knownProtocol hash for supported protocols
- New protocols require sample submission

**Thermal Parameter Problems:**
- Verify magic number presence at expected offsets
- Different thermal cameras use different formats
- Parameter scaling varies between formats

**Protobuf Parsing:**
- Missing tags indicate default value of 0
- Field numbers are protocol-specific
- Format specifications must match data type

### Advanced Debugging

**XMP Issues:**
- DJI misspelled "GpsLongtitude" in original implementation
- Multiple coordinate tag variants for compatibility
- Avoid flag set to prevent coordinate conflicts

**Coordinate Conversion:**
- GPS units field determines radians vs degrees
- Automatic coordinate system conversion
- GPS member variables set for time correlation

**Beauty Settings:**
- Semicolon-separated format parsing
- Dynamic tag creation for unknown settings
- Integer/float value detection

**Thermal Debugging:**
- Multiple magic numbers for format detection
- Device header separate from temperature header
- Parameter scaling varies by camera model

The DJI module demonstrates sophisticated multi-format metadata handling with emphasis on professional drone cinematography, thermal imaging, and modern protobuf-based communication protocols.