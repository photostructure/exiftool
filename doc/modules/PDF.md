# PDF.pm Module Outline

**ExifTool Version:** 13.26  
**Module Version:** 1.61  
**Document Version:** 1.0  
**Last Updated:** 2025-06-29

## Overview

The PDF module handles Adobe Portable Document Format files, extracting metadata from document info dictionaries, XMP packets, and embedded resources. It supports encrypted PDFs and handles the complex object/stream structure of PDF files.

## Module Structure

### Tag Tables (32 total)

**Core Document Tables:**
- **%PDF::Main** - Root PDF structure (linearized flag)
- **%PDF::Info** - Document information dictionary
- **%PDF::Root** - Document catalog
- **%PDF::Pages** - Page tree structure
- **%PDF::Metadata** - XMP metadata stream

**Security/Forms Tables:**
- **%PDF::Encrypt** - Encryption dictionary
- **%PDF::Perms** - Document permissions
- **%PDF::AcroForm** - Interactive forms
- **%PDF::Signature** - Digital signatures

**Resource Tables:**
- **%PDF::Resources** - Page resources
- **%PDF::XObject** - External objects
- **%PDF::ColorSpace** - Color space definitions
- **%PDF::ImageResources** - Embedded images
- **%PDF::AF** - Associated files (PDF 2.0)

**Adobe-Specific Tables:**
- **%PDF::AdobePhotoshop** - Photoshop data
- **%PDF::Illustrator** - Illustrator data
- **%PDF::AIMetaData** - AI metadata
- **%PDF::AIPrivate** - AI private data

### Object/Stream Architecture

PDF uses indirect object references:
```
12 0 obj
<< /Type /Info /Title (Document) >>
endobj
```

**Object Types:**
- Dictionaries: `<< /Key /Value >>`
- Arrays: `[ item1 item2 ]`
- Strings: `(text)` or `<hex>`
- Names: `/Name`
- Numbers, Booleans, Null
- Streams: Dictionary + binary data

## Processing Flow

### 1. Main Processing
```perl
ProcessPDF($$$)
  ├── Validate PDF header
  ├── Read cross-reference table
  ├── Process trailer dictionary
  ├── Handle encryption
  └── Extract metadata objects
```

### 2. Object Resolution

**FetchObject($$$$)**
- Resolves indirect references (N 0 R)
- Handles object streams
- Manages decryption
- Caches fetched objects

**ReadPDFValue($)**
- Parses PDF syntax
- Handles all object types
- Processes escape sequences
- Manages nested structures

### 3. Special Processing

**ProcessDict($$$)**
- Recursively processes dictionaries
- Follows subdirectory links
- Handles special cases (Info, Root)
- Manages circular references

**DecodeStream($$$)**
- Handles multiple filters
- Supports compression (Flate, LZW)
- ASCII encoding (Hex, ASCII85)
- Manages encryption

## Key Features

### Encryption Support

**Encryption Types:**
- Standard security (RC4, AES)
- Password protection
- Permission restrictions
- String/stream encryption

**Decryption Process:**
- Extract encryption dictionary
- Calculate encryption key
- Decrypt strings/streams
- Handle different algorithms

### Filter Support

**Supported Filters:**
```perl
/FlateDecode    # ZIP compression
/LZWDecode      # LZW compression
/ASCIIHexDecode # Hex encoding
/ASCII85Decode  # Base85 encoding
/DCTDecode      # JPEG (pass-through)
/JPXDecode      # JPEG2000 (pass-through)
/Crypt          # Encryption
/Identity       # No filtering
```

### Metadata Extraction

**Multiple Sources:**
1. Info dictionary (traditional)
2. XMP metadata stream
3. Embedded resources
4. Form field data
5. Document properties

**User-Defined Tags:**
- Extracts unknown Info tags
- Preserves custom metadata
- Handles vendor extensions

### Cross-Reference Handling

**XRef Types:**
- Traditional table format
- Compressed XRef streams
- Hybrid XRef (both types)
- Incremental updates

## Unique Patterns

### Object Reference Pattern
```perl
# Indirect reference: "12 0 R"
if ($val =~ /^(\d+) (\d+) R$/s) {
    FetchObject($et, $1, $2, $xref);
}
```

### Dictionary Processing
```perl
# Nested dictionary with subdirectory
Info => {
    SubDirectory => { TagTable => 'Image::ExifTool::PDF::Info' },
    IgnoreDuplicates => 1,
},
```

### Stream Decoding
```perl
# Multiple filter chain
/Filter [ /ASCIIHexDecode /FlateDecode ]
```

## Special Considerations

### PDF Versions
- PDF 1.0-1.7 (ISO 32000-1)
- PDF 2.0 (ISO 32000-2)
- Version-specific features
- Backward compatibility

### Linearization
- Fast web view optimization
- Special object ordering
- Hint tables
- Progressive loading

### Incremental Updates
- Appended modifications
- Multiple versions in file
- Latest objects take precedence
- Preserves document history

## Write Support

**Writable Tags:**
- Info dictionary entries
- XMP metadata (via XMP.pm)
- Custom user tags
- Date/time values

**Write Limitations:**
- No structural changes
- Preserves encryption
- Incremental update only
- XMP synchronization issues

## Usage Notes

- Large files may be slow
- Encrypted PDFs need passwords
- Some filters not supported
- Circular references possible
- Memory usage for complex files

## Debugging Tips

- Use `-v3` for object details
- Check encryption with `-v2`
- XRef issues shown in warnings
- Filter chains in `-v`
- Object streams need `-v3`