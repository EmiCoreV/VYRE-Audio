# Third-Party Notices

VYRE Audio is proprietary software. The notices below apply to third-party components and do not grant an open-source license to VYRE Audio itself.

## Electron and bundled runtime components

The Windows x64 and macOS ARM64 2.0.0 installer archives contain Electron's MIT license and the bundled components' notices:

- Windows installed application directory: `LICENSE.electron.txt` and `LICENSES.chromium.html`.
- macOS application bundle: `VYRE Audio.app/Contents/Resources/LICENSE.electron.txt` and `LICENSES.chromium.html`.

These files contain the original notices and terms supplied with Electron, including notices for Chromium, Node.js, V8 and other bundled components. Preserve them when distributing the application; the individual entries may have different conditions. The files were observed inside both release archives. Their presence is not a determination that all redistribution conditions have been satisfied.

## Windows installer

The Windows installer uses NSIS and contains installer components including `System.dll`, `StdUtils.dll`, `SpiderBanner.dll`, `nsExec.dll`, `nsis7z.dll` and `WinShell.dll`. The unmodified [NSIS COPYING file](third-party/NSIS-COPYING.txt) supplied with the NSIS build tools describes its zlib/libpng, bzip2 and Common Public License 1.0 terms and exceptions. Its inclusion does not establish which compression modules are in the installer or establish the licenses of every additional plugin. The exact provenance, license and applicable redistribution notices for the bundled installer plugins and compression components remain to be verified before public distribution.

## Images, fonts and other assets

The built-in Neon Mirage and GTA VI Special themes contain coastal image assets. Their origin, licenses and redistribution permissions were not established from the available project files and must be verified before publication. Verify provenance and redistribution rights for the VYRE artwork and any other bundled third-party media. The built-in theme folders do not declare packaged WOFF2 fonts; community themes and separately distributed examples may contain fonts or other assets with their own terms.

## Scope and release review

Electron Builder and Playwright are development dependencies in the desktop package configuration; they are not automatically treated as installed application dependencies. The Windows installer itself contains the NSIS components noted above. The repository and the available files do not establish a complete third-party inventory, nor whether source-code or other obligations attach to any particular installer component. Complete that review and include any additional required notices and corresponding materials before public distribution.
