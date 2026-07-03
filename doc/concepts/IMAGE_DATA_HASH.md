# ExifTool ImageDataHash Implementation

**ExifTool Version:** 13.26
**Document Version:** 1.0

## Overview

ImageDataHash computes a cryptographic digest of an image's actual pixel/media data, excluding metadata. This allows detecting changes to the image content itself while ignoring metadata modifications. The hash is **only generated when explicitly requested** via the `-ImageDataHash` tag.

## Supported Formats

| Format Type | Formats | Hash Coverage |
|-------------|---------|---------------|
| Still Images | JPEG, TIFF, PNG, JP2, JXL, HEIC, AVIF | Main image data |
| RAW Formats | CRW, CR3, MRW, RAF, X3F, IIQ | Raw sensor data |
| Video/Audio | MOV, MP4, AVI, WAV, WEBP | Video + audio streams |

## Hash Algorithm Selection

Configured via the `ImageHashType` API option:

| Algorithm | Default | Hash Length | Empty Hash (skipped) |
|-----------|---------|-------------|---------------------|
| MD5 | Yes | 32 hex chars | `d41d8cd98f00b204e9800998ecf8427e` |
| SHA256 | No | 64 hex chars | `e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855` |
| SHA512 | No | 128 hex chars | `cf83e1357...` (full hash omitted) |

Empty hashes are detected and suppressed—no ImageDataHash tag is produced if no image data was found.

## Implementation Architecture

### 1. Hash Object Lifecycle

**Creation** (`lib/Image/ExifTool.pm` lines 2766-2780):
```perl
# In ImageInfo(), when ImageDataHash is requested:
if ($$req{imagedatahash} and not $$self{ImageDataHash}) {
    my $imageHashType = $self->Options('ImageHashType');
    if ($imageHashType =~ /^SHA(256|512)$/i) {
        $$self{ImageDataHash} = Digest::SHA->new($1);
    } elsif (require Digest::MD5) {
        $$self{ImageDataHash} = Digest::MD5->new;
    }
}
```

**Finalization** (`lib/Image/ExifTool.pm` lines 4378-4386):
```perl
# In DoneExtract():
if ($$self{ImageDataHash}) {
    my $digest = $$self{ImageDataHash}->hexdigest;
    # Skip empty digests (no image data found)
    $self->FoundTag(ImageDataHash => $digest) unless $digest eq $emptyMD5 or ...;
}
```

### 2. Core Hash Function

**`ImageDataHash($$$;$$)`** in `lib/Image/ExifTool/Writer.pl` lines 7086-7109:

```perl
sub ImageDataHash($$$;$$) {
    my ($self, $raf, $size, $type, $noMsg) = @_;
    my $hash = $$self{ImageDataHash} or return;
    my ($bytesRead, $n) = (0, 65536);
    my $buff;
    for (;;) {
        if (defined $size) {
            last unless $size;
            $n = $size > 65536 ? 65536 : $size;
            $size -= $n;
        }
        unless ($raf->Read($buff, $n)) {
            $self->Warn("Error reading $type data") if $type and defined $size;
            last;
        }
        $hash->add($buff);
        $bytesRead += length $buff;
    }
    # Verbose output
    if ($$self{OPTIONS}{Verbose} and $bytesRead and $type and not $noMsg) {
        $self->VPrint(0, "$$self{INDENT}(ImageDataHash: $bytesRead bytes of $type data)\n");
    }
    return $bytesRead;
}
```

Key characteristics:
- Reads data in 64KB chunks to handle large files
- If `$size` is undefined, reads until EOF
- Returns total bytes read for verbose reporting
- Does not seek—caller must position RAF first

### 3. TIFF/EXIF Tag Identification

Tags with `IsImageData => 1` flag identify image data for hashing.

**`AddImageDataHash($$$)`** in `lib/Image/ExifTool/Exif.pm` (via WriteExif.pl lines 425-462):

```perl
sub AddImageDataHash($$$) {
    my ($et, $dirInfo, $offsetInfo) = @_;
    foreach $tagID (sort keys %$offsetInfo) {
        next unless ref $$offsetInfo{$tagID} eq 'ARRAY';
        my $tagInfo = $$offsetInfo{$tagID}[0];
        next unless $$tagInfo{IsImageData};     # Only image data tags
        my $sizeID = $$tagInfo{OffsetPair};     # Get corresponding size tag
        my @sizes = split ' ', $$offsetInfo{$sizeID}[1];
        my @offsets = split ' ', $$offsetInfo{$tagID}[1];
        foreach $offset (@offsets) {
            my $size = shift @sizes;
            $raf->Seek($offset, 0);             # Offsets are absolute
            $total += $et->ImageDataHash($raf, $size);
        }
    }
}
```

## Format-Specific Implementations

