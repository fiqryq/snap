---
name: snap-swiftui
description: Launches an autonomous, pixel-perfect SwiftUI implementation loop using Figma MCP and the iOS Simulator.
allowed-tools: [Bash, Read, Glob, Grep, Edit, Write]
---

# /snap-swiftui: The Pixel-Perfect SwiftUI Loop

User provides a Figma link. You implement it pixel-perfect in SwiftUI. That's it.

> Tool names may vary based on your MCP configuration (e.g., `figma__get_screenshot` or `mcp__figma__get_screenshot`). Run `/mcp` to see available tools. If XcodeBuildMCP is connected, prefer its build/run/screenshot/`describe_ui` tools over raw `xcodebuild`/`simctl`.

## Resource Strategy

### Tool Costs

| Tool | Cost | Use For |
|------|------|---------|
| `get_metadata` | **Cheap** | Node tree with child IDs, positions, sizes. No styling. |
| `get_variable_defs` | **Cheap** | All design tokens (colors, spacing, radii) as name→value. |
| `get_code_connect_map` | **Cheap** | Check which Figma nodes map to existing SwiftUI views. |
| `describe_ui` (XcodeBuildMCP) | **Cheap** | Accessibility tree with on-screen frames in points. Zero image cost. |
| `get_screenshot` | **Medium** | Visual image of a specific node. Target a `nodeId` to crop/zoom. |
| Simulator screenshot | **Medium** | Rendered result. Always downscale before reading (see below). |
| `get_design_context` | **Expensive** | Full code + styling + assets. NEVER call on a large parent — call on individual sections. |

**Core rule**: Cheap calls first to build a map, then expensive calls only on the smallest necessary nodes.

### Image Budget

Claude API limits images to 2000px per dimension when >20 images are in a conversation. Old images cannot be individually removed.

- **~20 screenshots per conversation** before the resolution limit kicks in
- **3-image working set**: Figma layout (overview), Figma detail (current section), Simulator state (rendered result)
- **Never re-screenshot a static Figma node** — designs don't change mid-session
- **Simulator screenshots are @3x** (e.g. 1179×2556). Downscale to the Figma frame's point size before reading: `sips -Z 852 shot.png` — this also makes 1 image px = 1 Figma px
- **Use `describe_ui` and code audit for numerical properties** — zero image cost
- **Recovery**: `/compact` drops old images from context (see Recovery section)

---

## Phase 1: Reconnaissance

