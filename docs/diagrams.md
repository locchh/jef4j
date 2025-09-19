# Diagrams

## End‑to‑end flow: EBCDIC Fujitsu bytes → UTF‑8 bytes

**Components involved**

`net/arnx/jef4j/Jef4jCharsetProvider`: SPI provider that creates custom Charset instances by name.

`net/arnx/jef4j/FujitsuCharset`: A `java.nio.charset.Charset` wrapper that returns the matching encoder/decoder.

`net/arnx/jef4j/FujitsuCharsetType`: Enum defining Fujitsu flavors like EBCDIC, JEF_EBCDIC, JEF_HD_EBCDIC, etc., with flags

- `handleShift()`: toggles 0x28/0x38 shift-in and 0x29 shift-out handling.
- `handleJEF()`: toggles JEF double‑byte logic.
- `handleIVS()`: allows IVS (variation selectors) logic during encode/decode.
- `getJEFTableNo()`: selects a JEF table variant.

`net/arnx/jef4j/FujitsuCharsetDecoder`: Decodes EBCDIC/JEF bytes to Unicode chars. Loads mapping tables from `FujitsuDecodeMap.dat` and handle:
- Single-byte maps EBCDIC_MAP/EBCDIK_MAP (and ASCII variant)
- JEF double-byte mapping via JEF_MAP when `handleJEF()`
- Optional shift-in/out handling when `handleShift()`
- Optional IVS handling when `handleIVS()`

`net/arnx/jef4j/FujitsuCharsetEncoder`: The reverse direction (Unicode to Fujitsu bytes). Not used for UTF‑8 output, but helpful to understand symmetry.

Utility structures: `LongObjMap`, `Record` and its impls ( `ByteRecord`, `CharRecord`, `IntRecord`, `LongRecord`) hold compact mapping rows used by JEF lookup.

**Sequence diagram**

```mermaid
sequenceDiagram
    participant App
    participant Provider as Jef4jCharsetProvider
    participant CS as FujitsuCharset
    participant Dec as FujitsuCharsetDecoder
    participant UTF8 as Standard UTF-8 Encoder

    App->>Provider: Charset.forName("x-Fujitsu-JEF-EBCDIC")
    Provider-->>App: FujitsuCharset(type=JEF_EBCDIC)

    App->>CS: newDecoder()
    CS-->>App: FujitsuCharsetDecoder(type=JEF_EBCDIC)

    Note over Dec: Load maps from FujitsuDecodeMap.dat

    App->>Dec: decode(ByteBuffer EBCDIC)
    Dec-->>App: CharBuffer Unicode

    App->>UTF8: encode(CharBuffer Unicode)
    UTF8-->>App: ByteBuffer UTF-8
```

1. Resolve the charset:

2. Decode bytes to Unicode:

3. Encode Unicode to UTF‑8:

**Example**

```java
import java.io.*;
import java.nio.*;
import java.nio.charset.*;
import java.nio.file.*;
import java.util.*;

public class ConvertFujitsuEBCDICToUTF8 {
    public static void main(String[] args) throws Exception {
        Path inputPath = Paths.get("input.ebc");   // your EBCDIC Fujitsu file
        Path outputPath = Paths.get("output.txt"); // UTF-8 destination

        // Choose the correct Fujitsu charset name for your data:
        //   "x-Fujitsu-EBCDIC"       // single-byte EBCDIC
        //   "x-Fujitsu-JEF-EBCDIC"   // JEF double-byte with EBCDIC single-byte plane
        //   "x-Fujitsu-JEF-HanyoDenshi-EBCDIC" // with IVS/HD handling
        Charset fujitsu = Charset.forName("x-Fujitsu-JEF-EBCDIC");

        byte[] inBytes = Files.readAllBytes(inputPath);

        // Step 1: Decode Fujitsu EBCDIC/JEF -> Unicode
        CharsetDecoder decoder = fujitsu.newDecoder();
        // Optional error handling policy:
        decoder.onMalformedInput(CodingErrorAction.REPORT);
        decoder.onUnmappableCharacter(CodingErrorAction.REPORT);

        CharBuffer chars = decoder.decode(ByteBuffer.wrap(inBytes));

        // Step 2: Encode Unicode -> UTF-8
        byte[] outBytes = new String(chars.array(), 0, chars.length()).getBytes(StandardCharsets.UTF_8);
        Files.write(outputPath, outBytes);
    }
}
```

