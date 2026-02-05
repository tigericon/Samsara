# Samsara Android NDK Build

This GitHub Actions workflow automatically builds Samsara LLVM Obfuscator for use with Android NDK.

## Supported Platforms

| Platform | Architecture | File Format |
|----------|--------------|-------------|
| Linux | x86_64 | `.tar.gz` |
| macOS | x86_64 | `.tar.gz` |
| macOS | ARM64 (Apple Silicon) | `.tar.gz` |
| Windows | x86_64 | `.7z` |

## Supported Android Targets

The built compilers support the following Android architectures:

- `AArch64` → `arm64-v8a`
- `ARM` → `armeabi-v7a`
- `X86` → `x86`
- `X86_64` → `x86_64`

## How to Trigger Build

### Automatic (Release)

Create a new release on GitHub, and the workflow will automatically build all platforms.

### Manual

1. Go to **Actions** tab
2. Select **Build Samsara for Android NDK**
3. Click **Run workflow**
4. Configure options:
   - `build_type`: Release / RelWithDebInfo
   - `platforms`: all / linux / macos / windows
   - `upload_release`: Upload to release page

## Download

After the build completes:

1. Go to **Actions** tab
2. Click on the completed workflow run
3. Download artifacts from the **Artifacts** section

Or download from **Releases** page if uploaded.

## Installation

### Replace NDK Clang (Linux Example)

```bash
# Set your NDK path
NDK_PATH="/path/to/android-ndk-r26b"
TOOLCHAIN="${NDK_PATH}/toolchains/llvm/prebuilt/linux-x86_64"

# Backup original files
mkdir -p "${TOOLCHAIN}/backup"
cp -a "${TOOLCHAIN}/bin/clang" "${TOOLCHAIN}/backup/"
cp -a "${TOOLCHAIN}/bin/clang++" "${TOOLCHAIN}/backup/"

# Extract Samsara
tar -xzvf Samsara-21-Android-NDK-linux-x86_64.tar.gz -C /tmp/samsara

# Replace files
cp /tmp/samsara/bin/* "${TOOLCHAIN}/bin/"
cp -r /tmp/samsara/lib/clang/* "${TOOLCHAIN}/lib/clang/"
```

### Windows

```powershell
# Set paths
$NDK_PATH = "C:\Users\xxx\AppData\Local\Android\Sdk\ndk\26.1.10909125"
$TOOLCHAIN = "$NDK_PATH\toolchains\llvm\prebuilt\windows-x86_64"

# Extract and copy
7z x Samsara-21-Android-NDK-windows-x86_64.7z -o"C:\temp\samsara"
Copy-Item "C:\temp\samsara\bin\*" "$TOOLCHAIN\bin\" -Force
Copy-Item "C:\temp\samsara\lib\clang" "$TOOLCHAIN\lib\" -Recurse -Force
```

## Usage in Android Project

### CMakeLists.txt

```cmake
# Enable obfuscation in Release build
if(CMAKE_BUILD_TYPE STREQUAL "Release")
    set(SAMSARA_FLAGS
        "-mllvm" "-irobf-indbr"    # Indirect branches
        "-mllvm" "-irobf-icall"    # Indirect calls
        "-mllvm" "-irobf-indgv"    # Indirect global variables
        "-mllvm" "-irobf-cse"      # String encryption
        "-mllvm" "-irobf-fla"      # Control flow flattening
        "-mllvm" "-irobf-cie"      # Integer constant encryption
        "-mllvm" "-level-indbr=2"  # Obfuscation level (0-3)
        "-mllvm" "-level-icall=2"
    )
    target_compile_options(your_library PRIVATE ${SAMSARA_FLAGS})
endif()
```

### Per-Function Control

```cpp
// Disable obfuscation for this function
[[clang::annotate("-fla -icall")]]
void performance_critical_function() {
    // ...
}

// Enable maximum obfuscation
[[clang::annotate("+indbr +icall +fla ^indbr=3 ^icall=3")]]
void verify_license() {
    // ...
}
```

### JSON Configuration

Create `samsara_config.json`:

```json
{
  "randomSeed": "YourSecretSeed32BytesHere!!!!!!",
  "indbr": { "enable": true, "level": 2 },
  "icall": { "enable": true, "level": 2 },
  "indgv": { "enable": true, "level": 2 },
  "cse": { "enable": true },
  "fla": { "enable": true },
  "cie": { "enable": true, "level": 2 }
}
```

Use in CMake:

```cmake
target_compile_options(your_library PRIVATE
    "-mllvm" "-samsara-cfg=${CMAKE_SOURCE_DIR}/samsara_config.json"
)
```

## Build Time

Approximate build times on GitHub Actions runners:

| Platform | Time |
|----------|------|
| Linux x86_64 | ~60-90 min |
| macOS x86_64 | ~90-120 min |
| macOS ARM64 | ~60-90 min |
| Windows x86_64 | ~120-180 min |

## Package Contents

```
package/
├── bin/
│   ├── clang          # C compiler
│   ├── clang++        # C++ compiler
│   ├── lld            # Linker
│   ├── ld.lld         # ELF linker
│   ├── llvm-ar        # Archive tool
│   ├── llvm-nm        # Symbol table
│   ├── llvm-objcopy   # Object copy
│   ├── llvm-objdump   # Disassembler
│   ├── llvm-strip     # Strip symbols
│   └── ...
├── lib/
│   └── clang/
│       └── 21/        # Clang runtime libraries
│           ├── include/
│           └── lib/
└── VERSION            # Build info
```

## Troubleshooting

### "Obfuscation options not recognized"

Make sure you replaced the correct NDK clang. Check version:

```bash
$NDK_TOOLCHAIN/bin/clang --version
# Should show: clang version 21.x.x ...
```

### Linker errors

Ensure `lld` is also replaced, or add to your build:

```cmake
set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -fuse-ld=lld")
```

### Build fails on specific function

Try excluding that function from obfuscation:

```cpp
[[clang::annotate("-fla -icall -indbr")]]
void problematic_function() { }
```

## License

Apache License 2.0

See the main Samsara repository for full license details.
