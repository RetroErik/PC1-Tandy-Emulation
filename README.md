# Tandy Graphics Emulation TSR for Olivetti PC1 — Feasibility Study

Feasibility study for a TSR that intercepts Tandy 1000 graphics modes and redirects them to the Olivetti PC1's Yamaha V6355D hidden 16-color mode.

By **Retro Erik** — [YouTube: Retro Hardware and Software](https://www.youtube.com/@RetroErik)

![Olivetti Prodest PC1](https://img.shields.io/badge/Platform-Olivetti%20Prodest%20PC1-blue)
![License](https://img.shields.io/badge/License-CC%20BY--NC%204.0-green)

---

> **VERDICT:** A mode 8–only TSR is **not worth building**. Of the 49 mode 8 games, ~43 already work via our NTSC-16COLOR TSR (CGA composite), leaving only ~6 obscure PCjr cartridge exclusives as genuinely new. The real prize is **mode 9** (320×200×16, hundreds of games, no CGA equivalent) — but that requires a VRAM upgrade and unverified V6355D register work. A Tandy TSR only makes sense if we commit to mode 9.

---

## 1. The Goal

Create a TSR (Terminate and Stay Resident) program for the Olivetti Prodest PC1 that intercepts Tandy 1000 graphics mode calls and redirects them to the PC1's Yamaha V6355D video hardware. Two Tandy modes are relevant:

| Tandy Mode | Resolution | Colors | VRAM | Interleaving | Games |
|------------|-----------|--------|------|--------------|-------|
| **Mode 8** | 160×200 | 16 | **16KB** | 2-bank | 49+ titles |
| **Mode 9** | 320×200 | 16 | 32KB | 4-bank | Hundreds of titles |

This is inspired by the existing **NTSC-16COLOR TSR** which successfully intercepts CGA modes 4/5/6 and redirects them to the V6355D's hidden 160×200×16 mode with zero CPU overhead.

**Context:** Simone reportedly built something similar but had to modify the PC1 BIOS to make it work.

---

## 2. Reference: Why the NTSC TSR Works (Zero Overhead)

The NTSC TSR exploits three hardware properties of the V6355D that combine to create a **perfect zero-overhead mapping**:

| Property | Detail |
|----------|--------|
| **VRAM aliasing** | Segments B000h and B800h map to the **same physical 16KB RAM** — the V6355D ignores address bit A15. Games write to B800h, the 16-color display reads from B000h — same physical memory. |
| **Identical memory layout** | CGA modes 4/5/6 and the hidden 160×200×16 mode both use **2-bank CGA interlacing**: even rows at 0x0000, odd rows at 0x2000, with 80 bytes per row. |
| **Compatible pixel format** | Both use **packed nibbles** (4 bits per unit). CGA writes 2bpp pixel-pairs that become 4-bit nibble indices; the V6355D reads those same nibbles as 16-color palette indices. |

**Result:** No data conversion, no copying, no timer hooks — the game writes CGA data and the V6355D displays it as 16 colors automatically. The TSR only needs to activate the hidden mode and set the palette.

---

## 3. Tandy Modes vs PC1 Hidden Mode — The Critical Differences

### 3.1 Memory Layout Comparison

| Feature | PC1 160×200×16 | Tandy 160×200×16 (Mode 8) | Tandy 320×200×16 (Mode 9) |
|---------|----------------|---------------------------|---------------------------|
| **VRAM Segment** | B000h / B800h (aliased) | B800h | B800h |
| **Bytes per row** | 80 | **80** ✅ | **160** |
| **Pixel format** | 4bpp packed nibbles | 4bpp packed nibbles ✅ | 4bpp packed nibbles (same!) |
| **Interleaving** | **2-bank** (even/odd) | **2-bank** (even/odd) ✅ | **4-bank** (rows mod 4) |
| **Bank offsets** | 0x0000, 0x2000 | 0x0000, 0x2000 ✅ | 0x0000, 0x2000, 0x4000, 0x6000 |
| **Rows per bank** | 100 | 100 ✅ | 50 |
| **Total VRAM** | 16,000 bytes (16KB) | 16,000 bytes (16KB) ✅ | 32,000 bytes (32KB) |

**Mode 8 is a near-perfect match** — same resolution, same pixel format, same interlacing, same VRAM size. The same zero-overhead aliasing trick used by the NTSC TSR applies directly.

### 3.2 Tandy 4-Bank Interlacing (Confirmed from TANDYDOT.ASM)

Tandy mode 9 uses **4-bank CGA interlacing**, NOT the standard 2-bank layout:

```
Bank 0 (offset 0x0000): rows 0, 4, 8, 12, ...196  →  50 rows × 160 bytes = 8,000 bytes
Bank 1 (offset 0x2000): rows 1, 5, 9, 13, ...197  →  50 rows × 160 bytes = 8,000 bytes
Bank 2 (offset 0x4000): rows 2, 6, 10, 14,...198  →  50 rows × 160 bytes = 8,000 bytes
Bank 3 (offset 0x6000): rows 3, 7, 11, 15,...199  →  50 rows × 160 bytes = 8,000 bytes
                                                       Total: 32,000 bytes
```

### 3.3 Tandy Mode 8: Zero-Overhead Match (Stock Hardware)

**Mode 8 CAN use the NTSC TSR's zero-overhead trick.** The memory layouts are identical:

- Game writes to B800h → V6355D reads from B000h (same physical 16KB RAM)
- 2-bank interlacing: even rows at 0x0000, odd rows at 0x2000
- 80 bytes per row, 4bpp packed nibbles → each nibble = a palette color index
- 16KB total — fits perfectly in the PC1's stock VRAM

The TSR only needs to: activate the hidden 160×200×16 mode, load the standard 16 CGA colors into the V6355D palette, and return. **No data conversion, no copying, zero CPU overhead** — exactly like the NTSC TSR.

At least **49 games** are confirmed to use Tandy mode 8, primarily early PCjr/Tandy titles from 1984–1989 (see Section 6).

### 3.4 Why 160×200 Looks Great on Composite (and 320×200 Doesn't)

An often-overlooked advantage of mode 8 is its **composite video behavior**. In 160×200 mode, the pixel clock runs at **3.58 MHz** — exactly matching the NTSC color burst frequency. This means the TV/monitor's color decoding circuitry can perfectly track the color changes, producing **clean, artifact-free 16-color output**.

By contrast:
- **320×200** (mode 9): pixel clock = 7.16 MHz → artifact colors, especially ugly with dithering on Tandy/PCjr hardware (their composite timing is "off" compared to IBM CGA, producing different and less useful artifacts)
- **640×200**: pixel clock = 14.318 MHz → even worse artifacts

This is directly relevant to the PC1 — the V6355D's hidden mode outputs via composite, and the NTSC TSR already exploits these same artifact-free properties. **Tandy mode 8 games on the PC1 will look exactly as intended on a composite monitor**, while mode 9 games would suffer from Tandy's composite artifact issues even if the VRAM upgrade works.

AGI games like King's Quest use mode 9 (320×200) with pixel doubling — which effectively becomes 160×200 and also displays correctly on composite. These games use mode 8 on PCjr/Tandy when available.

### 3.5 Tandy Mode 9: Cannot Use Zero-Overhead on Stock Hardware

Three fundamental mismatches prevent a direct zero-overhead approach for mode 9:

1. **Row width mismatch (160 vs 80 bytes/row):** Tandy games write 160 bytes per row; the V6355D in 160×200×16 mode reads 80 bytes per row. Every row would be clipped at the halfway point.

2. **Bank interleaving mismatch (4-bank vs 2-bank):** Tandy writes to 4 banks; V6355D reads from 2 banks. Rows written to banks 2–3 (offsets 0x4000–0x7FFF) would either:
   - **With 16KB VRAM:** Wrap around and overwrite banks 0–1 (address aliasing), producing a garbled image.
   - **With 32+KB VRAM:** Be invisible to the V6355D display (it only reads from 0x0000 and 0x2000).

3. **VRAM size (32KB vs 16KB):** Tandy mode 9 requires twice the PC1's available video memory.

**Verdict:** The zero-overhead trick is **NOT applicable** to mode 9 on stock hardware. Mode 9 requires a VRAM upgrade.

---

## 4. Feasibility Analysis: Four Approaches

### Approach A0: TSR for Mode 8 on Stock Hardware (Zero Overhead) ★

**Concept:** Intercept INT 10h mode 8 (160×200×16), activate the V6355D's hidden 160×200×16 mode, load standard CGA palette. Done.

**This is structurally identical to the NTSC TSR.** The same VRAM aliasing trick applies:
- Game writes to B800h → same physical RAM → V6355D reads from B000h
- Same 2-bank interleaving, same 80 bytes/row, same 4bpp nibble format
- No data conversion, no timer hooks, zero CPU overhead

**What the TSR does:**
1. Intercept INT 10h AH=00h, AL=08h (mode 8 request)
2. Force CGA mode 4 via BIOS (sets CRTC defaults)
3. Activate hidden mode: `OUT 0xD8, 0x4A`
4. Load 16 standard CGA colors into V6355D DAC (ports 0xDD/0xDE)
5. Return — game runs with zero overhead

**Palette:** Unlike the NTSC TSR (which maps CGA pixel-pair artifacts to custom colors), a Tandy mode 8 TSR maps palette entries 1:1. Tandy color index 0 = black, 1 = blue, ... 15 = white. Each entry is loaded as the corresponding CGA RGB333 value into the V6355D.

**Compatibility:** 49+ games use mode 8 (see Section 6), including major Sierra AGI titles, LucasArts adventures, and many PCjr/Tandy exclusives from 1984–1989. These games explicitly set INT 10h mode 8 and write to B800h in the standard 2-bank CGA interlaced layout.

**Assessment: ✅ HIGHLY FEASIBLE — no hardware mod, zero overhead, proven architecture**

This can be built immediately using the NTSC TSR as a template. The only engineering work is the Tandy detection bypass and the CGA-to-RGB333 palette table.

---

### Approach A1: TSR-Only with Stock 16KB VRAM — Mode 9 Software Conversion

**Concept:** Intercept mode 9, activate 160×200×16, and use CPU to convert the Tandy framebuffer layout to the PC1 layout in real-time.

**What it would need to do:**
- Downsample 320→160 horizontal pixels (drop every other pixel, or merge pairs)
- Remap 4-bank → 2-bank interlacing
- Run on a timer interrupt (INT 08h) or intercept write operations

**Technical challenges:**

| Challenge | Severity | Notes |
|-----------|----------|-------|
| CPU overhead | **CRITICAL** | Converting 32KB of data per frame on a ~7.16 MHz NEC V40 is very expensive. A full 320→160 conversion + bank remapping takes ~50,000+ cycles per frame, consuming 15–25% of available CPU time. |
| Screen tearing | **HIGH** | Timer-driven conversion creates visible tearing — the game updates VRAM asynchronously, and the conversion lags behind. |
| Games that bypass INT 10h | **HIGH** | Tandy games write directly to B800h; there's no way to intercept individual memory writes without hardware support. |
| Halved resolution | **MODERATE** | 320→160 means every other pixel column is lost. Text becomes unreadable, fine details disappear. |
| Direct port I/O | **MODERATE** | Games that program Tandy registers directly (0x3DA/0x3DE for palette) won't work unless those ports are emulated. |

**Assessment: ❌ NOT RECOMMENDED**  
The CPU overhead, tearing, and halved resolution make this impractical. It defeats the purpose of the PC1's elegant hardware approach. Games would run noticeably slower and look worse than intended.

---

### Approach B: TSR + VRAM Upgrade (Hardware + Software)

**Concept:** Upgrade the PC1's VRAM from 16KB to 64KB (by replacing the two 4416 DRAM chips with pin-compatible 4464 chips), then configure the V6355D for a true 320×200×16 mode that matches Tandy's memory layout.

**Hardware requirement:** Replace 2× TI TMS4416 (16K×4) with 2× TMS4464 (64K×4) DRAM chips. These are pin-compatible drop-in replacements.

**Software approach:**

1. **Intercept INT 10h AH=00h AL=09h** (Tandy mode 9 request)
2. **Configure V6355D for 320×200×16:**
   - Set CRTC registers via 0x3D4/0x3D5 for 160 bytes/row (40 "characters" × 4 bytes/char)
   - Set port 0x3D8 = 0x4A (hidden mode unlock: bit 6 + graphics enable)
   - Set Register 0x67 bit 6 = 1 (enable 4-page VRAM mode)
   - Disable pixel doubling (so 320 logical pixels = 320 physical pixels)
3. **Set V6355D palette** to match Tandy's default 16 CGA colors
4. **Return to caller** — games write directly to B800h, V6355D displays from B000h (aliased)

**The key question: Does the V6355D in 4-page mode use 4-bank interleaving identical to Tandy?**

If YES → **zero-overhead** like the NTSC TSR. Games write Tandy data, V6355D reads it directly.  
If NO → CPU conversion is still needed, but with full resolution (no downsampling).

**Evidence suggesting YES:**
- The V6355D documentation explicitly mentions Register 0x67 bit 6 as "4-page mode" for systems with 64KB VRAM
- The V6355D is CGA-heritage hardware — 4-bank interleaving is the standard way CGA-family chips handle >16KB modes
- The documentation calls 320×200×16 "likely — Tandy/PC1 variant, timing compatible with 320×200×4"
- The ACV-1030 card (same V6355 chip family) supports 16-color modes with larger VRAM

**Evidence for uncertainty:**
- 320×200×16 on V6355D has **never been tested** on actual PC1 hardware
- The specific CRTC register values for this mode are unknown
- Register 0x67 bit 6 behavior is documented from the Z-180 manual, not verified on PC1
- The bank interleaving order (whether banks 0–3 use the same offsets as Tandy) is unconfirmed

**Palette handling:**
- Tandy games set palette via INT 10h AH=10h subfunctions or direct Tandy port I/O
- The TSR would intercept INT 10h AH=10h and redirect to V6355D DAC (ports 0xDD/0xDE)
- Direct Tandy port I/O (0x3DA/0x3DE) would NOT be interceptable — but these ports have different meanings on the V6355D, potentially causing conflicts

**What CRTC values might look like (estimated):**

```
160×200×16 (working):          320×200×16 (theoretical):
  Bytes/row:     80               Bytes/row:     160
  CRTC R01:      40 (chars)       CRTC R01:      80 (chars)?
  CRTC R00:      ~49 (total)      CRTC R00:      ~113 (total)?
  Port 0x3D8:    0x4A             Port 0x3D8:    0x4A (same?)
  Reg 0x67 b6:   0 (1-page)      Reg 0x67 b6:   1 (4-page)
```

**Assessment: ⚠️ MOST PROMISING — but requires hardware mod + experimentation**

This is the most likely path to success. If the V6355D supports 320×200×16 natively (which the documentation suggests is "likely"), this could be as elegant as the NTSC TSR. However, it requires:
1. Physical VRAM chip replacement (4416 → 4464)
2. Experimental register probing on real hardware
3. Discovering the correct CRTC values

---

### Approach C: TSR + BIOS ROM Modification (Simone's Approach)

**Concept:** Modify the PC1's BIOS ROM (replace with EPROM) to add native Tandy mode 9 support, then use a lightweight TSR for any remaining interception.

**Why modify the BIOS?**
- The BIOS initializes VRAM sizing during POST — with 64KB VRAM, the BIOS may not initialize correctly without a patch
- BIOS mode-set routines (INT 10h) handle screen clearing, cursor positioning, and CRTC programming — modifying these ensures proper support
- A BIOS-level implementation is cleaner and more reliable than a TSR intercepting BIOS calls
- Some games call BIOS functions that need to know the current mode's properties (rows, columns, VRAM size)

**What BIOS modifications would involve:**
1. Add mode 9 to the BIOS mode table (CRTC values, VRAM parameters)
2. Modify the mode-set handler to recognize AL=09h
3. Handle VRAM clearing for 32KB (instead of 16KB)
4. Add or modify palette functions (AH=10h) to use V6355D DAC
5. Burn modified BIOS to EPROM (requires EPROM programmer + socket adapter)

**Assessment: ✅ PROVEN VIABLE (Simone did this)**  
This is confirmed to work. The main barrier is the complexity of reverse-engineering and modifying the PC1 BIOS ROM, plus the hardware work of burning an EPROM and installing it.

---

## 5. Tandy INT 10h Calls That Need Interception

For any TSR approach, these INT 10h functions must be handled:

| Function | Purpose | Required |
|----------|---------|----------|
| **AH=00h, AL=08h** | Set Tandy mode 8 (160×200×16) | **CRITICAL** — zero-overhead mode |
| **AH=00h, AL=88h** | Set mode 8, no clear screen | **CRITICAL** — same with bit 7 set |
| **AH=00h, AL=09h** | Set Tandy mode 9 (320×200×16) | **CRITICAL** — main mode activation |
| **AH=00h, AL=89h** | Set mode 9, no clear screen | **CRITICAL** — same with bit 7 set |
| **AH=0Fh** | Get current video mode | **HIGH** — must return mode 9, 40 columns |
| **AH=10h, AL=00h** | Set individual palette register | **HIGH** — redirect to V6355D DAC |
| **AH=10h, AL=02h** | Set all 16 palette registers | **HIGH** — redirect to V6355D DAC |
| **AX=1A00h** | Get display combination code | **MODERATE** — report Tandy/MCGA |
| **AH=0Bh** | Set color palette (CGA legacy) | **LOW** — rarely used in mode 9 |

### Palette Mapping Challenge

Tandy palette registers use a **CGA-compatible 4-bit color index** (16 fixed CGA colors: 0=black, 1=blue, ..., 15=white). The V6355D uses **programmable RGB333** (3 bits per channel, 512 colors via ports 0xDD/0xDE).

The TSR must maintain a **CGA-to-RGB333 lookup table** to translate Tandy palette values to V6355D DAC values:

```
Tandy index 0  (Black)       → V6355D: R=0, G=0x00, B=0  (0, 0x00)
Tandy index 1  (Blue)        → V6355D: R=0, G=0x00, B=5  (0, 0x05)
Tandy index 2  (Green)       → V6355D: R=0, G=0x50, B=0  (0, 0x50)
...
Tandy index 15 (White)       → V6355D: R=7, G=0x70, B=7  (7, 0x77)
```

---

## 6. Games That Would Benefit

### 6.1 Mode 8 Games (160×200×16 — Stock Hardware, Zero Overhead)

The following **49 games** are confirmed to use mode 8 (160×200×16) on PCjr/Tandy 1000. Source: [Great Hierophant, "Advantages of the 160x200 16-color Tandy/PCjr. Resolution," Nerdly Pleasures, October 2015](https://nerdlypleasures.blogspot.com/2015/10/advantages-of-160x200-16-color.html).

**All 49 of these would work on a stock PC1 with the mode 8 TSR — zero overhead, no hardware mod.**

| # | Game | Notes |
|---|------|-------|
| 1 | Black Cauldron, The | Sierra AGI |
| 2 | Boulder Dash | PCjr port |
| 3 | Boulder Dash II: Rockford's Revenge | PCjr port |
| 4 | Bruce Lee | PCjr port |
| 5 | California Games | Epyx |
| 6 | Demon Attack | PCjr cartridge |
| 7 | Donald Duck's Playground | Sierra AGI |
| 8 | F-15 Strike Eagle | MicroProse |
| 9 | Ghostbusters | Activision |
| 10 | Gold Rush! | Sierra AGI |
| 11 | Indianapolis 500 | |
| 12 | Jumpman | Epyx/PCjr cartridge |
| 13 | King's Quest I: Quest for the Crown | Sierra AGI¹ |
| 14 | King's Quest II: Romancing the Throne | Sierra AGI¹ |
| 15 | King's Quest III: To Heir is Human | Sierra AGI¹ |
| 16 | King's Quest IV: The Perils of Rosella | Sierra AGI¹ |
| 17 | Leisure Suit Larry in the Land of the Lounge Lizards | Sierra AGI¹ |
| 18 | Lost Tomb | |
| 19 | Manhunter 2: San Francisco | Sierra AGI |
| 20 | Manhunter: New York | Sierra AGI |
| 21 | Maniac Mansion | LucasArts (uses 4×8 font for 20-column text) |
| 22 | Mickey's Space Adventure | Sierra |
| 23 | Microsoft Flight Simulator (v2.0) | |
| 24 | Microsurgeon | PCjr cartridge |
| 25 | Mixed-Up Mother Goose | Sierra AGI |
| 26 | Mouser | PCjr port |
| 27 | Murder on the Zinderneuf | |
| 28 | Ninja | |
| 29 | Dr. J and Larry Bird go One-on-One | EA |
| 30 | Pitfall II: Lost Caverns | Activision |
| 31 | Pitstop II | Epyx |
| 32 | Police Quest: In Pursuit of the Death Angel | Sierra AGI |
| 33 | Rasterscan | |
| 34 | River Raid | Activision/PCjr |
| 35 | ScubaVenture | IBM PCjr exclusive |
| 36 | Sea Speller | IBM PCjr exclusive |
| 37 | Silent Service | MicroProse |
| 38 | Slugger, The | |
| 39 | Space Quest I: The Sarien Encounter | Sierra AGI |
| 40 | Space Quest II: Vohaul's Revenge | Sierra AGI |
| 41 | Starflight | EA/Binary Systems |
| 42 | Storm | |
| 43 | Super Bowl Sunday | |
| 44 | The World's Greatest Baseball Game | Epyx |
| 45 | Touchdown Football | |
| 46 | Troll's Tale | Sierra (PCjr-only detection²) |
| 47 | Winnie the Pooh in the Hundred Acre Wood | Sierra |
| 48 | Wizard and the Princess, The | Sierra |
| 49 | Zak McKracken and the Alien Mindbenders | LucasArts |

¹ **AGI note:** Sierra AGI games render internally at 160×200 and use mode 8 on Tandy/PCjr, but on Tandy they can also use mode 9 (320×200) with pixel doubling. The mode 8 path is the native resolution — no quality loss.

² **Detection note:** Troll's Tale was released before the Tandy 1000 existed and only checks for PCjr hardware (BIOS byte, not Tandy signature). The TSR's Tandy detection bypass alone may not be sufficient — may also need PCjr detection spoofing.

**Game categories:**
- **Sierra AGI titles** (14 games): All AGI games render at 160×200 internally, so mode 8 is their native resolution
- **PCjr cartridge/exclusive ports** (7+ games): Originally designed for PCjr's 16KB mode
- **Epyx/EA/Activision ports** (6+ games): Cross-platform titles with 160×200 paths
- **MicroProse simulations** (2 games): F-15, Silent Service
- **LucasArts adventures** (2 games): Maniac Mansion, Zak McKracken

### 6.2 NTSC TSR Overlap — Why Mode 8 Alone Isn't Worth It

Our existing **NTSC-16COLOR TSR** intercepts CGA modes 4/5/6 and displays them on the V6355D with **100% pixel-perfect accuracy and ~90% color accuracy**. Since nearly all mode 8 games also have CGA fallback modes, they already work well on the PC1.

**~43 of the 49 mode 8 games already have CGA composite support:**
- All 14 Sierra AGI titles (use CGA mode 6 composite — our best mode)
- Both LucasArts titles (Maniac Mansion, Zak McKracken)
- MicroProse titles (F-15 Strike Eagle, Silent Service)
- EA titles (Starflight, Dr. J One-on-One)
- Epyx titles (California Games, Boulder Dash 1–2, Pitstop II, Jumpman, World's Greatest Baseball)
- Activision titles (Ghostbusters, Pitfall II, Bruce Lee)
- Indianapolis 500, MS Flight Simulator 2.0, and most others

**Only ~6 games are PCjr cartridge exclusives with NO CGA fallback:**

| Game | Why no CGA |
|------|------------|
| Demon Attack | PCjr cartridge (Imagic) |
| ScubaVenture | IBM PCjr exclusive |
| Sea Speller | IBM PCjr exclusive |
| Microsurgeon | PCjr cartridge |
| Mouser | PCjr cartridge port |
| River Raid | PCjr cartridge version (separate CGA disk version may exist) |

These are obscure PCjr titles. **6 games do not justify building a separate TSR.**

A mode 8 TSR would give "true 16-color artwork" vs CGA composite approximations for the other ~43 games — a visual improvement, but our NTSC TSR's 90% color accuracy is already very good. The upgrade from "very good" to "perfect" for games that already work is not compelling enough on its own.

### 6.3 Mode 9 Games (320×200×16 — Requires VRAM Upgrade)

Tandy mode 9 (320×200×16) is supported by **hundreds of DOS games** from the mid-1980s to early 1990s, including:

- Sierra SCI0 games (King's Quest IV SCI, Leisure Suit Larry 2–3, Space Quest III, Police Quest 2)
- SSI Gold Box RPGs (Pool of Radiance, Curse of the Azure Bonds)
- Thexder, Silpheed, Prince of Persia, SimCity, Lemmings
- Many Epyx, Electronic Arts, Origin, and MicroProse titles

These games detect Tandy hardware via INT 10h or BIOS signature checks, then use INT 10h AH=00h AL=09h to set mode 9. Mode 9 requires the 4464 VRAM upgrade (see Approach B).

---

## 7. Side-by-Side: NTSC TSR vs Tandy TSR(s)

| Aspect | NTSC TSR (Working) | Tandy Mode 8 TSR | Tandy Mode 9 TSR |
|--------|-------------------|------------------|------------------|
| **Intercepted modes** | CGA 4, 5, 6 | Tandy 8 | Tandy 9 |
| **Target mode** | 160×200×16 | 160×200×16 | 320×200×16 |
| **VRAM needed** | 16KB (stock) | **16KB (stock)** ✅ | 32KB (needs 4464 upgrade) |
| **Memory layout match** | ✅ Perfect | **✅ Perfect** | ⚠️ Unknown (4-bank, 160 B/row) |
| **CPU overhead** | Zero | **Zero** | Zero (if V6355D supports it natively) |
| **Palette handling** | Custom NTSC colors | CGA 16-color → RGB333 | CGA 16-color → RGB333 |
| **Hardware mod needed** | None | **None** ✅ | VRAM chip replacement |
| **BIOS mod needed** | None | **None** | Likely |
| **Tandy detection bypass** | Not needed | **YES** | **YES** |
| **Game count** | ~60+ (CGA composite) | **49+** | Hundreds |

---

## 8. Tandy Hardware Detection — Why Games Won't Just Work

Tandy-aware games typically detect Tandy hardware before using mode 9. Common detection methods:

### Method 1: BIOS Signature Check
```asm
mov ax, 0FC00h          ; BIOS ROM segment
mov es, ax
mov ah, [es:3FFEh]      ; Model ID byte
mov al, [es:0]          ; Signature byte
cmp ax, 0FF21h          ; Tandy 1000 magic number
```
**PC1 will FAIL this check** — the PC1 BIOS has different signature bytes.

### Method 2: INT 15h AH=C0h (Get Configuration)
```asm
mov ah, 0C0h
int 15h                 ; Carry clear = Tandy Video II
```
**PC1 will FAIL this check** — different system configuration.

### Method 3: Equipment Word Check
```asm
int 11h                 ; Get equipment word
and al, 30h             ; Video bits
cmp al, 10h             ; 01 = Tandy/PCjr
```
**PC1 will FAIL this check** — the PC1 reports CGA video.

### TSR Solution
The TSR must also intercept these detection mechanisms:
- **INT 15h AH=C0h** → respond as Tandy Video II
- **INT 11h** → modify video equipment bits to report Tandy
- **BIOS signature** → cannot be easily intercepted (direct memory reads). Options:
  - Patch a few bytes at FC00:0000 and FC00:3FFE in RAM range (if not ROM-mapped)
  - Or accept that some games won't detect the hardware (user runs them with `/t` or Tandy override switches)

### Prior Art: Tand-Em and SimCGA

Similar attempts have been made before:
- **Tand-Em** — a program to simulate Tandy/PCjr hardware for self-booting disks on standard PCs. According to Great Hierophant, it is "very much out of date now." It tackled the same detection-bypass problem.
- **SimCGA** — a well-known TSR for running CGA software on Hercules adapters. A blog commenter on Nerdly Pleasures asked about creating a "SimCGA-like" patch to run Tandy 16-color games on CGA — essentially the exact concept of this project, but we have the advantage of the V6355D's native 16-color mode instead of dithering to monochrome.

### PCjr vs Tandy Detection Differences

Some early games (pre-1985) only check for PCjr hardware, not Tandy. Example: Troll's Tale checks a specific BIOS byte to detect PCjr but does not check the Tandy signature (0xFF21h), since it was released before the Tandy 1000 existed. The TSR may need to spoof **both** PCjr and Tandy detection paths to cover the full 49-game library.

---

## 9. Key Technical Unknowns (Requires Hardware Testing)

| Unknown | How to Test | Impact |
|---------|-------------|--------|
| Does V6355D support 320×200×16 with 64KB VRAM? | Replace DRAM chips, try Register 0x67 bit 6 = 1, program CRTC, write test pattern | **BLOCKING** — determines if Approach B is viable |
| Does 4-page mode create 4-bank interlacing? | Same as above — check if data at 0x4000/0x6000 displays correctly | **BLOCKING** — determines if zero-overhead works |
| What CRTC values does 320×200×16 need? | Start from CGA mode 4 values, double horizontal displayed characters | **HIGH** — needed for mode setup |
| Does the BIOS initialize correctly with 64KB VRAM? | Boot PC1 with 4464 chips, observe behavior | **MODERATE** — determines if BIOS patch is needed |
| Can BIOS signature bytes be shadowed to RAM? | Check if FC00h segment is ROM or can be overlaid | **LOW** — only affects games using signature detection |

---

## 10. Recommendations

### Do NOT build a mode 8–only TSR
The overlap with our NTSC-16COLOR TSR is too large. Only ~6 obscure PCjr exclusives would benefit. Not worth the effort.

### If we pursue a Tandy TSR at all, go straight to mode 9
Mode 9 (320×200×16) is the real prize — hundreds of games (SCI0, Gold Box RPGs, Prince of Persia, etc.) with true 16-color artwork that CGA composite **cannot** reproduce. This requires:

1. **Acquire 4464 DRAM chips** (TMS4464 or equivalent, widely available as NOS/pulls)
2. **Perform the VRAM upgrade** on a PC1 (swap 2× 4416 → 2× 4464)
3. **Write a test program** (not TSR) to answer the blocking unknowns:
   - Set Register 0x67 bit 6 = 1 (4-page mode)
   - Program CRTC for 160 bytes/row
   - Write a test pattern in 4-bank interleaved layout to B800h
   - Activate hidden mode (port 0x3D8 = 0x4A)
   - See if it displays correctly
4. **If step 3 succeeds:** Build a full Tandy TSR handling mode 9 (and mode 8 for free)
5. **If step 3 fails:** Consider Simone's BIOS modification approach (Approach C)

### Mode 8 comes for free with Mode 9
Any TSR that handles mode 9 can trivially include mode 8 — it's the simpler case. No reason to build mode 8 separately.

---

## 11. Conclusion

| Approach | Tandy Mode | Feasibility | Quality | Hardware Mod | Effort |
|----------|------------|-------------|---------|--------------|--------|
| **A0: Mode 8 TSR (stock)** | **8** (160×200) | ✅ Feasible but **not worth it** | Native (zero overhead) | None | Low |
| **A1: Mode 9 TSR (16KB)** | 9 (320×200) | ❌ Poor | Low (CPU overhead, halved) | None | Medium |
| **B: Mode 9 TSR + VRAM** | 9 (320×200) | ⚠️ Promising | High (zero overhead) | DRAM swap | Medium-High |
| **C: BIOS mod + VRAM** | 8 + 9 | ✅ Proven | Highest (native) | DRAM + EPROM | High |

### Mode 8 is not worth it standalone

Of the 49 mode 8 games, ~43 already work via our **NTSC-16COLOR TSR** through CGA composite (100% pixel-perfect, ~90% color-accurate). Only ~6 obscure PCjr cartridge exclusives have no CGA fallback. That's not enough to justify a new TSR.

### Mode 9 is the only reason to build this

Mode 9 (320×200×16) unlocks **hundreds of games** with true 16-color artwork that CGA composite cannot reproduce — SCI0 adventures, Gold Box RPGs, Prince of Persia, SimCity, Lemmings, and many more. But it requires a VRAM upgrade (4416 → 4464) and unverified V6355D register configuration.

### Bottom line

A Tandy TSR only makes sense if we commit to mode 9 and the VRAM upgrade. Mode 8 comes for free with that effort. The first step is acquiring 4464 DRAM chips and testing whether the V6355D can do 320×200×16 at all.

---

## Appendix A: Tandy 4-Bank Pixel Address Calculation

From TANDYDOT.ASM (verified reference code):

```asm
; Tandy mode 9 pixel plot: x, y, colour → B800h
;
; Memory = 4 banks × 8,000 bytes at B800:0000, B800:2000, B800:4000, B800:6000
; Each bank holds every 4th row: bank = y mod 4
; Row stride = 160 bytes (320 pixels ÷ 2 pixels/byte)

mov ax, 0B800h
mov es, ax              ; VRAM segment

mov ax, y
ror ax, 1               ; Extract bit 0 of y → bit 15
ror ax, 1               ; Extract bit 1 of y → bit 15, bit 0 → bit 14
mov di, ax              ; DI = bank_select bits in upper positions
xor ah, ah              ; AX = y >> 2 (row within bank)
sub di, ax              ; DI = bank base offset (0x0000/0x2000/0x4000/0x6000)

xchg ah, al             ; AX = (y >> 2) * 256
add di, ax              ; Add row×256
shr ax, 1
shr ax, 1
add di, ax              ; Add row×64 → total = row×320 (nibble offset in bank)

add di, x               ; Add pixel X position (nibble offset)
shr di, 1               ; Convert nibble offset → byte offset
```

## Appendix B: V6355D Hidden Mode Activation (Reference)

```asm
; Activate 160×200×16 mode on stock PC1 (16KB VRAM)
mov al, 0x4A            ; Bit 6 (hidden mode unlock) + 0x0A (CGA mode 4 base)
out 0xD8, al            ; Port 0xD8 = mode control

; Set border to black
xor al, al
out 0xD9, al            ; Port 0xD9 = color select

; Load 16-color palette via V6355D DAC
mov al, 0x40            ; 0x40 = start palette write at entry 0
out 0xDD, al            ; Port 0xDD = palette address
; Write 32 bytes (16 colors × 2 bytes: R, then G|B packed)
; ... (OUTSB or loop with OUT 0xDE, al) ...
mov al, 0x80            ; 0x80 = end palette write
out 0xDD, al
```

## Appendix C: Theoretical 320×200×16 Mode Setup (Untested)

```asm
; THEORETICAL — requires 64KB VRAM upgrade (4464 DRAM)
; Based on V6355D register documentation + Tandy mode 9 analysis

; Step 1: Set standard CGA mode 4 via BIOS (establishes CRTC defaults)
mov ax, 0004h
int 10h

; Step 2: Enable 4-page VRAM mode
mov al, 0x67            ; Register 0x67 (configuration)
out 0xDD, al
in al, 0xDE             ; Read current value
or al, 0x40             ; Set bit 6 = 1 (4-page mode)
out 0xDE, al

; Step 3: Reprogram CRTC for 160 bytes/row (80 character positions × 2 bytes)
; These values are ESTIMATED — need hardware testing
mov dx, 0x3D4
mov al, 0x01            ; CRTC R01: Horizontal Displayed
out dx, al
inc dx
mov al, 80              ; 80 characters × 2 bytes = 160 bytes/row
out dx, al

; Step 4: Activate hidden 16-color mode
mov al, 0x4A            ; Bit 6 + CGA mode 4 base
out 0xD8, al

; Step 5: Load default CGA 16-color palette
; ... (same palette loading as NTSC TSR) ...

; If this works: B800h data in 4-bank layout → V6355D displays 320×200×16
```
