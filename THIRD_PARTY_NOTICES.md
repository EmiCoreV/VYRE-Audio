# VYRE Audio 2.1.1 — Third-Party Notices

VYRE Audio is proprietary. Third-party components have their own licenses and copyrights; their licenses do not grant an open-source license to VYRE's proprietary code. These notices describe the four 2.1.1 release artifacts listed in `SHA256SUMS.txt`.

## Electron and bundled components — all platforms

Each application includes Electron 45.5.0 and its distribution of Chromium, Node.js, V8 and other components. Electron's MIT notice is included as [third-party/LICENSE.electron.txt](third-party/LICENSE.electron.txt). The complete platform-specific Electron/Chromium credits, individual copyright notices and license texts are:

- macOS ARM64 and Linux x64/ARM64: [third-party/LICENSES.chromium.html](third-party/LICENSES.chromium.html).
- Windows x64: [third-party/LICENSES.chromium-windows.html](third-party/LICENSES.chromium-windows.html).

The Windows credits differ from the macOS/Linux credits; do not substitute one for the other. The corresponding files are also included in each application distribution: in the Windows installed application directory, in `VYRE Audio.app/Contents/Resources` on macOS, and at the root of each Linux archive. Preserve the original Electron notice and the appropriate complete credits with every distribution. The credits list components and license texts; a mention in that inventory alone does not prove that every listed optional component is present or enabled on every platform.

### FFmpeg source, build configuration and replacement

The Windows app contains `ffmpeg.dll`, the Linux archives contain `libffmpeg.so`, and the macOS Electron Framework contains `libffmpeg.dylib`. FFmpeg is distributed under LGPL 2.1-or-later with the component notices in the supplied source and credits; the examined Chrome configurations set `CONFIG_GPL=0` and `CONFIG_NONFREE=0`. A GitHub-ready [FFmpeg/Electron source and build-reference package](third-party/FFmpeg-Electron-45.5.0/README.md) contains the Chromium FFmpeg source revision, the five Electron patches, a patched source archive, relevant upstream build files and license material. Windows and Linux library hashes match official Electron 45.5.0 files. The macOS code comparison and signing differences are documented in the package.

The distributed FFmpeg libraries are separate shared libraries. A temporary macOS copy was successfully ad-hoc re-signed after replacing the library, but no modified library was run. The exact generated release build arguments and functional replacement on each platform were not independently verified. Source availability, required user modification and debugging rights, and any applicable LGPL relinking conditions must be respected; see the source package and [FFmpeg's guidance](https://ffmpeg.org/legal.html). The checked-in Chrome configurations enable H.264 and AAC, but their presence is not proof of a specific patent claim or licensing obligation in a particular jurisdiction.

## Windows installer

The current Windows x64 EXE is the custom NSIS installer built with NSIS 3.04 (electron-builder toolset 3.0.4.1). Archive inspection found the Unicode standard `nsDialogs.dll` and `System.dll`, the NSIS installer stub and Deflate compression. The [NSIS upstream license notice](third-party/NSIS-COPYING.txt) is supplied; NSIS, these standard plugins and its zlib module are covered by the zlib/libpng terms stated there. The executable also contains a generated NSIS uninstaller and the protected application payload. The inspected archive did not list StdUtils, Nsis7z, WinShell or the earlier `elevate.exe`; previous concerns specific to those components do not describe this installer. [Windows installer notes](WINDOWS-INSTALLER-LICENSES.md) provide the component-specific summary.

**Historical in-app notice:** The already-built application ASAR still contains an older `THIRD_PARTY_NOTICES.md` that describes the previous electron-builder NSIS installer and attributes its plugins to Windows. It is inaccurate for the current Windows EXE. These external release notices correct it; distribute them with the artifacts. The old in-app text cannot be corrected without rebuilding the applications, which was outside this documentation-only update.

## Obfuscation and development dependencies

The protected staging process uses `javascript-obfuscator` 5.9.0 during the build. Its [BSD 2-Clause license](third-party/JAVASCRIPT-OBFUSCATOR-LICENSE.txt) is included in the application ASAR and in this release directory. Electron Builder, Playwright and the obfuscator package are development/build dependencies, not separate production npm dependencies in the shipped ASAR. Their use during development must not be confused with the Electron runtime and NSIS components actually redistributed in the builds.

## Images, fonts, SDKs and community extensions

According to the project owner, the built-in Neon Mirage and Neon Vice coastal background images were generated specifically for VYRE Audio with ChatGPT/OpenAI rather than downloaded from wallpaper collections. A prior visual review found no obvious third-party logo; the Neon Mirage sports car may resemble a vehicle design and visual review cannot establish rights clearance. The Neon Vice background SVG is a separate stylized illustration. OpenAI's [European terms](https://openai.com/policies/eu-terms-of-use/) address rights between the user and OpenAI and do not resolve third-party rights. Review any potentially protected motifs and applicable rights before distribution.

The inspected bundled theme assets contain no font files. The renderer refers to system font families; it does not thereby distribute them. The separate Plugin/Engine and Theme SDK documents describe extension interfaces; inclusion of documentation does not itself grant rights to third-party assets or the proprietary VYRE source. Authors are responsible for licenses and permissions for independently distributed plugins, themes, graphics and fonts. The Ion Atlas and Obsidian Wave example fonts in the development tree are not present in the inspected release ASARs; if those examples are distributed separately, their own licenses must be reviewed and included.

## Release status

Preserve these external notices and the relevant original license files with each download. A complete license clearance has not been established: FFmpeg source and build references are supplied, but complete release-build reproduction and functional library replacement on each target have not been independently verified; any applicable codec or asset rights also require separate assessment. Free distribution does not eliminate applicable obligations.
