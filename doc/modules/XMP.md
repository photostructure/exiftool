# XMP.pm Module Outline

**ExifTool Version:** 13.26  
**Module Version:** 3.66  
**Document Version:** 1.0  
**Last Updated:** 2025-06-29

## Overview

The XMP module implements Adobe's eXtensible Metadata Platform, providing comprehensive RDF/XML parsing, namespace management, and structure handling. It supports 72 namespace tables across XMP.pm and XMP2.pl, making it one of ExifTool's most extensible metadata systems.

## Module Structure

### Namespace Tables (72 total)

**Core Adobe Namespaces (25 in XMP.pm):**
- **xmp** - Basic XMP properties
- **dc** - Dublin Core
- **xmpMM** - Media Management
- **xmpRights** - Rights Management
- **photoshop** - Photoshop-specific
- **crs** - Camera Raw Settings
- **tiff/exif** - TIFF/EXIF mappings

**Extended Namespaces (47 in XMP2.pl):**
- **iptcCore/iptcExt** - IPTC standards
- **PLUS** - Picture Licensing Universal System
- **prism** - Publishing industry
- **GPano/GAudio/GImage** - Google formats
- **dwc** - Darwin Core (biodiversity)
- **drone-dji** - DJI drone metadata

### Structure System

**Core Structure Definitions:**
```perl
%sDimensions    # Width/height pairs
%sArea          # Rectangular regions
%sColorant      # Color specifications
%sResourceRef   # Resource references
%sContactInfo   # Contact information
%sLocation      # GPS coordinates
```

**Structure Features:**
- Nested structure support
- Automatic flattening with dot notation
- Type validation for fields
- JSON serialization for complex values

## Processing Architecture

### 1. Main Processing Flow
```perl
ProcessXMP($$$)
  ├── UTF encoding detection
  ├── BOM handling
  ├── Namespace extraction
  ├── ParseXMPElement() recursion
  └── Structure flattening
```

### 2. RDF/XML Parser

**ParseXMPElement($$$$$$$)**
- Recursive XML/RDF parser
- Handles elements, attributes, namespaces
- Processes RDF containers (Bag, Seq, Alt)
- Manages blank nodes and references

**FoundXMP($$$$$$)**
- Processes discovered properties
- Applies conversions
- Handles language alternatives
- Manages list items

### 3. Writing System

**WriteXMP($$$)**
- Rebuilds XMP from scratch
- Maintains RDF/XML compliance
- Preserves unknown namespaces
- Handles structure serialization

## Key Features

### Namespace Management

**Dynamic Namespace System:**
- 72 pre-defined namespaces
- Custom namespace support
- URI validation and normalization
- Prefix conflict resolution

**Namespace Categories:**
1. Adobe Standard (xmp, dc, etc.)
2. Industry Standards (IPTC, PLUS)
3. Vendor-Specific (Microsoft, Apple)
4. Domain-Specific (DICOM, drone-dji)

### Structure Flattening

**AddFlattenedTags($$$;$$)**
- Converts structures to flat tags
- Preserves namespace prefixes
- Handles nested structures
- Supports list of structures

**Flattening Example:**
```
ContactInfo => {
    CiAdrCity => "New York",
    CiAdrCtry => "USA"
}
Becomes:
ContactInfo.CiAdrCity = "New York"
ContactInfo.CiAdrCtry = "USA"
```

### Language Support

**Language Alternatives:**
- Full xml:lang support
- x-default handling
- Language code standardization
- GetLangInfo() for localization

**Example:**
```xml
<dc:title>
  <rdf:Alt>
    <rdf:li xml:lang="en">Title</rdf:li>
    <rdf:li xml:lang="fr">Titre</rdf:li>
  </rdf:Alt>
</dc:title>
```

### List Types

**RDF Container Support:**
- **Bag** - Unordered lists
- **Seq** - Ordered sequences
- **Alt** - Alternative values
- Lists of simple values or structures

## Special Patterns

### Namespace Declaration
```perl
%Image::ExifTool::XMP::dc = (
    NAMESPACE => 'dc',
    STRUCT_NAME => 'Dublin Core',
    NOTES => 'Dublin Core namespace tags.',
    # tag definitions...
);
```

### Structure Definition
```perl
%sDimensions = (
    STRUCT_NAME => 'Dimensions',
    NAMESPACE => 'xmpDM',
    w => { Writable => 'real' },
    h => { Writable => 'real' },
    unit => { },
);
```

### Conversion Functions
```perl
ValueConv => 'Image::ExifTool::XMP::DecodeBase64($val)',
PrintConv => 'Image::ExifTool::XMP::ConvertRational($val)',
```

## Advanced Features

### Validation System
- XMP specification compliance
- MWG compliance checking
- Namespace URI validation
- Structure field validation

### Encoding Support
- UTF-8, UTF-16, UTF-32
- BOM detection/handling
- XML entity encoding
- Proper escaping

### Extensibility
- Custom namespace addition
- User-defined structures
- Plugin architecture
- Backward compatibility

## Comparison with Other Metadata

| Feature | XMP | EXIF | IPTC |
|---------|-----|------|------|
| Structure | RDF/XML | Binary | Binary |
| Extensibility | Unlimited | Limited | Limited |
| Namespaces | Yes | No | No |
| Unicode | Native | Limited | Limited |
| Lists | Full | No | Limited |

## Usage Notes

- Most flexible metadata format
- Human-readable XML format
- Extensive Adobe support
- Industry-standard adoption
- Large but compressible

## Debugging Tips

- Use `-v3` to see namespace resolution
- Check `-validate` for XMP compliance
- Structure flattening visible in tag names
- Language codes shown with `-lang`
- Use `-struct` to preserve structures