## Class relationships

```mermaid
classDiagram
    direction LR

    class Charset
    class CharsetProvider
    class CharsetEncoder
    class CharsetDecoder

    class Jef4jCharsetProvider {
      +Jef4jCharsetProvider()
      +Iterator<Charset> charsets()
      +Charset charsetForName(String)
      -ConcurrentMap<String, Charset> map
    }

    class FujitsuCharsetType {
      <<enum>>
      +EBCDIC
      +EBCDIK
      +ASCII
      +JEF
      +JEF_EBCDIC
      +JEF_EBCDIK
      +JEF_ASCII
      +JEF_HD
      +JEF_HD_EBCDIC
      +JEF_HD_EBCDIK
      +JEF_HD_ASCII
      +JEF_AJ1
      +JEF_AJ1_EBCDIC
      +JEF_AJ1_EBCDIK
      +JEF_AJ1_ASCII
      +JEF_R
      --
      -String charsetName
      -boolean handleShift()
      -boolean handleIVS()
      -int getJEFTableNo()
    }

    class FujitsuCharset {
      -FujitsuCharsetType type
      +FujitsuCharset(FujitsuCharsetType)
      +boolean contains(Charset)
      +CharsetDecoder newDecoder()
      +CharsetEncoder newEncoder()
    }

    class FujitsuCharsetEncoder {
      -FujitsuCharsetType type
      -byte[] map
      -boolean kshifted
      -StringBuilder backup
      +FujitsuCharsetEncoder(Charset, FujitsuCharsetType)
      +CoderResult encodeLoop(CharBuffer, ByteBuffer)
      +CoderResult implFlush(ByteBuffer)
      +void implReset()
      -boolean isEndOfInput()
      -- static maps --
      -static byte[] ASCII_MAP
      -static byte[] EBCDIC_MAP
      -static byte[] EBCDIK_MAP
      -static LongObjMap<Record[]> JEF_MAP
    }

    class FujitsuCharsetDecoder {
      -FujitsuCharsetType type
      -byte[] map
      -boolean kshifted
      +FujitsuCharsetDecoder(Charset, FujitsuCharsetType)
      +CoderResult decodeLoop(ByteBuffer, CharBuffer)
      +void implReset()
      -- static maps --
      -static byte[] ASCII_MAP
      -static byte[] EBCDIC_MAP
      -static byte[] EBCDIK_MAP
      -static LongObjMap<Record[]> JEF_MAP
    }

    class LongObjMap~T~ {
      +LongObjMap()
      +LongObjMap(int)
      +LongObjMap(int,float)
      +T put(long, T)
      +T get(long)
      +int size()
      -rehash(int)
    }

    class Record {
      <<interface>>
      +boolean exists(int pos)
      +long get(int pos)
    }

    class ByteRecord {
      +ByteRecord()
      +ByteRecord(char, byte[])
      +void set(char, byte[])
      +boolean exists(int)
      +long get(int)
      +String toString()
      -char pattern
      -byte[] array
    }

    class CharRecord {
      +CharRecord()
      +CharRecord(char, char[])
      +void set(char, char[])
      +boolean exists(int)
      +long get(int)
      +String toString()
      -char pattern
      -char[] array
    }

    class IntRecord {
      +IntRecord()
      +IntRecord(char, int[])
      +void set(char, int[])
      +boolean exists(int)
      +long get(int)
      +String toString()
      -char pattern
      -int[] array
    }

    class LongRecord {
      +LongRecord()
      +LongRecord(char, long[])
      +void set(char, long[])
      +boolean exists(int)
      +long get(int)
      +String toString()
      -char pattern
      -long[] array
    }

    class ByteUtils {
      <<utility>>
      +static String hex(long,int)
      +static String hex(int,int)
      +static String hex(CharBuffer)
      +static String hex(ByteBuffer)
      +static String hex(byte[])
    }

    CharsetProvider <|-- Jef4jCharsetProvider
    Charset <|-- FujitsuCharset
    CharsetEncoder <|-- FujitsuCharsetEncoder
    CharsetDecoder <|-- FujitsuCharsetDecoder

    Jef4jCharsetProvider --> FujitsuCharset : creates per name
    FujitsuCharset --> FujitsuCharsetType : holds type
    FujitsuCharset --> FujitsuCharsetEncoder : newEncoder()
    FujitsuCharset --> FujitsuCharsetDecoder : newDecoder()

    FujitsuCharsetEncoder --> LongObjMap~Record[]~ : JEF_MAP lookup
    FujitsuCharsetDecoder --> LongObjMap~Record[]~ : JEF_MAP lookup
    LongObjMap~Record[]~ --> Record
    ByteRecord ..|> Record
    CharRecord ..|> Record
    IntRecord ..|> Record
    LongRecord ..|> Record

    FujitsuCharsetEncoder --> ByteUtils : formatting (toString helpers)
    FujitsuCharsetDecoder --> ByteUtils : formatting (via Records)
```