### JPEG (`lib/Image/ExifTool.pm` lines 7217-7406)

Hashes the scan data (SOS segment through EOI):

```perl
# In ProcessJPEG:
if ($hash and defined $marker and ($marker == 0x00 or $marker == 0xda or
    ($marker >= 0xd0 and $marker <= 0xd7)))  # SOS, stuffed bytes, RST markers
{
    $hash->add("\xff" . chr($marker));
    $hash->add($buff);
    $hashsize += $skipped + 2;
}
```

**What's included:**
- SOS (Start of Scan) marker and data
- RST (Restart) markers 0xd0-0xd7
- Stuffed null bytes (0xff00)
- Image entropy-coded data

**What's excluded:**
- All APP segments (APP0-APP15)
- COM (Comment) segments
- DQT, DHT, SOF segments (these define the codec, not the image content)

### PNG (`lib/Image/ExifTool/PNG.pm` lines 1419-1593)

Hashes IDAT (image data) chunks:

```perl
# Data chunks (IDAT for PNG, JDAT for JNG, etc.)
if ($isDataChunk{$chunk}) {
    if ($hash) {
        $et->ImageDataHash($raf, $len);
        $raf->Read($cbuf, 4);  # Read CRC separately
    }
}
```

**What's included:**
- IDAT chunks (PNG image data)
- JDAT chunks (JNG JPEG data)
- fdAT chunks (APNG frame data)

**What's excluded:**
- All text chunks (tEXt, iTXt, zTXt)
- Ancillary chunks (tIME, gAMA, cHRM, etc.)
- IHDR header

### TIFF/RAW (`lib/Image/ExifTool/Exif.pm` lines 6200-7094)

Uses `IsImageData` flag on offset/size tag pairs:

```perl
# Example from Exif.pm tag definitions:
0x111 => {
    Name => 'StripOffsets',
    IsOffset => 1,
    IsImageData => 1,        # <-- Marks for hashing
    OffsetPair => 0x117,     # Points to StripByteCounts
},
```

**Tags with `IsImageData => 1`:**
- `StripOffsets` (0x111) + `StripByteCounts` (0x117)
- `TileOffsets` (0x144) + `TileByteCounts` (0x145)
- `JpgFromRawStart` (0x201) + `JpgFromRawLength` (0x202)
- `ImageOffset` (0xbcc0) + `ImageByteCount` (0xbcc1) [PhaseOne]
- Various OtherImage/PreviewJXL offsets in DNG

**What's excluded:**
- ThumbnailImage (IFD1 thumbnail)
- PreviewImage
- Maker note embedded images

### QuickTime/MOV/MP4 (`lib/Image/ExifTool/QuickTime.pm`, `QuickTimeStream.pl`)

Processes video and audio sample data:

```perl
# In QuickTime.pm line 514:
my %hashBox = ( vide => { %eeStd }, soun => { %eeStd } );
# Requires: stco/co64 (chunk offsets), stsz (sample sizes), stsc (sample-to-chunk)
```

**Sample processing** (`QuickTimeStream.pl` lines 1284-1570):
```perl
# For each sample:
$raf->Seek($$start[$i], 0);
$raf->Read($buff, $$size[$i]);
$hash->add($buff);
$hashSize += $$size[$i];
```

**What's included:**
- `vide` (video) track samples
- `soun` (audio) track samples

**What's excluded:**
- `meta` (metadata) tracks
- `text` (subtitle) tracks
- Atoms outside mdat

### HEIC/AVIF (`lib/Image/ExifTool/QuickTime.pm` lines 9367-9374)

Hashes image items by codec type:

```perl
my %isImageData = ( av01 => 1, avc1 => 1, hvc1 => 1, lhv1 => 1, hvt1 => 1 );

if ($isImageData{$type} and $$et{ImageDataHash}) {
    foreach $extent (@{$$item{Extents}}) {
        $raf->Seek($$extent[1] + $base, 0);
        $tot += $et->ImageDataHash($raf, $$extent[2], "$type image", 1);
    }
}
```

### JPEG 2000 / JXL (`lib/Image/ExifTool/Jpeg2000.pm`)

Hashes codestream boxes:

```perl
my %isImageData = ( jp2c=>1, jbrd=>1, jxlp=>1, jxlc=>1 );

if ($hash and $isImageData{$boxID}) {
    $et->ImageDataHash($raf, $boxLen, $boxID);
}
```

### RIFF (AVI/WAV/WEBP) (`lib/Image/ExifTool/RIFF.pm`)

Hashes media data chunks:

```perl
my %isImageData = (
    LIST_movi => 1,  # AVI: contains ##db, ##dc, ##wb
    data => 1,       # WAV audio data
    'VP8 '=>1, VP8L=>1, ANIM=>1, ANMF=>1, ALPH=>1,  # WebP
);

if ($hash and $isImageData{$tag}) {
    $et->ImageDataHash($raf, $len2, "'${tag}' chunk");
}
```

