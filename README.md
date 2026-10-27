# VYRE Audio

**Audyssey Calibration Optimization Tool**

VYRE Audio is a standalone HTML tool for working with Audyssey `.ady` calibration files, using Room EQ Wizard (REW) as part of the optimization workflow.

The project originated from **A1Evo MJ Custom (MJC)** by Navid, which is itself based on the original A1Evo work by **OCA**. VYRE Audio has since been modified and extended with its own development direction, including changes to the optimization workflow, room-mode correction, tuning options, reporting, and user interface.

VYRE Audio is a separate project and is **not an official release of MJC**. The original authors' work is acknowledged below.

## Credits & Original Projects

### Navid — A1Evo MJ Custom (MJC)

MJC provided the direct foundation for VYRE Audio. Thank you to Navid for his work on MJC and for giving the go-ahead to publish VYRE Audio as a further-developed project rather than simply rebranding MJC.

- **AVS Forum:** https://www.avsforum.com/threads/a1evo-mj-custom-mjc.3325897/
- **GitHub:** https://github.com/navid0308/A1Evo_MJ_Custom

### OCA — Original A1Evo

MJC builds on OCA's original A1Evo work. Thank you to OCA for creating and sharing the project that formed an important part of this tool's history.

- **YouTube:** https://www.youtube.com/watch?v=lmZ5yV1-wMI
- **Google Drive:** https://drive.google.com/drive/folders/1rfwzbnET3wYoSSgo-4LDRpKHVXwCYoMF

## Main Features

- Audyssey `.ady` calibration workflow with REW API integration.
- Configurable target curves and support for custom target curves.
- Dedicated room-mode correction controls.
- Analytical Tuning mode.
- Crossover and subwoofer-related optimization options.
- Progress information, detailed logging, response curves, and an HTML report.
- Standalone HTML interface; no conventional installer is required.

## Requirements

- An Audyssey `.ady` calibration file.
- Room EQ Wizard (REW), running with its API server enabled.
- A desktop browser that supports the application.

In REW, open **Preferences → View** and set **Maximum Measurements** to **1000**.

## Basic Workflow

1. Start REW and enable its API server.
2. Open the VYRE Audio HTML file in a desktop browser.
3. Load the Audyssey `.ady` file.
4. Choose the desired target curve, microphone calibration, and other settings.
5. Run the optimization and review the log and resulting curves.
6. Review the generated calibration file and settings before using it with your receiver.

Keep an untouched copy of your original `.ady` file. Read REW measurement cleanup prompts carefully and save any existing REW measurements you want to retain.

## Distribution & License

VYRE Audio is provided free of charge. No donations are requested.

This project builds upon existing upstream work. Please retain all copyright and license notices included with the distributed files, and comply with the applicable license terms for the source material and any redistributed versions. Refer to the license notices included with the application for the applicable terms.

## Feedback

Feedback and bug reports are welcome. When reporting an issue, please include the VYRE Audio version, REW version, a description of the steps that led to the issue, and the relevant part of the optimization log. Remove any personal information before sharing logs or screenshots publicly.