When the user provides a Figma link, extract `fileKey` and `nodeId` from the URL and immediately start working. If any MCP call, build, or simulator interaction fails, run the [Doctor Check](#doctor-check) to diagnose and fix, then resume.

### Step 1: Get the Node Tree

Call `get_metadata(nodeId, fileKey)`:

```xml
<Frame id="45:1" name="Home" type="FRAME" x="0" y="0" width="393" height="852">
  <Frame id="45:10" name="NavBar" x="0" y="59" width="393" height="44"/>
  <Frame id="45:15" name="Cards" x="16" y="119" width="361" height="400">
    <Text id="45:16" name="Title" x="0" y="0" width="200" height="28"/>
  </Frame>
</Frame>
```

Build a **mental map**:
- Identify every major section and note each `nodeId`
- Note positions and sizes to understand the layout (stacks, grids, scroll areas)
- Identify components vs frames vs text vs icons
- Note the **root frame size** — it tells you which simulator to use (393×852 → iPhone 15/16 Pro, 402×874 → iPhone 17 Pro, 375×812 → iPhone 13 mini). Figma px on an iPhone frame = SwiftUI points.
- Identify system chrome drawn in the frame (status bar, home indicator, nav bar) — don't reimplement what iOS draws for you

### Step 2: Extract All Design Tokens (Once)

Call `get_variable_defs(nodeId, fileKey)`. Save the result — do NOT call this again.

### Step 3: Check Existing Components

Call `get_code_connect_map(nodeId, fileKey)`. For matched views, follow this priority:
1. **Reuse as-is** — if it covers the Figma design exactly
2. **Extend minimally** — add a parameter, style, or variant if close but not exact
3. **Compose** — combine existing views
4. **Create new** — only if nothing existing fits

If Code Connect is empty, grep the project for existing views/styles (`ButtonStyle`, `ViewModifier`, `*Card*`, `*Row*`) before creating anything.

### Step 4: Visual Overview

Call `get_screenshot(nodeId, fileKey)` on the root selection. This is Image 1 of your budget — your layout reference. Do NOT retake it.

**At this point you have NOT called `get_design_context` at all.** You have a complete structural map, all tokens, reusable component info, and a visual reference — all from cheap calls.

---

## Phase 2: Study & Implement (Code in the Dark)

Study the design deeply, memorize every detail, then code from memory.

### Step 1: Layout Shell

Using the metadata tree, implement the outer layout:
- Root container: `NavigationStack`, `ScrollView`, `TabView` as the design implies
- Major section placement with `VStack`/`HStack`/`ZStack`/`Grid`/`LazyVStack`
- Background colors from tokens, `.ignoresSafeArea()` only where the design bleeds edge-to-edge
- Dividers and borders

The metadata + visual overview are usually sufficient. Only call `get_design_context` if you need specific properties you can't infer.

### Step 2: Study Every Section

For each major section:

1. **Figma Detail**: `get_screenshot(sectionNodeId, fileKey)` — zoomed-in visual reference.

2. **Design Context**: `get_design_context(sectionNodeId, fileKey)` — text only, no image cost. If truncated, use child node IDs from `get_metadata` and fetch children individually. The output is usually web code — read it as a spec, not as code to port.

3. **Absorb every detail**: fonts, sizes, weights, colors, spacing, corner radii, borders, shadows, icon shapes. Burn it into memory.

4. **Color sanity check**: Compare `get_design_context` colors against the Figma screenshot. If the screenshot reveals opacity layering, overlapping fills, blur/material, or gradients — the raw token values will be wrong. Use the visual truth, not the raw token.

**No simulator screenshots during this phase.** You are studying, not checking.

**Key rule**: NEVER call `get_design_context` on the root selection. Always target the smallest meaningful node.

### Step 3: Code from Memory

Implement everything using what you memorized:
- Design context output for exact properties
- Tokens from Phase 1 (do not re-extract)
- Reusable views from Phase 1
- **Respect project rules**: Check for `.claude/rules`, `CLAUDE.md`, `AGENTS.md`, and project instruction files. Follow established patterns for view structure, file placement, naming, state management (`@Observable`, view models, etc.), and styling. Translate the design into the project's conventions.
- **Add a `#Preview`** for every new view, matching the project's preview style.

**UI only.** Don't touch networking, persistence, or business logic unless the user explicitly asks. Use mock/sample data if models or services don't exist yet.

**Minimize screenshots during coding.** You studied the design — use what you memorized. But if you're missing crucial data for a specific element (exact icon shape, a nested layout you didn't drill into, a subtle gradient), take a targeted `get_screenshot` on that Figma node rather than guessing.

### Step 4: Design System Sync

NEVER hardcode values in views. Sync to the project's design system:

- **Colors**: Add a Color Set to `Assets.xcassets` (with dark variant if the design has one) or extend the project's `Color`/`ShapeStyle` extension. Never `Color(hex: "#F3F3F3")` or `Color(red:green:blue:)` inline in a view.
- **Typography**: Use the project's font extension / text styles. Custom fonts must be in the target and registered under `UIAppFonts` in Info.plist. Prefer `Font.custom(_:size:relativeTo:)` so Dynamic Type scales.
- **Spacing & radii**: Add to the project's spacing/radius constants (enum or extension). Never scatter magic numbers.
- **Component library**: If the project has a design-system module/package, add tokens there, not in the feature.

### Step 5: Icons & Assets

**Icons**: Find a match first — project asset catalog by name, then **SF Symbols** by visual shape (match weight with `.fontWeight`/`.imageScale` and size with `.font(.system(size:))`). Then check the project's existing icon set.

**Fallback**: If nothing matches, export the icon from Figma as SVG/PDF, add it to `Assets.xcassets` as a vector image with **Preserve Vector Data** and **Render As: Template** (for tintable icons), and use `Image("name")`.

**Images/illustrations**: Download from the Figma MCP assets endpoint and add to `Assets.xcassets` with @2x/@3x or a single PDF/SVG. Don't draw complex illustrations with `Path`. Don't add new Swift packages without asking.

### Property Checklist

Before writing code for ANY element, verify ALL applicable properties:

- **Text**: font family, size, weight, line height, tracking, color, opacity, alignment (`.multilineTextAlignment`), line limit, truncation, text case, underline/strikethrough
- **Container**: frame (prefer flexible `maxWidth: .infinity` over fixed), padding (all 4 edges), background, corner radius (`RoundedRectangle(cornerRadius:style: .continuous)` — Figma corner smoothing ≈ `.continuous`), border (`.overlay` + `.strokeBorder`), shadow, opacity, clipping
- **Icon**: size, rendering mode, foreground color (independent from parent text), symbol weight
- **Button**: all text + container props + pressed/disabled states — implement via `ButtonStyle`, not inline gestures
- **Image**: frame, `.resizable()`, `.scaledToFill/Fit`, `.clipShape`, aspect ratio
- **Spacing**: stack `spacing:` (always explicit — the SwiftUI default is NOT 0), grid spacing

