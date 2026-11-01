# symphonia-play dist build size report

**Build:** `target/dist/symphonia-play` — 2,772,464 B (2.64 MiB), arm64 Mach-O, built with the pinned `nightly-2026-09-15` toolchain. Added `[profile.dist]` (fat LTO, `codegen-units=1`, `panic=abort`, `strip`) mirroring ish — commit `e116cc0`. `--version` runs. Links only macOS system frameworks (AudioToolbox, CoreAudio, Foundation, CoreFoundation, libSystem, libiconv, libobjc), no Rust dylibs.

## File layout

| Region | Bytes | % file |
|---|---:|---:|
| `__TEXT` | 2,621,440 | 94.5% |
| — `__text` (code) | 2,368,988 | 85.4% |
| — `__const` | 160,880 | 5.8% |
| — `__cstring` | 32,855 | 1.2% |
| — `__eh_frame` / `__unwind_info` | 42,928 | 1.5% |
| — stubs / except tables / padding | 15,789 | 0.6% |
| `__DATA_CONST` (`__const` 84,208) | 98,304 | 3.5% |
| `__DATA` | 16,384 | 0.6% |
| `__LINKEDIT` | 36,336 | 1.3% |

The unstripped twin is 3,547,248 B, so stripping removes ~775 KB of symbol/string metadata without touching code.

## Code by crate

Symbol-range attribution from the unstripped twin; ±320 B vs. final `__text`.

| Group | Bytes | % code |
|---|---:|---:|
| Symphonia crates | 839,808 | 35.5% |
| std + core + alloc + backtrace (gimli/addr2line/rustc_demangle/hashbrown) | 585,264 | 24.7% |
| `env_logger` + regex stack | 490,440 | 20.7% |
| `rustfft` (via rubato) | 181,528 | 7.7% |
| `clap_builder` | 158,224 | 6.7% |
| everything else | 113,404 | 4.8% |

Largest individual crates: `core` 296K, `regex_automata` 268K, `rustfft` 182K, `clap_builder` 158K, `symphonia_core` 145K, `std` 119K, `regex_syntax` 115K, `symphonia_format_isomp4` 110K, `symphonia_metadata` 109K, `aho_corasick` 87K, `symphonia_format_mkv` 85K, `symphonia_play` 77K.

Largest functions: `regex_automata::meta::strategy::new` 49,352; clap parser `get_matches_with` 29,048; PCM `decode_inner` 28,508; MP3 Layer3 `decode` 25,972; `symphonia_play::main` 22,524.

Notable static data: CRC32 table 32,412; `log::NOP` 23,690; Unicode whitespace map 17,432; float dec2flt table 10,496; AAC spectrum tables 31,600 + MP3 codebooks 3,832; `log::LOG_LEVEL_NAMES` 32,328.

## Compressed

| Format | Bytes | Size |
|---|---:|---:|
| gzip -9 | 1,353,898 | 1.29 MiB |
| xz -9 | 965,672 | 943 KiB |

## Notes

Biggest size lever is `pretty_env_logger` → `env_logger` → `regex`: ~490 KB of code (~21%) for log filtering. Swapping in `env_logger` without the regex feature or a tiny custom logger would recover most of that. Symphonia's own `symphonia-metadata` `regex_lite` is only 35 KB.

Verified: dist build compiles and runs; section numbers from `size -m`/`otool`; crate numbers from `nm` + `rustfilt` on the unstripped build. Not done: Linux static/musl cross build.

Open decisions: keep `panic=abort` in `dist`? Trim the `env_logger` regex stack? Also produce the ish-style static musl build for Linux?
