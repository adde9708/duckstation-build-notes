---
title: "Building DuckStation locally on Arch Linux and NixOS"
description: "How to build DuckStation on Arch Linux and NixOS by patching the build system locally without distributing patched source files."
tags: [linux, arch, nixos, duckstation, cmake, packaging]
---

# Building DuckStation locally on Arch Linux and NixOS

DuckStation is an excellent PlayStation 1 emulator. Upstream intentionally
restricts building on some Linux distributions, including Arch Linux and NixOS,
as part of a policy to avoid unsupported packaging environments.

If you’re a power user on Arch or NixOS, you may still wish to build DuckStation
locally. This guide shows how to apply minimal build-system adjustments during
your own package builds, without distributing patched source code or modified
build scripts.

The goal is to respect upstream’s preferences while allowing advanced users to
build DuckStation in unsupported environments.

---

## Why Does DuckStation Block Arch and NixOS?

DuckStation’s CMake build system reads `/etc/os-release` and some environment
variables to detect distros. If it finds Arch, NixOS, or related identifiers,
it will abort the build with a fatal error:

```cmake
if(EXISTS /etc/os-release)
  file(READ /etc/os-release OS_RELEASE_CONTENT)
  if(OS_RELEASE_CONTENT MATCHES "ID=arch" OR
     OS_RELEASE_CONTENT MATCHES "ID_LIKE=arch" OR
     OS_RELEASE_CONTENT MATCHES "ID=nixos")
    message(FATAL_ERROR "Unsupported environment.")
  endif()
endif()
if(DEFINED ENV{NIX_BUILD_TOP} OR DEFINED ENV{NIX_STORE} OR
   DEFINED ENV{IN_NIX_SHELL} OR EXISTS "/etc/NIXOS")
  message(FATAL_ERROR "Unsupported environment.")
endif()
```

The upstream author requests **no distribution of patched build
scripts or packaging overlays** that bypass this block. This is to avoid
dealing with packaging bugs and conflicts that arise from downstream
modifications.

---

## How to Patch Locally During Build Without Distributing Patched Files

The key principle:
**patch or modify the source *inside* your package build process, without
distributing modified source or build files.**

You do this by embedding patch commands directly into your build scripts.

---

### Arch Linux: PKGBUILD Example

In Arch, your `PKGBUILD` file can include a `prepare()` function that
modifies source files before building.

Here’s a minimal example that changes the fatal error to a warning, allowing
the build to continue:

```bash
prepare() {
  cd "$srcdir/$pkgname"
  sed -i 's/message(FATAL_ERROR/message(WARNING)/' \
    CMakeModules/DuckStationBuildSummary.cmake
}
```

This means:

* No patch files are distributed separately.
* The upstream source tarball remains unchanged; modifications are applied only
  inside your local build directory.
* Your PKGBUILD remains local so you avoid violating
  upstream's redistribution policy.

---

### NixOS: nixpkgs Overlay Example

In Nix, you override the package derivation to patch the source inline.

Here’s an example snippet you can add in your overlay:

```nix
postPatch = ''
  substituteInPlace CMakeLists.txt \
    --replace 'message(FATAL_ERROR "Unsupported environment.")' \
              'message(WARNING "Unsupported environment.")'
'';
```

This performs an in-place substitution during the build process, disabling
the fatal error without changing the distributed source.

---

## How to Build After Applying Local Patch

After patching in your build script, configure the build as usual.

If you add the opt-in flag in the source, remember to enable it:

```bash
cmake -DALLOW_UNSUPPORTED_ENVIRONMENT=ON ..
```

Otherwise, the `sed` or substitution commands above simply prevent the fatal
error and let the build continue.

---

## Why This Approach Matters

* **Respecting Licensing and Upstream Wishes:** You never distribute
  modified build files or patches, only instructions embedded in your
  packaging scripts.
* **Easy to Maintain:** When DuckStation updates, just update your
  packaging scripts accordingly.
* **Clear Boundaries:** Users know that builds on Arch or NixOS are
  unsupported and come with risks.

---

> **Note:** Builds produced using these methods are unsupported by upstream.
> DuckStation officially supports Ubuntu, Debian, Fedora, and Windows.
> If you encounter issues, please reproduce them on a supported distribution before reporting bugs.

## Final Thoughts

DuckStation maintains a strict policy regarding unsupported distributions like
Arch and NixOS. By maintaining local patching within your package build scripts,
you can build and run DuckStation on Arch or NixOS while respecting upstream
boundaries.

If you found this useful, feel free to share the method, not patched files!
Happy emulating!

---
