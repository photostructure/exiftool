# How to Rebuild ExifTool Module Documentation

This guide helps future Claude Code sessions regenerate the documentation in this directory when ExifTool is updated.

## Overview

The documentation in `doc/` and `doc/modules/` provides high-level overviews of ExifTool's library modules to help Claude Code understand the codebase structure without reading entire module files. This is essential because:

- ExifTool modules are often thousands of lines long
- Claude's file reading has length limits (truncation after ~2000 lines)
- Understanding patterns is more important than memorizing every tag

## Prerequisites

Before starting, ensure you have:

- [ ] Access to the ExifTool source repository
- [ ] The current ExifTool version number (check `exiftool -ver` or the main ExifTool.pm file)
- [ ] A working Claude Code session with file creation capabilities

## Step-by-Step Instructions

### Step 1: Initial Setup

Create a todo list to track your progress:

```
Create a todo list with these tasks:
1. Create MODULE_OVERVIEW.md with categorized module listing
2. Create PATTERNS.md documenting cross-module patterns
3. Update CLAUDE.md to reference the new documentation
4. Create module outline for Canon.pm
5. Create module outline for Nikon.pm
6. Create module outline for Sony.pm
7. Create module outline for QuickTime.pm
8. Create module outline for XMP.pm
9. Create module outline for Exif.pm
10. Create module outline for PDF.pm
11. Create module outline for DICOM.pm
12. Create module outline for MXF.pm
13. Create module outline for Pentax.pm
14. Create module outline for Validate.pm
15. Create module outline for TagLookup.pm
16. Create module outline for BuildTagLookup.pm
17. Create module outline for Writer.pl
18. Create module outline for Validate.pm
19. Create module outline for Fixup.pm
20. Create module outline for Charset.pm
21. Create module outline for IPTC.pm
22. Create module outline for ICC_Profile.pm
23. Create module outline for MWG.pm
24. Create module outline for Olympus.pm
25. Create module outline for Minolta.pm
26. Create module outline for MinoltaRaw.pm
27. Create module outline for Panasonic.pm
28. Create module outline for PanasonicRaw.pm
29. Create module outline for Ricoh.pm
30. Create module outline for Samsung.pm
31. Create module outline for Sanyo.pm
32. Create module outline for Sigma.pm
33. Create module outline for SigmaRaw.pm
```

### Step 2: Analyze the Module Landscape

First, get an overview of all modules:

```bash
# List all Perl modules with sizes
wc -l lib/Image/ExifTool/*.pm | sort -n

# Count total modules
ls -1 lib/Image/ExifTool/*.pm | wc -l
```

### Step 3: Create MODULE_OVERVIEW.md

This file should contain:

1. **Module Categories:**

   - Camera/Manufacturer Modules (Canon, Nikon, Sony, etc.)
   - File Format Modules (JPEG, PNG, TIFF, etc.)
   - Metadata Standard Modules (XMP, IPTC, EXIF)
   - Utility/Infrastructure Modules
   - Language Modules (in Lang/)
   - Character Set Modules (in Charset/)

2. **For each category, include:**

   - Module name
   - File size (lines of code)
   - Complexity rating (Very High, High, Medium, Low)
   - Brief purpose description

3. **Version Header Template:**

   ```markdown
   # ExifTool Module Overview

   **ExifTool Version:** [current version]
   **Document Version:** 1.0
   **Last Updated:** [YYYY-MM-DD]
   ```

### Step 4: Create PATTERNS.md

Document recurring patterns across modules:

1. **Tag Table Structure Pattern**
2. **Module Organization Patterns**
3. **Processing Procedure Patterns**
4. **Common Tag Patterns**
5. **Value Conversion Patterns**
6. **Offset Handling Patterns**
7. **Writing Patterns**
8. **Error Handling Patterns**

Include code examples for each pattern.

### Step 5: Create Individual Module Outlines

For each major module, create a file in `doc/modules/[ModuleName].md` with this structure:

```markdown
# [ModuleName].pm Module Outline

**ExifTool Version:** [current]
**Module Version:** [from $VERSION in module]
**Document Version:** 1.0
**Last Updated:** [YYYY-MM-DD]

## Overview

[Brief description of what this module handles]

## Module Structure

### Tag Tables ([N] total)

[List major tag tables and their purposes]

### Key Data Structures

[Important hashes, arrays, or complex structures]

## Processing Architecture

### 1. Main Processing Flow

[Describe the main entry point and flow]

### 2. Special Processing Functions

[List and briefly describe unique functions]

## Key Features

[Module-specific features like encryption, lens databases, etc.]

## Special Patterns

[Module-specific patterns or unique approaches]

## Usage Notes

[Important information for users/developers]

## Debugging Tips

[Common issues and how to diagnose them]
```

### Step 6: Module Analysis Commands

Use these commands to analyze each module:

