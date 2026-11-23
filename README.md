# VYRE Audio 2.0.0

VYRE Audio Desktop is a calibration application with an independent native DSP engine. It works from Audyssey measurement data to calculate equalization, levels, distances, crossover settings and subwoofer optimization, then exports an Audyssey calibration file. Supported workflows include multi-subwoofer configurations.

Normal native calibration does not require a separate REW installation.

## Downloads

Installers are distributed through the repository's **GitHub Releases** page. The final repository address has not yet been set; the release-page link will be added here when it is available.

- **Windows x64:** `VYRE Audio Setup 2.0.0.exe`
- **macOS ARM64 / Apple Silicon:** `VYRE Audio-2.0.0-arm64.dmg`

## Installation

On Windows, download and run the `.exe` installer. On an Apple Silicon Mac, open the `.dmg` and install the application from the disk image. The desktop application includes its runtime; Node.js, npm and other development tools do not need to be installed separately.

The current installers are not digitally signed. Windows and macOS may display security warnings when opening unsigned software.

## Community extensions

VYRE Audio supports community themes and plugins. The **Community Theme SDK** styles and arranges supported host controls. The **Community Plugin SDK** provides sandboxed capabilities, including the Universal Extension API for supported native integration points. The independent **Engine API 2** lets a plugin provide its own calibration engine and export workflow.

- [Theme SDK documentation](VYRE_THEME_SDK.md)
- [Plugin and Engine SDK documentation](VYRE_PLUGIN_ENGINE_SDK.md)

Third-party extensions run under the application's existing permission and validation rules. Only install extensions from sources you trust.

## License and copyright

Copyright © 2026 Emelie / EmiCoreV. All rights reserved.

VYRE Audio is free to use and is proprietary software. Providing the application free of charge does not make it open source. No additional rights to redistribute, modify or publish the proprietary source code are granted without express permission. Third-party components are subject to their respective licenses; see [Third-Party Notices](THIRD_PARTY_NOTICES.md).
