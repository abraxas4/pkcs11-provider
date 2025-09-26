# Below is made by Copilot

# Windows (MSYS2 MinGW) Usage Guide

This document is fork-specific and describes how to build and use the pkcs11 OpenSSL provider on Windows 11 using MSYS2 MinGW (32-bit focus, 64-bit similar).

## 1. Prerequisites (32-bit)
Install MSYS2 then open the *MSYS2 MinGW32* shell.

Packages:
```
pacman -S --needed mingw-w64-i686-gcc mingw-w64-i686-openssl \
  mingw-w64-i686-meson mingw-w64-i686-ninja mingw-w64-i686-pkgconf git
```

## 2. Build
```
meson setup builddir-32
meson compile -C builddir-32
meson install -C builddir-32
```
Result: `C:\msys64\mingw32\lib\ossl-modules\pkcs11.dll`

## 3. Testing Load
```
export OPENSSL_MODULES=/mingw32/lib/ossl-modules
# Must specify the actual PKCS#11 driver:
export PKCS11_PROVIDER_MODULE=/path/to/vendorpkcs11.dll
openssl list -providers -provider pkcs11 -provider default
```
If you omit `PKCS11_PROVIDER_MODULE` you'll get: `No PKCS#11 module specified.`

### Provider + URI Example
```
openssl storeutl -provider pkcs11 -provider default 'pkcs11:token=testtoken'
```

## 4. SoftHSM2 (If you need a software token)
No prebuilt i686 package in MSYS2 currently; build from source (OpenSSL backend):
```
pacman -S --needed mingw-w64-i686-cmake mingw-w64-i686-sqlite3 mingw-w64-i686-pkgconf
cd /d/Github
git clone https://github.com/opendnssec/SoftHSMv2.git softhsm2-src
cd softhsm2-src
mkdir build32 && cd build32
cmake -G Ninja -DCMAKE_BUILD_TYPE=Release -DCMAKE_INSTALL_PREFIX=/mingw32 \
  -DBUILD_SHARED_LIBS=ON -DSOFTHSM2_WITH_BOTAN=OFF -DSOFTHSM2_WITH_OPENSSL=ON ..
ninja && ninja install
export PKCS11_PROVIDER_MODULE=/mingw32/bin/libsofthsm2.dll
```
Initialize token:
```
mkdir -p ~/softhsm2/tokens ~/.config/softhsm2
cat > ~/.config/softhsm2/softhsm2.conf <<'EOF'
directories.tokendir = /home/$USER/softhsm2/tokens
objectstore.backend = file
slots.removable = true
EOF
softhsm2-util --init-token --free --label testtoken --so-pin 0000 --pin 1234
softhsm2-util --gen-keypair --token testtoken --label rsa1 --id 01 --pin 1234 --key-type RSA:2048
```
Check keys:
```
openssl storeutl -provider pkcs11 -provider default -text 'pkcs11:token=testtoken'
```

## 5. Signing Example
```
echo data > msg.txt
openssl pkeyutl -sign -provider pkcs11 -provider default \
  -inkey 'pkcs11:token=testtoken;object=rsa1;type=private;pin-value=1234' \
  -pkeyopt digest:sha256 -in msg.txt -out sig.bin
openssl pkeyutl -verify -provider pkcs11 -provider default \
  -pubin -inkey 'pkcs11:token=testtoken;object=rsa1;type=public' \
  -pkeyopt digest:sha256 -in msg.txt -sigfile sig.bin
```

## 6. Configuration File Integration (optional)
Append to an `openssl.cnf` copy:
```
[pkcs11_sect]
module = /mingw32/lib/ossl-modules/pkcs11.dll
pkcs11-module-path = /path/to/vendorpkcs11.dll
activate = 1
```
Set:
```
export OPENSSL_CONF=/path/to/custom_openssl.cnf
```

## 7. Packaging
Provide:
```
pkcs11.dll
LICENSE-Apache-2.0.txt
README-win32.md
```
And optionally a SHA256 sum:
```
sha256sum pkcs11.dll
```

## 8. Troubleshooting
| Symptom | Cause | Fix |
|---------|-------|-----|
| No PKCS#11 module specified | Missing driver path | Set `PKCS11_PROVIDER_MODULE` |
| Provider not listed | OPENSSL_MODULES unset | `export OPENSSL_MODULES=/mingw32/lib/ossl-modules` |
| Bad architecture | Mixed 32/64 bit | Rebuild driver & provider same arch |
| C_GetInterface missing | Older token driver | Fallback auto (debug log if needed) |

Enable provider debug:
```
export PKCS11_PROVIDER_DEBUG=file:/tmp/p11prov.log,level:2
openssl list -providers -provider pkcs11 -provider default
cat /tmp/p11prov.log
```

## 9. Notes
- All changes are portability only; crypto logic unmodified.
- Keep vendor PKCS#11 driver 32-bit.
- For upstream sync later, see `FORK_NOTES.md`.
