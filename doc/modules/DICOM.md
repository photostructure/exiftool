# DICOM.pm Module Outline

**ExifTool Version:** 13.26  
**Module Version:** 1.23  
**Document Version:** 1.0  
**Last Updated:** 2025-06-29

## Overview

The DICOM module handles Digital Imaging and Communications in Medicine files, the standard format for medical imaging. It supports both DICOM and the older ACR-NEMA formats, extracting metadata from over 3,100 standardized tags plus vendor-specific private tags.

## Module Structure

### Tag Organization

**Tag Count:**
- 3,155 tag definitions in main table
- Covers DICOM 2009 specification
- Includes vendor-specific private tags
- Groups organized by function

**Tag ID Format:**
- Format: `(group,element)` in hex
- Example: `'0008,0060'` = Modality
- Groups define categories
- Elements specify attributes

### Value Representation (VR)

**VR Types:**
```perl
FD => 'double'      # Floating point double
FL => 'float'       # Floating point single
OB => 'int8u'       # Other byte string
OF => 'float'       # Other float string
OW => 'int16u'      # Other word string
SL => 'int32s'      # Signed long
SS => 'int16s'      # Signed short
UL => 'int32u'      # Unsigned long
US => 'int16u'      # Unsigned short
# Plus text types: AE, AS, CS, DA, DS, DT, IS, LO, LT, PN, SH, ST, TM, UI, UT
```

**VR Categories:**
- Numeric types (FD, FL, SL, SS, UL, US)
- Text types (various string formats)
- Binary types (OB, OW, OF)
- Special types (SQ for sequences)

## Processing Architecture

### 1. Format Detection
```perl
ProcessDICOM($$$)
  ├── Check for DICM signature
  ├── Determine ACR-NEMA vs DICOM
  ├── Detect transfer syntax
  └── Process data elements
```

### 2. Transfer Syntax

**Supported Syntaxes:**
- Implicit VR Little Endian
- Explicit VR Little Endian
- Explicit VR Big Endian
- Deflated (compressed)
- Various JPEG syntaxes

**Byte Order:**
- Default: Little-endian
- Big-endian for specific syntaxes
- Dynamic detection and switching

### 3. Data Element Processing

**Element Structure:**
```
[Tag(4)] [VR(2)] [Length(2/4)] [Value(Length)]
```
- Tag: Group and element numbers
- VR: Value representation (explicit)
- Length: 16 or 32-bit
- Value: Actual data

## Key Features

### Group Organization

**Major Groups:**
- `0002` - File Meta Information
- `0004` - Directory Structure
- `0008` - Identifying Information
- `0010` - Patient Information
- `0018` - Acquisition Information
- `0020` - Relationship Information
- `0028` - Image Presentation
- `0040` - Procedure Information
- `0070` - Graphic Annotation
- `7FE0` - Pixel Data

### Special Elements

**Implicit VR Elements:**
```perl
'FFFE,E000' => Item
'FFFE,E00D' => Item Delimitation
'FFFE,E0DD' => Sequence Delimitation
```

**Length Encoding:**
- 32-bit for: OB, OW, OF, SQ, UT, UN
- 16-bit for all others
- Undefined length: 0xFFFFFFFF

### Private Tags

**Vendor Extensions:**
- GE Medical Systems
- Siemens
- Philips
- Toshiba
- Agfa
- Other manufacturers

**Private Tag Format:**
- Odd group numbers
- Vendor-specific meanings
- Often proprietary formats

### UID Handling

**UID Types:**
- SOP Class UIDs
- Transfer Syntax UIDs
- Implementation UIDs
- Instance UIDs

**UID Database:**
- Built-in UID definitions
- Human-readable names
- Classification support

## Special Processing

### Sequence Handling

**Nested Structures:**
- Sequences contain items
- Items contain elements
- Recursive processing
- Proper delimitation

### Multi-Frame Support

**Frame Organization:**
- Per-frame metadata
- Shared metadata
- Frame extraction
- Time sequences

### Character Sets

**Encoding Support:**
- ISO_IR character sets
- Unicode support
- Multi-byte encodings
- Proper conversion

### Pixel Data

**Image Formats:**
- Uncompressed
- JPEG (various)
- JPEG 2000
- RLE
- Deflated

## Medical Context

### Patient Information
- Patient name, ID, birth date
- Sex, age, weight
- Medical record number
- Patient comments

### Study Information
- Study date/time
- Study description
- Accession number
- Referring physician

### Series Information
- Series number
- Series description
- Modality (CT, MR, US, etc.)
- Body part examined

### Image Information
- Image number
- Image type
- Pixel spacing
- Window center/width
- Image position/orientation

## Special Considerations

### Privacy (PHI)
- Patient identifiable information
- DICOM anonymization
- Safe tag lists
- Privacy profiles

### Large Files
- Multi-gigabyte images
- Streaming processing
- Memory efficiency
- Pixel data handling

### Compliance
- DICOM standard conformance
- IHE profile support
- Validation capabilities
- Error handling

## Usage Notes

- Medical imaging standard
- Complex nested structures
- Privacy considerations important
- Large file sizes common
- Vendor variations exist

## Debugging Tips

- Use `-v3` for element details
- Check transfer syntax early
- Validate VR consistency
- Monitor sequence nesting
- Verify character encoding