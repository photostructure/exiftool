# MXF.pm Module Outline

**ExifTool Version:** 13.26  
**Module Version:** 1.08  
**Document Version:** 1.0  
**Last Updated:** 2025-06-29

## Overview

The MXF module handles Material Exchange Format files, a professional video container format used in broadcast and post-production. It extracts metadata from 1,652 standardized SMPTE tags using a complex Key-Length-Value (KLV) structure with 16-byte Universal Labels (UL).

## Module Structure

### Tag Organization

**Tag Count:**
- 1,652 tag definitions
- Based on SMPTE standards
- Hierarchical namespace structure
- 16-byte Universal Labels as IDs

**UL Format:**
```
06 0e 2b 34 . NN NN NN NN . NN NN NN NN . NN NN NN NN
^^ ^^ ^^ ^^   registry      category      subcategory
Object ID
```

### Key MXF Concepts

**KLV Structure:**
- **Key**: 16-byte Universal Label
- **Length**: BER-encoded length
- **Value**: Actual data payload

**Local Sets:**
- Use 2-byte tags mapped via Primer Pack
- More efficient than full ULs
- Dynamic tag assignment

## Processing Architecture

### 1. Main Processing Flow
```perl
ProcessMXF($$$)
  ├── Read Run-In (if present)
  ├── Process Header Partition
  ├── Read Primer Pack
  ├── Process Metadata Sets
  └── Handle Essence Containers
```

### 2. Partition Structure

**Partition Types:**
- Header Partition (metadata)
- Body Partition (essence)
- Footer Partition (index)
- Run-In Partition (legacy)

**Partition Components:**
- Partition Pack
- Header Metadata
- Index Tables
- Essence Containers

### 3. Metadata Processing

**ProcessLocalSet($$$)**
- Resolves local tags via Primer
- Handles Strong References
- Processes nested sets
- Manages instance relationships

**ProcessPrimer($$$)**
- Maps local tags to ULs
- Builds dynamic tag table
- Enables efficient encoding

## Key Features

### Reference System

**Reference Types:**
- **Strong Reference**: Direct object link
- **Weak Reference**: Optional link
- **UUID**: Unique identifier
- **UMID**: Unique Material ID

**Reference Arrays:**
- StrongReferenceArray
- StrongReferenceBatch
- Multiple object links

### Time Handling

**Temporal Metadata:**
- Timecode tracks
- Frame rates
- Edit rates
- Duration calculations

**Timestamp Format:**
- Based on Windows FILETIME
- 100-nanosecond intervals
- Since January 1, 1601

### Multi-Language Support

**Text Encoding:**
- UTF-16 for international text
- Language definitions
- Alternate language sets
- Serialization dependent

### Location Data

**GPS Support:**
```perl
Type => 'Lat'  # Latitude
Type => 'Lon'  # Longitude  
Type => 'Alt'  # Altitude
```
- Rational number format
- Coordinate conversion
- Standard GPS formatting

## Special Data Types

### Common Types
```perl
AUID         # 16-byte identifier
Boolean      # 1-byte true/false
GUID         # Windows GUID format
Label        # 16-byte UL
PackageID    # 32-byte UMID
Position     # Edit unit position
ProductVersion # Version structure
Timestamp    # 8-byte time
UID          # Instance identifier
UUID         # Standard UUID
'UTF-16'     # Unicode text
```

### Complex Types
- BatchOfUL: Array of Labels
- StrongReference: Object link
- WeakReference: Optional link
- VersionType: Major.Minor

## Metadata Categories

### Production Metadata
- Clip names and descriptions
- Creation dates/times
- Modification tracking
- Project information

### Technical Metadata
- Video parameters
- Audio specifications
- Codec information
- Frame rates

### Descriptive Metadata
- Rights information
- Content classification
- Personnel credits
- Location data

### Administrative
- Device information
- Organization IDs
- Archive references
- Workflow tracking

## Special Considerations

### Large Files
- 64-bit addressing
- Streaming capability
- Partial parsing support
- Memory efficiency

### Operational Patterns
- OP-Atom: Single item/file
- OP1a: Single item, multiple essence
- OP1b: Ganged packages
- OP2a-3c: Complex structures

### Essence Handling
- Video/Audio streams
- Ancillary data
- Multiple tracks
- Synchronization

## Professional Features

### Broadcast Standards
- SMPTE compliance
- EBU specifications
- AMWA application specs
- Interoperability focus

### Workflow Integration
- AAF compatibility
- Archive systems
- MAM integration
- Edit decision lists

## Usage Notes

- Professional broadcast format
- Complex hierarchical structure
- Large file sizes typical
- Reference-based metadata
- Industry-standard compliance

## Debugging Tips

- Use `-v3` for KLV details
- Check Primer Pack mappings
- Validate partition structure
- Monitor reference chains
- Verify essence alignment