### Canon CRW (`lib/Image/ExifTool/CanonRaw.pm` lines 702-703)

Hashes raw sensor data (tag 0x2005):

```perl
if ($$et{ImageDataHash} and $tagID == 0x2005) {
    $raf->Seek($ptr, 0) and $et->ImageDataHash($raf, $size, 'raw');
}
```

### Fuji RAF (`lib/Image/ExifTool/FujiFilm.pm` line 1979)

Hashes raw data if FujiIFD parsing fails:

```perl
unless ($et->ProcessTIFF(\%dirInfo, $tagTablePtr, \&ProcessTIFF)) {
    $et->ImageDataHash($raf, $len, 'raw') if $$et{ImageDataHash};
}
```

### Sigma X3F (`lib/Image/ExifTool/SigmaRaw.pm` lines 553-554)

Hashes non-preview IMAG sections:

```perl
if ($$et{ImageDataHash} and substr($buff,8,1) ne "\x02") {
    $et->ImageDataHash($raf, $len, 'SigmaRaw IMAG');
}
```

### Minolta MRW (`lib/Image/ExifTool/MinoltaRaw.pm` line 492)

Hashes raw data for non-A100:

```perl
$et->ImageDataHash($raf, undef, 'raw') unless $$et{A100DataOffset};
```

Sony A100 uses different handling via `A100DataOffset`.

### PhaseOne IIQ (`lib/Image/ExifTool/PhaseOne.pm` lines 682-691)

Hashes tags with `IsImageData`:

```perl
if ($hash and $tagInfo and $$tagInfo{IsImageData}) {
    while ($len) {
        my $n = $len > 65536 ? 65536 : $len;
        my $tmp = substr($$dataPt, $pos, $n);
        $hash->add($tmp);
        $len -= $n;
        $pos += $n;
    }
}
```

## XMP Storage Tags

ExifTool provides XMP tags to persist hash values in files:

| Tag | Purpose |
|-----|---------|
| `XMP-et:OriginalImageHash` | Stores computed ImageDataHash |
| `XMP-et:OriginalImageHashType` | Records algorithm used (MD5/SHA256/SHA512) |

Defined in `lib/Image/ExifTool/XMP.pm` lines 2701-2708.

## Tribal Knowledge

1. **Hash object is reused**: Once created in `ImageInfo()`, the same Digest object accumulates all image data across format-specific handlers.

2. **Verbose output is per-chunk**: Each format handler prints its own "(ImageDataHash: N bytes of X data)" message when verbose mode is enabled.

3. **Empty hash suppression**: The three known empty hashes (MD5, SHA256, SHA512 of zero bytes) are detected in `DoneExtract()` to avoid storing meaningless values.

4. **JpgFromRaw vs ThumbnailImage**: JpgFromRaw is included in the hash (it's the full embedded JPEG), but ThumbnailImage/PreviewImage are intentionally excluded.

5. **A100 special case**: Sony A100 stores raw data at a specific offset (`A100DataOffset`) that requires special handling in both MRW and TIFF processing paths.

6. **Panasonic NotRealPair hack**: Some Panasonic cameras have offset tags without proper size pairs; these use `NotRealPair` flag and assume data runs to EOF (`@sizes = 999999999`).

## Files Referenced

- `lib/Image/ExifTool.pm` (lines 2014-2026, 2766-2780, 4378-4386, 7217-7406)
- `lib/Image/ExifTool/Writer.pl` (lines 7086-7109)
- `lib/Image/ExifTool/Exif.pm` (lines 69, 582-617, 6200-6206, 7088-7094)
- `lib/Image/ExifTool/WriteExif.pl` (lines 425-462)
- `lib/Image/ExifTool/PNG.pm` (lines 1419-1593)
- `lib/Image/ExifTool/QuickTime.pm` (lines 509-537, 9367-9374, 9965-10067)
- `lib/Image/ExifTool/QuickTimeStream.pl` (lines 1284-1570)
- `lib/Image/ExifTool/RIFF.pm` (lines 41-46, 2034, 2151-2186)
- `lib/Image/ExifTool/Jpeg2000.pm` (lines 38, 1061, 1131-1166, 1629-1631)
- `lib/Image/ExifTool/CanonRaw.pm` (lines 702-704)
- `lib/Image/ExifTool/FujiFilm.pm` (line 1979)
- `lib/Image/ExifTool/SigmaRaw.pm` (lines 553-555)
- `lib/Image/ExifTool/MinoltaRaw.pm` (line 492)
- `lib/Image/ExifTool/PhaseOne.pm` (lines 589, 682-691)
- `lib/Image/ExifTool/XMP.pm` (lines 2701-2708)
