# VYRE Audio

VYRE Audio is a free room calibration application with an independent native DSP engine, designed for Denon and Marantz AV receivers with Audyssey support.

Using acoustic measurement data, VYRE calculates equalization, speaker levels, distances, crossover settings and multi-subwoofer optimization before exporting an Audyssey calibration file.

Native calibration does not require a separate REW installation.

## Features

- Independent native DSP and calibration engine.
- Advanced multi-subwoofer optimization.
- Standard, SWMIX and Directional Bass support.
- Automatic crossover and polarity optimization.
- Custom target curves and manual PEQ adjustments.
- Target Curve Studio.
- Calibration presets and detailed response visualization.
- Automatic update notifications.
- Customizable user interface with multiple built-in themes.
- Community plugin, theme and engine support.

## Downloads

Download the latest version from the [GitHub Releases page](https://github.com/EmiCoreV/VYRE-Audio/releases/latest).

Supported platforms:

- **Windows (x64)**
- **macOS (Apple Silicon / ARM64)**
- **Linux (x64) - Experimental**
- **Linux (ARM64) - Experimental**

## Installation

**Windows:** Download and run the EXE installer.

**macOS:** Open the DMG file and move VYRE Audio to your Applications folder.

**Linux:** Extract the TAR.GZ archive and launch the application. Linux builds are currently experimental.

All builds include the required application runtime. Node.js, npm and other development tools do not need to be installed separately.

The Windows installer is unsigned. The macOS application is ad-hoc signed but not Apple-notarized. Operating system security warnings may therefore appear.

## Community Extensions

VYRE Audio provides dedicated SDKs for community developers.

- **Community Theme SDK:** Create custom themes and personalize the VYRE Audio interface.
- **Community Plugin SDK:** Extend native calibration functionality, integrate external measurement sources, provide custom target curves and add new features through the Universal Extension API.
- **Engine API 2:** Develop independent calibration engines and export workflows.

Documentation:

- [Theme SDK Documentation](VYRE_THEME_SDK.md)
- [Plugin and Engine SDK Documentation](VYRE_PLUGIN_ENGINE_SDK.md)

Third-party extensions operate within the application's permission and validation rules. Only install extensions from sources you trust.

## License and Copyright

Copyright © 2026 Emelie / EmiCoreV. All rights reserved.

VYRE Audio is free to use and is proprietary software. Providing the application free of charge does not make it open source.

No additional rights to redistribute, modify or publish the proprietary source code are granted without express permission.

Third-party components are subject to their respective licenses. See the [Third-Party Notices](THIRD_PARTY_NOTICES.md) and [VYRE License Notice](VYRE-LICENSE-NOTICE.md) for further information.

Corresponding third-party source packages are provided alongside the applicable releases.
