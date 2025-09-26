# Below is made by Copilot

# Changelog (Fork Only)

All notable changes in the private fork will be documented here.

## [v1.1-fork1] - 2025-09-27
### Added
- Windows MSYS2 MinGW build support (32-bit & 64-bit) without p11-kit dependency.
- Fallback dynamic loader for Windows (`LoadLibraryA` / `GetProcAddress`) with `NO_DLFCN` logic.
- VS Code tasks for Meson workflow.
- Fork-specific documentation: `FORK_NOTES.md`, `README-win32.md`.

### Changed
- `src/interface.c`: Added conditional dynamic loading block, const correctness for `dlerror()` shim, explicit casts for dlsym results.
- `src/provider.h`: Added `<pthread.h>` include early for mutex/rwlock types.
- `src/platform/endian.h`: Guard inclusion of absent headers on Windows.

### Fixed
- Build failures on Windows (missing dlfcn.h, byteswap.h, pthread types).
- Reduced warnings for discarded qualifiers and unused values.

### Notes
- No functional crypto changes; portability only.
- Default PKCS#11 module still unset if p11-kit not present?must set `PKCS11_PROVIDER_MODULE`.

## Unreleased
- SoftHSM2 32-bit build instructions (pending actual compiled artifact in this fork).
