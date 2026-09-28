# Snap

Build native mobile UI on autopilot. Pixel-perfect Figma-to-SwiftUI and Figma-to-Compose with autonomous refinement.

Figma recon → study → build → refine loop, running on the iOS Simulator or Android Emulator.

## Skills

| Skill | Platform | Renders on |
|-------|----------|------------|
| `/snap-swiftui` | SwiftUI (iOS) | iOS Simulator |
| `/snap-compose` | Jetpack Compose (Android) | Android Emulator |

## Install

```bash
git clone <repo-url> ~/.claude/plugins/snap
```

## Usage

Open Claude Code in your iOS or Android project and run:

```
/snap-swiftui
```

or

```
/snap-compose
```

Then paste a Figma "Link to Selection".

## How it works

1. **Recon**: reads the Figma node tree, tokens, and Code Connect mappings with cheap calls, then takes one overview screenshot
2. **Build**: studies each section and implements it with your project's theme, components, and conventions
3. **Refine**: builds and runs on a simulator/emulator sized to the Figma frame, screenshots it, audits frames numerically (`describe_ui` / `uiautomator dump`), and fixes mismatches until nothing is off
4. **Cleanup**: removes temporary debug hooks and simulator/emulator overrides, then asks for your review

## What it handles

- Picks a simulator/emulator that matches the Figma frame size (1 Figma px = 1pt / 1dp)
- Finds your design system (asset catalog colors, `Color`/`Font` extensions, Compose `MaterialTheme`, custom `CompositionLocal`s)
- Reuses existing views/composables via Code Connect and project search
- Maps icons to SF Symbols, Material Icons, or your own set, and falls back to exported vectors
- Converts Figma line height, letter spacing, and shadows to their SwiftUI/Compose equivalents
- Keeps the image budget low by downscaling device screenshots and auditing numbers without images

## Requirements

- **Claude Code**
- **Figma MCP** configured ([setup guide](https://help.figma.com/hc/en-us/articles/32132100833559))
- **iOS**: Xcode + iOS Simulator. [XcodeBuildMCP](https://github.com/cameroncooke/XcodeBuildMCP) recommended for `describe_ui`
- **Android**: Android SDK with `adb` and an emulator (AVD), JDK (Android Studio's JBR works)
