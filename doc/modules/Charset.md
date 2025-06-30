# Charset.pm Module Outline

**ExifTool Version:** 13.26
**Module Version:** 1.11
**Document Version:** 1.0
**Last Updated:** 2025-06-30

## Overview

The Charset module serves as ExifTool's character encoding and conversion engine. It provides character set translation between various encodings and Unicode, encoding detection with byte order handling, support for 1-byte, 2-byte, 4-byte, and variable-width character encodings, Unicode processing including UTF-8/UTF-16/UCS with surrogate pairs, and support for numerous legacy and platform-specific character sets.

The module supports 42 different character sets including Unicode/UTF variants, Western/Eastern European encodings, Middle Eastern scripts, Asian character sets, and Mac platform-specific encodings.

## Module Structure

### Core Data Structures

- **%charsetTable** - Cache for loaded character set lookup tables
- **%unicode2byte** - Reverse lookup tables for Unicode to byte conversion
- **%csType** - Bit flags defining characteristics of each supported character set

### Character Set Categories (42 total)

1. **Unicode/UTF** - UTF8, UTF16, UCS2, UCS4, ASCII
2. **Western** - Latin, Latin2, DOSLatinUS, DOSLatin1, PDFDoc
3. **Eastern European** - Baltic, Cyrillic, Greek, Turkish, DOSCyrillic
4. **Middle Eastern** - Arabic, Hebrew, MacArabic, MacHebrew
5. **Asian** - JIS, ShiftJIS, MacJapanese, MacChineseCN/TW, MacKorean, Thai, Vietnam
6. **Mac Platform** - MacRoman, MacCroatian, MacCyrillic, etc. (10 variants)

## Processing Architecture

### 1. Main Processing Flow

Character conversion uses a two-stage process: Decompose (encoded string → Unicode codepoints) and Recompose (Unicode codepoints → target encoding). The module includes automatic byte order detection using statistical analysis and error counting.

### 2. Special Processing Functions

- **LoadCharset($charset)** - Dynamically loads character set translation modules
- **IsUTF16($arrayRef)** - Validates UTF-16 character arrays and detects surrogate pairs
- **Decompose($et, $string, $charset, $byteOrder)** - Converts encoded strings to Unicode arrays
- **Recompose($et, $unicodeArray, $charset, $byteOrder)** - Converts Unicode back to encoded strings

## Key Features

- **Byte Order Detection Algorithm** - Statistical analysis counting unique values and zero frequency
- **Variable-Width Character Handling** - Nested hash structures for complex encodings like ShiftJIS
- **Performance Optimizations** - Pre-loaded tables, lazy loading, caching, Perl version handling
- **UTF-16 Surrogate Processing** - Proper handling of Unicode code points above U+FFFF

## Special Patterns

- **Statistical Byte Order Detection** - Analyzes high/low byte patterns and encoding errors
- **Lazy Loading Architecture** - Character set modules loaded only when needed
- **Caching Strategy** - Both forward and inverse lookup tables cached for performance
- **Error Recovery** - Inserts replacement characters for invalid sequences

## Usage Notes

- Use simpler character sets when possible for better performance
- Specify byte order when known to avoid detection overhead
- Public interface only supports type 0x101 and lower character sets
- UTF-8 warning behavior controlled by `$$et{WarnBadUTF8}` flag
- Batch processing multiple strings with same charset is more efficient

## Debugging Tips

- Check `$$et{WrongByteOrder}` flag for byte order detection results
- Monitor `$$et{EncodingError}` flag for conversion issues
- Use ExifTool `-v` option to show character conversion steps
- Arabic and Hebrew directional characters not fully supported
- No provision for specifying byte order in public interface for input/output values