## Encoding workflow (FujitsuCharsetEncoder.encodeLoop())

```mermaid
flowchart TD
    A[Start encodeLoop: CharBuffer in; ByteBuffer out] --> B{backup exists}
    B -- yes --> B1[Ensure output has 2 bytes]
    B1 -- no --> BO[OVERFLOW]
    B1 -- yes --> B2[Read next chars into backup\nHandle surrogate pairing]
    B2 --> B3[Re-run encodeLoop with backup restored]
    B -- no --> D{Input has more chars}
    D -- no --> Z[UNDERFLOW]

    D -- yes --> E[Read next char c]
    E --> F{c is U+FFFE or higher}
    F -- yes --> F1[UNMAPPABLE 1 char]
    F -- no --> G{Single-byte path}
    G -- yes --> H{map present}
    H -- no --> H1[UNMAPPABLE 1 char]
    H -- yes --> I[Derive 1-byte value\nEBCDIK kana: FF61-FF9F to C0-FE]
    I --> J{kshifted is true}
    J -- yes --> J1{Output has space}
    J1 -- no --> J2[OVERFLOW]
    J1 -- yes --> J3[Write 0x29 to end shift; kshifted=false]
    J --> K{Output has space}
    K -- no --> K1[OVERFLOW]
    K -- yes --> K2[Write 1 byte and advance mark]
    K2 --> D

    G -- no --> L{Type handles JEF}
    L -- no --> L1[UNMAPPABLE 1 char]

    L -- yes --> M{PUA in U+E000..U+EC1D}
    M -- yes --> M1{Output has 2 bytes space}
    M1 -- no --> M2[OVERFLOW]
    M1 -- yes --> M3[Write PUA mapping; advance mark]
    M3 --> D

    M -- no --> N[Build key from c\nconsume low surrogate if needed]
    N --> N1{Surrogate issues}
    N1 -- malformed --> NM[MALFORMED]
    N1 -- need more --> NU[UNDERFLOW]
    N1 -- ok --> O
    O[Lookup records in JEF_MAP for key; select table by type] --> O1{Record exists at pos}
    O1 -- no --> OU[UNMAPPABLE 1 char]

    O1 -- yes --> P{Type handles shift and not kshifted}
    P -- yes --> P1{Output has space}
    P1 -- no --> P2[OVERFLOW]
    P1 -- yes --> P3[Write 0x28 to start shift; kshifted=true]

    P --> Q{Output has 2 bytes space}
    Q -- no --> Q1[OVERFLOW]
    Q -- yes --> R[Handle IVS if enabled; lookahead markers; may set mc or backup]
    R --> R1[Emit 2 bytes from mc]
    R1 --> R2[Advance mark by consumed; if pending error, return]
    R2 --> D
```

Key points:

- Single-byte mappings use `ASCII_MAP`, `EBCDIC_MAP`, `EBCDIK_MAP`.
- JEF double-byte mappings and IVS are resolved via `JEF_MAP`: `LongObjMap<Record[]>`.
- Shift mode uses 0x28 to enter and 0x29 to leave; `kshifted` tracks state.
- `implFlush()` ends shift (writes 0x28 if necessary) and flushes any backup.

## Decoding workflow (FujitsuCharsetDecoder.decodeLoop())

