# NONDETERMINISM.md

This document explains cases where ExifTool exhibits nondeterministic behavior - producing different outputs between runs despite identical inputs.

## Overview

ExifTool is generally deterministic, but certain composite tag calculations can produce different results between runs due to Perl's hash randomization feature. This primarily affects tags that involve pattern matching across multiple possible values where the selection order matters.

## Known Nondeterministic Behaviors

### 1. Nikon LensID Composite Tag

**Affected Tag:** `Composite:LensID`  
**Scope:** Nikon cameras with ambiguous lens identification  
**Manifestation:** Order of lens alternatives changes between runs  

**Example:**
```
Run 1: "AF-P DX Nikkor 18-55mm f/3.5-5.6G VR or AF-P DX Nikkor 18-55mm f/3.5-5.6G"
Run 2: "AF-P DX Nikkor 18-55mm f/3.5-5.6G or AF-P DX Nikkor 18-55mm f/3.5-5.6G VR"
```

**Root Cause:** In `/lib/Image/ExifTool/Nikon.pm` line 13453, the `LensIDConv` function uses:

```perl
my @ids = sort grep /^$pat$/, keys %$conv;
```

The `keys %$conv` operation iterates over the `%nikonLensIDs` hash in unpredictable order due to Perl's hash randomization (introduced in Perl 5.18 for security). When multiple lens variants match the same pattern after masking, the selection becomes nondeterministic.

**Affected Implementation Details:**

1. **Pattern Masking Logic:** The function masks parts of lens IDs to handle variations:
   - `s/^\w+ \w+/.. ../` - ignores LensIDNumber and LensFStops
   - `s/\w(\w)$/.$1/` - ignores bits 4-7 of LensType

2. **Ambiguous Lens Entries:** Multiple Nikon 18-55mm variants can match similar patterns:
   ```perl
   'A1 40 2D 53 2C 3C CB 86' => 'AF-P DX Nikkor 18-55mm f/3.5-5.6G',
   'A0 40 2D 53 2C 3C CA 8E' => 'AF-P DX Nikkor 18-55mm f/3.5-5.6G',
   'A0 40 2D 53 2C 3C CA 0E' => 'AF-P DX Nikkor 18-55mm f/3.5-5.6G VR',
   ```

3. **Selection Logic:** When multiple entries match, `$good[0]` selection varies:
   ```perl
   my @good = grep /^$pat$/, @ids;
   return $$conv{$good[0]} if @good;  # Nondeterministic selection
   ```

**When This Occurs:**
- Nikon cameras with newer AF-P DX lenses
- Situations where lens firmware variations create multiple possible matches
- Pattern matching cannot distinguish between legitimate lens variants

**Workaround for Consumers:**
Applications parsing ExifTool output should normalize "A or B" patterns by sorting the alternatives lexicographically to ensure consistent behavior.

## Technical Background

### Perl Hash Randomization

Starting with Perl 5.8.1, hash key iteration order is randomized for security reasons. This prevents certain denial-of-service attacks but introduces nondeterminism in code that depends on hash iteration order.

**Impact on ExifTool:**
- Most ExifTool operations are unaffected since they use direct hash lookups
- Issues arise only in code that iterates over hash keys for pattern matching
- The `sort` operation provides consistent ordering of matched results, but when multiple entries match subsequent filters, the selection becomes arbitrary

### Composite Tag Processing

Composite tags are calculated after all other metadata is extracted. The nondeterminism occurs during the value conversion phase when:

1. Multiple lens variants exist in the lookup table
2. Pattern matching reduces the candidates but still leaves multiple matches  
3. The first match is selected arbitrarily based on hash iteration order

## Scope Analysis

**Limited Impact:** This nondeterministic behavior appears to be unique to the Nikon lens identification system. Other modules using similar patterns have been analyzed:

- **Exif.pm:** Uses `keys %$conv` for building inverse conversion tables (deterministic context)
- **Olympus.pm:** Uses in OTHER functions but with different logic patterns
- **Canon.pm:** Uses direct lookups without pattern matching iterations
- **Other modules:** Generally use deterministic hash access patterns

**Future Considerations:**
- ExifTool's monthly release cycle may introduce new lens variants that trigger similar ambiguities
- Other manufacturers might adopt similar lens ID strategies that could exhibit this behavior
- Camera firmware updates sometimes change lens identification methods

## Best Practices for ExifTool Integration

### For Application Developers

1. **Normalize Composite Values:** When comparing ExifTool outputs, sort alternatives within "A or B" patterns
2. **Use Specific Group Prefixes:** Request `GPS:GPSLatitude` instead of `GPSLatitude` to avoid composite/raw ambiguity
3. **Version Consistency:** Use the same ExifTool version across environments to minimize variation
4. **Test with Multiple Runs:** Validate that your application handles nondeterministic composite values correctly

### For ExifTool Contributors

1. **Avoid Hash Iteration:** Use sorted keys when iteration order affects output
2. **Implement Tie-Breaking:** For ambiguous cases, use consistent criteria (alphabetical, chronological, etc.)
3. **Document Ambiguities:** Note when lens/camera identification cannot be fully deterministic
4. **Test Consistency:** Run test suites multiple times to catch nondeterministic behaviors

## See Also

- [COMPOSITE_TAGS.md](COMPOSITE_TAGS.md) - Complete composite tag system documentation
- [PATTERNS.md](PATTERNS.md) - Common ExifTool implementation patterns
- Perl documentation on hash randomization: `perldoc perlsec`

## References

- **Primary Source:** `/lib/Image/ExifTool/Nikon.pm` lines 13429-13469 (`LensIDConv` function)
- **Hash Definition:** `/lib/Image/ExifTool/Nikon.pm` lines 83-322 (`%nikonLensIDs`)
- **Composite Registration:** `/lib/Image/ExifTool/Nikon.pm` lines 13163-13189
- **Core Processing:** `/lib/Image/ExifTool.pm` lines 3552-3556 (OTHER function invocation)