**Figma → SwiftUI conversions:**

| Figma | SwiftUI |
|-------|---------|
| Line height `L` for font size `S` | `.lineSpacing(L - UIFont(name/size).lineHeight)` plus half the difference as vertical padding if the box height matters |
| Letter spacing `X%` | `.tracking(S * X / 100)` |
| Letter spacing `Xpx` | `.tracking(X)` |
| Drop shadow blur `B`, offset `x,y` | `.shadow(color:radius: B / 2, x:, y:)` — spread has no direct equivalent; fake with padding/second shape |
| Auto layout gap | stack `spacing:` |
| Auto layout "space between" | `Spacer()` between children |
| Fill container / Hug contents | `.frame(maxWidth: .infinity)` / no frame |
| Background blur | `.background(.ultraThinMaterial)` etc. — match visually |

**Layout Principle**: Avoid fixed frames. With correct font, padding, spacing, and flexible frames, views size themselves. Fixed `.frame(width:height:)` on text containers is a symptom of wrong upstream layout.

**Adaptive**: Respect safe areas, keep layouts working on smaller iPhones, and check Dynamic Type doesn't break the design at default size. If the design has a dark variant, support it via asset catalog appearances.

---

## Phase 3: Refinement Loop

You think you're done. Now prove it. Keep cycling until the result is perfect.

### Getting the Screen on the Simulator

- Build and run on a simulator whose size matches the Figma frame (XcodeBuildMCP `build_run_sim`, or `xcodebuild -scheme <S> -destination 'platform=iOS Simulator,name=<Device>' build` then `xcrun simctl install booted <App.app>` + `xcrun simctl launch booted <bundleId>`).
- If the screen isn't the launch screen: use an existing deep link (`xcrun simctl openurl booted <url>`) or navigate via XcodeBuildMCP UI taps. As a last resort, add a **temporary** debug launch hook (e.g. a `-snapScreen` launch argument checked in `#if DEBUG`) and **remove it in Phase 4**.
- Match appearance to the design: `xcrun simctl ui booted appearance light|dark`.
- Clean status bar for comparison: `xcrun simctl status_bar booted override --time 9:41 --batteryState charged --batteryLevel 100`.

### The Loop

**Step 1: Simulator screenshot (1 image)**

```bash
xcrun simctl io booted screenshot /tmp/snap.png && sips -Z <figmaFrameHeight> /tmp/snap.png
```

Read it and compare against the Figma screenshots in context. Be extremely picky:
- Visual alignment issues
- Missing elements or wrong proportions
- Spacing that looks off, wrong safe-area handling
- Wrong icon shapes or symbol weights
- Color or font weight mismatches
- **Color comparison**: Sample dominant colors from Figma vs Simulator. Flag anything that looks off — opacity, materials, and color-space (sRGB vs Display P3) can cause perceived differences even when hex values match.

**Step 2: Numerical audit (zero images)**

1. **Frame audit**: Add `.accessibilityIdentifier("snap.<name>")` to key views (keep them if the project uses identifiers, otherwise remove in Phase 4). Call XcodeBuildMCP `describe_ui` and compare each element's frame (x, y, width, height in points) against `get_metadata` positions.
2. **Code audit**: For every element, walk the Property Checklist and compare the literal values in code (after resolving tokens) against `get_design_context`.

12pt is not 10pt. #F97316 is not #FF611C.

**Tint check**: Icons inside `Button`/`NavigationLink`/`Label` inherit the accent tint. If an icon's color doesn't match, set `.foregroundStyle` on the image itself or use `.buttonStyle(.plain)` / a custom `ButtonStyle`.

**Step 3: Picky mismatch list**

Combine visual + numerical issues into one list. No issue is too small. 2pt off? List it.

**Step 4: Fix everything**

Batch all fixes. One rebuild, no screenshots between individual fixes.

**Step 5: Repeat from Step 1**

**Exit condition**: Retake `get_screenshot` on the Figma root node. Place side-by-side with your latest simulator screenshot. Compare colors directly. Only move on when both screenshots match AND the numerical audit shows zero mismatches.

### Icon & Shape Verification

If an icon looks wrong during any loop iteration:
- `get_screenshot(iconNodeId, fileKey)` on the Figma icon
- Crop the simulator icon at full @3x resolution (before downscaling) for a zoomed view
- Compare the SILHOUETTE — stroke count, shape, proportions, fill vs outline

