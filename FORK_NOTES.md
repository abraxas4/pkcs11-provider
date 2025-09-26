# Below is made by Copilot

# Fork Notes (abraxas4/pkcs11-provider)

This fork keeps private Windows (MSYS2 / MinGW) focused adjustments without submitting them upstream yet.

## Scope
- Enable build and runtime on Windows 11 MSYS2 MinGW32 & MinGW64.
- Provide 32-bit pkcs11 provider module for integration with a 32-bit only application.
- Do NOT depend on p11-kit (no default proxy) unless the user builds it manually.

## Key Changes vs Upstream
| Area | File(s) | Change | Reason |
|------|---------|--------|--------|
| Dynamic Loading | `src/interface.c` | Added Windows fallback (LoadLibraryA/GetProcAddress) when `dlfcn.h` absent, defined RTLD_* stubs, dlerror shim | Allow clean build under MinGW without dlfcn.h |
| Thread Types | `src/provider.h` | Included `<pthread.h>` early | Fix undefined pthread_mutex_t / rwlock types in multiple C files |
| Endian Helpers | `src/platform/endian.h` | Guard inclusion of `<byteswap.h>` / `<endian.h>` on Windows; rely on fallback macros | MinGW32 missing those headers |
| Const / Warnings | `src/interface.c` | Cast `dlsym` results + make dlerror const to reduce warnings | Cleaner build output |
| VS Code Tasks | `.vscode/tasks.json` | Added Meson setup/compile/test/install tasks | Easier iteration in VS Code |

## Build Matrix (Tested)
| Arch | Shell | GCC | OpenSSL | Status |
|------|-------|-----|---------|--------|
| x86_64 | MSYS2 MinGW64 | 15.2.0 | 3.5.3 | OK |
| i686 (32-bit) | MSYS2 MinGW32 | 15.2.0 | 3.5.3 | OK (provider loads once PKCS11 module specified) |

## 32-bit Build Quick Steps
```
pacman -S --needed mingw-w64-i686-gcc mingw-w64-i686-openssl \
  mingw-w64-i686-meson mingw-w64-i686-ninja mingw-w64-i686-pkgconf
meson setup builddir-32
meson compile -C builddir-32
meson install -C builddir-32
```
Resulting module:
```
C:\msys64\mingw32\lib\ossl-modules\pkcs11.dll
```

## Runtime Environment
Set the OpenSSL modules directory (example MinGW32 shell):
```
export OPENSSL_MODULES=/mingw32/lib/ossl-modules
```
Then specify the actual PKCS#11 driver (token) you want the provider to bridge:
```
export PKCS11_PROVIDER_MODULE=/path/to/vendorpkcs11.dll
openssl list -providers -provider pkcs11 -provider default
```
If no driver is specified you'll see: `No PKCS#11 module specified.`

## SoftHSM2 (Not packaged for i686 in MSYS2)
To test locally you must build SoftHSM2 from source (OpenSSL backend) or use a vendor module. Steps (abridged):
```
pacman -S --needed mingw-w64-i686-cmake mingw-w64-i686-sqlite3 mingw-w64-i686-pkgconf
# Clone SoftHSMv2
cd /d/Github
git clone https://github.com/opendnssec/SoftHSMv2.git softhsm2-src
cd softhsm2-src
mkdir build32 && cd build32
cmake -G Ninja -DCMAKE_BUILD_TYPE=Release -DCMAKE_INSTALL_PREFIX=/mingw32 \
  -DBUILD_SHARED_LIBS=ON -DSOFTHSM2_WITH_BOTAN=OFF -DSOFTHSM2_WITH_OPENSSL=ON ..
ninja && ninja install
export PKCS11_PROVIDER_MODULE=/mingw32/bin/libsofthsm2.dll
```

## Token Quick Start
```
mkdir -p ~/softhsm2/tokens ~/.config/softhsm2
cat > ~/.config/softhsm2/softhsm2.conf <<'EOF'
directories.tokendir = /home/$USER/softhsm2/tokens
objectstore.backend = file
slots.removable = true
EOF
softhsm2-util --init-token --free --label testtoken --so-pin 0000 --pin 1234
softhsm2-util --gen-keypair --token testtoken --label rsa1 --id 01 --pin 1234 --key-type RSA:2048
openssl storeutl -provider pkcs11 -provider default -text 'pkcs11:token=testtoken'
```

## Packaging Guidance
Deliverables for internal consumers:
```
dist/
  win32/
    pkcs11.dll
    LICENSE-Apache-2.0.txt
    README-win32.txt
    SHA256SUMS
```
Compute hash:
```
sha256sum /mingw32/lib/ossl-modules/pkcs11.dll
```

## Version Tagging (Fork Only)
Use tags like: `v1.1-fork1`, `v1.1-fork1-win32fix2`.
Maintain a fork-specific changelog (see `CHANGELOG_FORK.md`).

## Upstream Sync (Deferred / Optional)
If later desired:
```
git remote add upstream https://github.com/latchset/pkcs11-provider.git
git fetch upstream
git merge --ff-only upstream/main
```

## Notes
- No functional alteration to crypto logic; changes are portability only.
- Keep INTERNAL patches rebased to reduce divergence.
- Consider a CI job later for MinGW32 build smoke test.