```bash
# Count tag tables
grep -c "^%Image::ExifTool::[ModuleName]::" lib/Image/ExifTool/[ModuleName].pm

# Get module version
grep '$VERSION = ' lib/Image/ExifTool/[ModuleName].pm

# Count tag definitions (hex format)
grep -c "^    0x" lib/Image/ExifTool/[ModuleName].pm

# Count tag definitions (string format)
grep -c "^    '[0-9a-f]" lib/Image/ExifTool/[ModuleName].pm

# Find encryption/special features
grep -i "encrypt\|decrypt\|crypt" lib/Image/ExifTool/[ModuleName].pm

# Find processing functions
grep "^sub Process" lib/Image/ExifTool/[ModuleName].pm
```

### Step 7: Module Priority Order

Analyze modules in this order (based on importance and complexity):

#### High Priority - Core Modules

1. **Exif.pm** - Foundation for all TIFF-based formats
2. **XMP.pm** - Complex RDF/XML parsing with namespaces
3. **QuickTime.pm** - Hierarchical container (MOV/MP4/HEIC)

#### High Priority - Major Camera Manufacturers

4. **Canon.pm** - Extensive maker notes, lens database
5. **Nikon.pm** - Complex encryption, largest module
6. **Sony.pm** - E-mount lenses, video metadata

#### Medium Priority - Specialized Formats

7. **PDF.pm** - Document structure with objects
8. **DICOM.pm** - Medical imaging standard
9. **MXF.pm** - Professional broadcast video
10. **Pentax.pm** - Comprehensive lens system

#### Additional Important Modules (if time permits)

- **Panasonic.pm** - Video-focused manufacturer
- **Olympus.pm** - Complex maker notes
- **PNG.pm** - Important image format
- **DNG.pm** - Adobe's raw format

### Step 8: Update CLAUDE.md

Add this section to the project's CLAUDE.md file:

```markdown
## Module Documentation

For detailed documentation on ExifTool's library modules:

- **`doc/concepts/MODULE_OVERVIEW.md`** - Overview of all modules, categorized by type
- **`doc/concepts/PATTERNS.md`** - Common patterns across all modules
- **`doc/modules/`** - Individual module outlines for major modules (when available)

When working with specific modules (especially Canon, Nikon, Sony, QuickTime, XMP), always check the module documentation first.
```

## Module-Specific Analysis Guidelines

### Camera Manufacturer Modules

Look for:

- Lens identification systems (lens type tables)
- Encryption/decryption methods
- Model-specific camera info tables
- Serial number handling
- AF system variations
- Custom function decoding

### Container Format Modules (QuickTime, MXF, RIFF)

Look for:

- Hierarchical structure parsing
- Atom/chunk/box handling
- Multiple stream support
- Codec information
- Metadata track handling

### Image Format Modules (JPEG, PNG, TIFF)

Look for:

- Chunk/segment structure
- Compression handling
- Color space information
- Embedded metadata support

### Metadata Standard Modules (XMP, IPTC, EXIF)

Look for:

- Tag organization structure
- Character encoding support
- Version differences
- Industry standard compliance

## Version Update Workflow

When ExifTool is updated:

1. **Check for Major Changes:**

   ```bash
   # Compare module line counts
   wc -l lib/Image/ExifTool/*.pm > new_counts.txt
   # Compare with previous counts to spot significant changes
   ```

2. **Update Version Headers:**

   - Update ExifTool Version in all .md files
   - Check module versions for changes
   - Note the update date

3. **Review Changed Modules:**

   - Check for new tag tables
   - Look for new processing functions
   - Note new features or patterns

4. **Update Documentation:**

   - Add new findings to relevant sections
   - Update tag counts if significantly changed
   - Increment Document Version to 1.1, 1.2, etc. for substantial updates

5. **Validate:**
   - Ensure all files are under 300 lines
   - Check that version headers are consistent
   - Verify CLAUDE.md references are correct

## Common Pitfalls to Avoid

1. **File Length**: Keep all .md files under 300 lines to prevent truncation
2. **Version Tracking**: Always update version headers when regenerating
3. **Tag Counting**: Different modules use different tag formats (hex vs. string)
4. **Module Names**: Some modules have irregular naming (e.g., "HP.pm" not "Hewlett-Packard.pm")
5. **Hidden Complexity**: Some small modules (like MakerNotes.pm) are crucial dispatchers

## Final Checklist

- [ ] All version headers updated with current ExifTool version
- [ ] MODULE_OVERVIEW.md covers all major module categories
- [ ] PATTERNS.md includes code examples
- [ ] Each module outline follows the standard template
- [ ] All files are under 300 lines
- [ ] CLAUDE.md updated with documentation references
- [ ] Todo list shows all tasks completed

## Notes for Future Claude Sessions

- This documentation is a map, not the territory - always verify against actual code
- Focus on understanding patterns rather than memorizing details
- When in doubt, check the module's $VERSION and compare with documented version
- The module landscape evolves - new modules may be added, old ones refactored
- Some modules may have moved between categories as features evolved

Remember: The goal is to help future Claude Code sessions quickly understand ExifTool's architecture without needing to read thousands of lines of code.