Common mismatches:
- SF Symbol `.fill` vs outline variant
- Symbol weight (regular vs medium vs semibold) against Figma stroke width
- "filter" → `line.3.horizontal.decrease` vs `slider.horizontal.3`
- "more" → `ellipsis` vs `ellipsis.circle`

If it doesn't match: try another symbol, or export the Figma vector into the asset catalog.

### Anti-Pattern: "Looks Good Enough"

This ALWAYS misses: 2-4pt spacing differences, default stack spacing, wrong symbol variant, slightly wrong color, missing shadow, font weight mismatch, line height ignored.

**RULE**: Not done until zero visual mismatches AND zero numerical mismatches.

---

## Recovery: Compact & Resume

If you hit the image limit (or proactively after ~15 screenshots):

1. **Run `/compact`** — summarizes conversation, drops old images. Notes and progress are preserved.
2. **Retake 3 reference images**: Figma overview, Figma detail (current section), Simulator state.
3. **Continue the refinement loop** — you still know everything from the compacted history. ~17 more image slots available.

Repeatable: compact → retake 3 → continue → compact again if needed.

---

## Phase 4: Cleanup & User Review

**Cleanup first**:
- Remove any temporary debug launch hooks and `snap.*` accessibility identifiers you added
- Clear the status bar override: `xcrun simctl status_bar booted clear`
- Make sure the project still builds

**Then stop and ask**:
- "Here's the final result. Are you happy with it?"
- "If something looks off, paste a Figma 'Link to Selection' for the specific area."

If the user provides a new link, re-run Phases 1-3 scoped to that selection.

---

## Doctor Check

Run this ONLY when something fails. Not at startup.

**Figma MCP not working?**
- Call `whoami` to check authentication
- Not connected → alert user to configure Figma MCP
- Not authenticated → guide through Figma OAuth

**Build failing or no project found?**
- Find the project: `*.xcworkspace` (prefer it), `*.xcodeproj`, or `Package.swift`; Tuist (`Project.swift`) → `tuist generate` first
- List schemes: `xcodebuild -list -workspace <W>` (or `-project <P>`)
- Read the compiler errors and fix them before continuing the loop

**Simulator not responding?**
- `xcrun simctl list devices available` — pick the device matching the Figma frame
- `xcrun simctl boot "<Device>" && open -a Simulator`
- Find the bundle ID: `PRODUCT_BUNDLE_IDENTIFIER` from `xcodebuild -showBuildSettings`

**Unknown design system or icon set?**
- Grep for `extension Color`, `extension ShapeStyle`, `extension Font`, `enum Spacing`, `Theme`, `DesignSystem`
- Check `Assets.xcassets` for Color Sets and icon image sets
- Check `Package.swift` / SPM dependencies for design-system or icon packages

---

## Examples

**Invocation:**
```
/snap-swiftui
> Paste Figma link: https://figma.com/design/abc123/MyApp?node-id=42-100
```

**What happens:**
1. Recon: `get_metadata` + `get_variable_defs` + `get_code_connect_map` (0 images). Frame is 393×852 → iPhone 16 Pro.
2. Figma overview: `get_screenshot` on root (image 1)
3. Study section 1: `get_screenshot` (image 2) + `get_design_context` (text). Memorize.
4. Study section 2: `get_screenshot` (image 3) + `get_design_context` (text). Memorize.
5. Code everything from memory, add colors to the asset catalog (0 images)
6. **Loop round 1**: build + simulator screenshot (image 4) + `describe_ui` audit → 4 mismatches + 1 wrong symbol
7. Fix mismatches. Icon check: `get_screenshot` on Figma icon (image 5) — swap to `.fill` variant.
8. **Loop round 2**: simulator screenshot (image 6) + audit → default `VStack` spacing left in one place
9. Fix spacing.
10. **Loop round 3**: simulator screenshot (image 7) + audit → zero mismatches.
11. **Final check**: Retake Figma `get_screenshot` (image 8) side-by-side with simulator → match.
12. Cleanup, ask for review. Total: 8 images.

**Bad patterns:**
- Screenshotting every element individually (budget killer)
- Reading raw @3x simulator screenshots (wastes resolution budget, breaks 1:1 comparison)
- Re-screenshotting the same Figma node (it's static)
- Rebuilding and screenshotting after every small tweak
- Porting `get_design_context` web code literally into SwiftUI
- `get_design_context` on root (token waste)
- `get_variable_defs` more than once (redundant)
- `Color(hex:)` or `Color(red:green:blue:)` inline in views
- Fixed `.frame(width: 247)` on content that should size itself
- Relying on default stack spacing
- Reimplementing the status bar or home indicator drawn in the Figma frame
- Saying "looks close" without numerical verification