```mermaid
flowchart TD
    A[Start decodeLoop: ByteBuffer in; CharBuffer out] --> B[mark set to input position]
    B --> C{Input has more bytes}
    C -- no --> Z[UNDERFLOW]

    C -- yes --> D[Read next byte b]
    D --> E{Type handles shift}
    E -- yes --> ES{Shift byte}
    ES -- 0x28 or 0x38 --> ES1[kshifted=true; mark++; continue]
    ES -- 0x29 --> ES2[kshifted=false; mark++; continue]
    ES -- otherwise --> F
    E -- no --> F{Is shift code}
    F -- yes --> F1[UNMAPPABLE 1 byte]
    F -- no --> G

    G{kshifted == false &amp;&amp; map != null} -- yes --> H[Map byte to character\nEBCDIK kana: C0-FE to FF61-FF9F]
    H --> H1{Is 0xFF sentinel}
    H1 -- yes --> H2[UNMAPPABLE 1 byte]
    H1 -- no --> H3{Output has space}
    H3 -- no --> H4[OVERFLOW]
    H3 -- yes --> H5[Write char and advance mark]
    H5 --> C

    G -- no --> I{Type handles JEF and b in 0x40..0xFE}
    I -- no --> I1[UNMAPPABLE 1 byte]

    I -- yes --> J{Input has more bytes}
    J -- no --> J1[UNDERFLOW]
    J -- yes --> K[Read next byte b2]
    K --> K1{Both bytes are 0x40}
    K1 -- yes --> K2[Write U+3000; advance 2; continue]
    K1 -- no --> L{b2 is shift code}
    L -- yes --> L1[UNMAPPABLE 1 byte]
    L -- no --> M{b in 0x80..0xA0 and b2 in 0xA1..0xFE}
    M -- yes --> M1[Write PUA mapping; advance 2; continue]

    M -- no --> N[Lookup record in JEF_MAP; pos is low nibble of b2]
    N --> N1{Record exists at pos}
    N1 -- no --> N2[UNMAPPABLE 2 bytes]
    N1 -- yes --> O[Get mapped code mc; split to base and combi]
    O --> O1[baseLen 1 or 2 surrogate\ncombiLen 0/1/2 IVS allowed]
    O1 --> O2{Output has base+combi space}
    O2 -- no --> O3[OVERFLOW]
    O2 -- yes --> P[Write base character]
    P --> Q[If combiLen>0 and allowed, write combi]
    Q --> R[mark += 2; continue]
```

Key points:

- Single-byte decoding uses `map` when not in shift mode.
- JEF double-byte sequences decode using `JEF_MAP` and produce base character plus optional combination (IVS).
- Shift-in/out bytes handled when `type.handleShift()` is enabled.
- Private Use Area and full-width space (0x40 0x40) special cases.

## Shift state machine

```mermaid
stateDiagram-v2
    [*] --> ASCII_Mode
    ASCII_Mode --> JEF_Shifted : receive 0x28 or 0x38 (encoder writes 0x28)
    JEF_Shifted --> ASCII_Mode : receive 0x29 (encoder writes 0x29)
```

## How names resolve to charsets

```mermaid
sequenceDiagram
    participant App
    participant Provider as Jef4jCharsetProvider
    participant Charset as FujitsuCharset
    participant Type as FujitsuCharsetType

    App->>Provider: charsetForName("x-Fujitsu-JEF-ASCII")
    Provider->>Type: iterate values(), match by getCharsetName()
    Type-->>Provider: JEF_ASCII
    Provider->>Provider: map.computeIfAbsent(name, ...)
    Provider->>Charset: new FujitsuCharset(JEF_ASCII)
    Charset-->>App: Charset instance (newEncoder/newDecoder ready)
```

**Notes and guidance**
- The binary mapping tables are loaded from serialized resources: `FujitsuEncodeMap.dat` and `FujitsuDecodeMap.dat` inside `net/arnx/jef4j/` via `ObjectInputStream`. They populate `ASCII_MAP`, `EBCDIC_MAP`, `EBCDIK_MAP`, and `JEF_MAP`.

- The `Record` implementations (`ByteRecord`, `CharRecord`, `IntRecord`, `LongRecord`) provide compact storage and bit-pattern addressing; `exists(pos)` tells if an entry is present at a 4-bit bucket, and `get(pos)` returns the mapped value.

- `LongObjMap` is a specialized open-addressing hash map keyed by `long` for fast lookups of `Record[]` by masked keys.