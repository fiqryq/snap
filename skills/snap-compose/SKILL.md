---
name: snap-compose
description: Launches an autonomous, pixel-perfect Jetpack Compose implementation loop using Figma MCP and the Android Emulator.
allowed-tools: [Bash, Read, Glob, Grep, Edit, Write]
---

# /snap-compose: The Pixel-Perfect Compose Loop

User provides a Figma link. You implement it pixel-perfect in Jetpack Compose. That's it.

> Tool names may vary based on your MCP configuration (e.g., `figma__get_screenshot` or `mcp__figma__get_screenshot`). Run `/mcp` to see available tools.

## Resource Strategy

### Tool Costs

| Tool | Cost | Use For |
|------|------|---------|
| `get_metadata` | **Cheap** | Node tree with child IDs, positions, sizes. No styling. |
| `get_variable_defs` | **Cheap** | All design tokens (colors, spacing, radii) as name→value. |
| `get_code_connect_map` | **Cheap** | Check which Figma nodes map to existing composables. |
| `uiautomator dump` | **Cheap** | Node tree with on-screen bounds in px. Zero image cost. |
| `get_screenshot` | **Medium** | Visual image of a specific node. Target a `nodeId` to crop/zoom. |
| Emulator screenshot | **Medium** | Rendered result. Always downscale before reading (see below). |
| `get_design_context` | **Expensive** | Full code + styling + assets. NEVER call on a large parent — call on individual sections. |

**Core rule**: Cheap calls first to build a map, then expensive calls only on the smallest necessary nodes.

### Image Budget

Claude API limits images to 2000px per dimension when >20 images are in a conversation. Old images cannot be individually removed.

- **~20 screenshots per conversation** before the resolution limit kicks in
- **3-image working set**: Figma layout (overview), Figma detail (current section), Emulator state (rendered result)
- **Never re-screenshot a static Figma node** — designs don't change mid-session
- **Emulator screenshots are in device px** (e.g. 1080×2400). Downscale to the Figma frame's dp width before reading: `sips --resampleWidth <figmaWidth> shot.png` — this also makes 1 image px = 1 Figma px (= 1dp)
- **Use `uiautomator dump` and code audit for numerical properties** — zero image cost
- **Recovery**: `/compact` drops old images from context (see Recovery section)

---

## Phase 1: Reconnaissance

When the user provides a Figma link, extract `fileKey` and `nodeId` from the URL and immediately start working. If any MCP call, build, or emulator interaction fails, run the [Doctor Check](#doctor-check) to diagnose and fix, then resume.

### Step 1: Get the Node Tree

Call `get_metadata(nodeId, fileKey)`:

```xml
<Frame id="45:1" name="Home" type="FRAME" x="0" y="0" width="360" height="800">
  <Frame id="45:10" name="TopAppBar" x="0" y="24" width="360" height="64"/>
  <Frame id="45:15" name="Cards" x="16" y="104" width="328" height="400">
    <Text id="45:16" name="Title" x="0" y="0" width="200" height="28"/>
  </Frame>
</Frame>
```

Build a **mental map**:
- Identify every major section and note each `nodeId`
- Note positions and sizes to understand the layout (rows, columns, boxes, lazy lists)
- Identify components vs frames vs text vs icons
- Note the **root frame size** — Figma px on an Android frame = dp. The emulator must render at that dp width (see Refinement Loop).
- Identify system chrome drawn in the frame (status bar, navigation bar) — don't reimplement what Android draws; handle it with window insets

### Step 2: Extract All Design Tokens (Once)

Call `get_variable_defs(nodeId, fileKey)`. Save the result — do NOT call this again.

### Step 3: Check Existing Components

Call `get_code_connect_map(nodeId, fileKey)`. For matched composables, follow this priority:
1. **Reuse as-is** — if it covers the Figma design exactly
2. **Extend minimally** — add a parameter or variant if close but not exact
3. **Compose** — combine existing composables
4. **Create new** — only if nothing existing fits

If Code Connect is empty, grep the project for existing composables (`@Composable fun *Button`, `*Card`, `*Row`, `*TopBar`) before creating anything.

### Step 4: Visual Overview

Call `get_screenshot(nodeId, fileKey)` on the root selection. This is Image 1 of your budget — your layout reference. Do NOT retake it.

**At this point you have NOT called `get_design_context` at all.** You have a complete structural map, all tokens, reusable component info, and a visual reference — all from cheap calls.

---

## Phase 2: Study & Implement (Code in the Dark)

Study the design deeply, memorize every detail, then code from memory.

### Step 1: Layout Shell

Using the metadata tree, implement the outer layout:
- Root container: `Scaffold` (top bar, bottom bar, FAB) as the design implies
- Major section placement with `Column`/`Row`/`Box`/`LazyColumn`/`LazyVerticalGrid`
- Background colors from tokens, edge-to-edge with correct `WindowInsets` padding
- Dividers and borders

The metadata + visual overview are usually sufficient. Only call `get_design_context` if you need specific properties you can't infer.

### Step 2: Study Every Section

For each major section:

1. **Figma Detail**: `get_screenshot(sectionNodeId, fileKey)` — zoomed-in visual reference.

2. **Design Context**: `get_design_context(sectionNodeId, fileKey)` — text only, no image cost. If truncated, use child node IDs from `get_metadata` and fetch children individually. The output is usually web code — read it as a spec, not as code to port.

3. **Absorb every detail**: fonts, sizes, weights, colors, spacing, corner radii, borders, shadows, icon shapes. Burn it into memory.

4. **Color sanity check**: Compare `get_design_context` colors against the Figma screenshot. If the screenshot reveals opacity layering, overlapping fills, or gradients — the raw token values will be wrong. Use the visual truth, not the raw token. Watch for Material 3 tonal elevation tinting surfaces.

**No emulator screenshots during this phase.** You are studying, not checking.

**Key rule**: NEVER call `get_design_context` on the root selection. Always target the smallest meaningful node.

### Step 3: Code from Memory

Implement everything using what you memorized:
- Design context output for exact properties
- Tokens from Phase 1 (do not re-extract)
- Reusable composables from Phase 1
- **Respect project rules**: Check for `.claude/rules`, `CLAUDE.md`, `AGENTS.md`, and project instruction files. Follow established patterns for module structure, file placement, naming, state hoisting, and ViewModel usage. Translate the design into the project's conventions.
- **Add a `@Preview`** for every new composable with `widthDp` matching the Figma frame, following the project's preview style.

**UI only.** Don't touch networking, repositories, database, or business logic unless the user explicitly asks. Use fake/sample data if models or data sources don't exist yet.

**Minimize screenshots during coding.** You studied the design — use what you memorized. But if you're missing crucial data for a specific element (exact icon shape, a nested layout you didn't drill into, a subtle gradient), take a targeted `get_screenshot` on that Figma node rather than guessing.

### Step 4: Design System Sync

NEVER hardcode values in composables. Sync to the project's design system:

- **Colors**: Add to the project's theme (`Color.kt` + `ColorScheme`, or a custom `CompositionLocal` palette). Never `Color(0xFFF3F3F3)` inline in a screen.
- **Typography**: Add to `Type.kt` / the project's `Typography`. Custom fonts go in `res/font/` and a `FontFamily`. Never inline `TextStyle(fontSize = 15.sp, ...)` in a screen when it belongs in the type scale.
- **Spacing & radii**: Add to the project's dimension/spacing object (or `Shapes`). Never scatter magic `dp` values.
- **Component library**: If the project has a `designsystem`/`core-ui` module, add tokens there, not in the feature module.

### Step 5: Icons & Assets

**Icons**: Find a match first — project drawables by name, then the project's icon set (`Icons.Default/Outlined/Rounded` if Material Icons is already a dependency, or its own `ImageVector` set) by visual shape. Match size and tint exactly.

**Fallback**: If nothing matches, export the icon from Figma as SVG and convert it to a Vector Drawable in `res/drawable/` (simple SVG paths map 1:1 to `<path android:pathData>`). Use `painterResource(R.drawable.ic_name)`.

**Images/illustrations**: Download from the Figma MCP assets endpoint and save to `res/drawable-nodpi/` (or density buckets) as WebP/PNG, or as a Vector Drawable for flat illustrations. Don't draw complex illustrations with `Canvas`. Don't add new Gradle dependencies without asking.

### Property Checklist

Before writing code for ANY element, verify ALL applicable properties:

- **Text**: font family, size (`sp`), weight, line height, letter spacing, color, alpha, `textAlign`, `maxLines`, `overflow`, text decoration, capitalization
- **Container**: size (prefer `fillMaxWidth`/`weight` over fixed), padding (all 4 sides), background, shape/corner radius, border, shadow, alpha, clipping — **modifier order matters** (padding before vs after background/clickable)
- **Icon**: size, tint (independent from parent `LocalContentColor`)
- **Button**: all text + container props + pressed/disabled states, ripple, `contentPadding`, minimum touch target (Material enforces 48dp — check it isn't inflating layout)
- **Image**: size, `contentScale`, `clip(shape)`, `aspectRatio`
- **Spacing**: `Arrangement.spacedBy`, `verticalArrangement`/`horizontalArrangement`

**Figma → Compose conversions:**

| Figma | Compose |
|-------|---------|
| Font size `S` px | `S.sp` |
| Line height `L` px | `lineHeight = L.sp` + `LineHeightStyle(alignment = Center, trim = None)` and `includeFontPadding = false` so the box matches Figma |
| Letter spacing `X%` | `letterSpacing = (X / 100f).em` |
| Letter spacing `Xpx` | `letterSpacing = X.sp` |
| Drop shadow (blur, spread, offset) | `Modifier.dropShadow(shape, Shadow(...))` on Compose UI 1.9+; otherwise `Modifier.shadow(elevation)` only approximates — match visually |
| Auto layout gap | `Arrangement.spacedBy(gap.dp)` |
| Auto layout "space between" | `Arrangement.SpaceBetween` |
| Fill container / Hug contents | `fillMaxWidth()` or `weight(1f)` / `wrapContentSize()` (default) |
| Corner smoothing | No exact equivalent — `RoundedCornerShape` and verify visually |

**Layout Principle**: Avoid fixed sizes. With correct text styles, padding, arrangement, and fill/weight modifiers, composables size themselves. Fixed `Modifier.size(247.dp)` on content is a symptom of wrong upstream layout.

**Adaptive**: Handle insets (`systemBarsPadding`, `Scaffold` `innerPadding`), keep layouts working on narrower devices and larger font scales. If the design has a dark variant, support it through the theme.

---

## Phase 3: Refinement Loop

You think you're done. Now prove it. Keep cycling until the result is perfect.

### Getting the Screen on the Emulator

- **Match the Figma frame**: density factor `d = dpi / 160` (`adb shell wm density`). The emulator's width in dp is `widthPx / d` (`adb shell wm size`). If it doesn't equal the Figma frame width, override temporarily: `adb shell wm size <figmaWidth*d>x<figmaHeight*d>` (reset in Phase 4).
- Font scale must be 1.0: `adb shell settings get system font_scale`.
- Build and install: `./gradlew :<appModule>:installDebug`, then launch: `adb shell monkey -p <applicationId> -c android.intent.category.LAUNCHER 1`.
- If the screen isn't the start destination: use an existing deep link (`adb shell am start -a android.intent.action.VIEW -d "<uri>" <applicationId>`) or navigate with `adb shell input tap x y`. As a last resort, add a **temporary** debug-only entry point (e.g. an intent extra checked under `BuildConfig.DEBUG`) and **remove it in Phase 4**.
- Match appearance to the design: `adb shell cmd uimode night no|yes`.
- Clean status bar for comparison (demo mode): `adb shell settings put global sysui_demo_allowed 1 && adb shell am broadcast -a com.android.systemui.demo -e command clock -e hhmm 0941`.

**Faster path**: If the project already uses Roborazzi, Paparazzi, or Compose Preview Screenshot Testing, you can render the `@Preview` to PNG without the emulator. Use it for inner loops, but do the final check on the emulator.

### The Loop

**Step 1: Emulator screenshot (1 image)**

```bash
adb exec-out screencap -p > /tmp/snap.png && sips --resampleWidth <figmaWidth> /tmp/snap.png
```

Read it and compare against the Figma screenshots in context. Be extremely picky:
- Visual alignment issues
- Missing elements or wrong proportions
- Spacing that looks off, wrong inset handling
- Wrong icon shapes (filled vs outlined, rounded vs sharp)
- Color or font weight mismatches
- **Color comparison**: Sample dominant colors from Figma vs Emulator. Flag anything that looks off — alpha, tonal elevation, and ripple/state layers can cause perceived differences even when hex values match.

**Step 2: Numerical audit (zero images)**

1. **Bounds audit**: Add `Modifier.testTag("snap_<name>")` to key composables and `Modifier.semantics { testTagsAsResourceId = true }` on the root (keep tags if the project uses them, otherwise remove in Phase 4). Then:
   ```bash
   adb exec-out uiautomator dump /dev/tty
   ```
   Convert each node's `bounds` from px to dp (divide by `d`) and compare against `get_metadata` positions and sizes.
2. **Code audit**: For every element, walk the Property Checklist and compare the literal values in code (after resolving tokens) against `get_design_context`.

12dp is not 10dp. #F97316 is not #FF611C.

**Content color check**: Icons and text inside `Button`/`Surface`/`TopAppBar` inherit `LocalContentColor`. If an icon's color doesn't match, set `tint` on the `Icon` itself or fix the component's `colors`.

**Step 3: Picky mismatch list**

Combine visual + numerical issues into one list. No issue is too small. 2dp off? List it.

**Step 4: Fix everything**

Batch all fixes. One rebuild, no screenshots between individual fixes.

**Step 5: Repeat from Step 1**

**Exit condition**: Retake `get_screenshot` on the Figma root node. Place side-by-side with your latest emulator screenshot. Compare colors directly. Only move on when both screenshots match AND the numerical audit shows zero mismatches.

### Icon & Shape Verification

If an icon looks wrong during any loop iteration:
- `get_screenshot(iconNodeId, fileKey)` on the Figma icon
- Crop the emulator icon from the full-resolution screenshot (before downscaling) for a zoomed view
- Compare the SILHOUETTE — stroke count, shape, proportions, fill vs outline

Common mismatches:
- `Icons.Filled` vs `Icons.Outlined` vs `Icons.Rounded`
- "filter" → `FilterList` vs `Tune`
- "more" → `MoreVert` vs `MoreHoriz`
- Icon rendered at 24dp default when the design uses 20dp

If it doesn't match: try another icon, or convert the Figma vector to a Vector Drawable.

### Anti-Pattern: "Looks Good Enough"

This ALWAYS misses: 2-4dp spacing differences, modifier order bugs, 48dp touch-target inflation, wrong icon variant, tonal elevation tint, missing shadow, font weight mismatch, font padding shifting text.

**RULE**: Not done until zero visual mismatches AND zero numerical mismatches.

---

## Recovery: Compact & Resume

If you hit the image limit (or proactively after ~15 screenshots):

1. **Run `/compact`** — summarizes conversation, drops old images. Notes and progress are preserved.
2. **Retake 3 reference images**: Figma overview, Figma detail (current section), Emulator state.
3. **Continue the refinement loop** — you still know everything from the compacted history. ~17 more image slots available.

Repeatable: compact → retake 3 → continue → compact again if needed.

---

## Phase 4: Cleanup & User Review

**Cleanup first**:
- Remove any temporary debug entry points and `snap_*` test tags you added
- Reset emulator overrides: `adb shell wm size reset`, `adb shell wm density reset`, `adb shell am broadcast -a com.android.systemui.demo -e command exit`
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

**Build failing?**
- Use the Gradle wrapper (`./gradlew`), never a global `gradle`
- Find the app module: the `build.gradle(.kts)` applying `com.android.application`
- `JAVA_HOME` errors → point it at Android Studio's JBR (`/Applications/Android Studio.app/Contents/jbr/Contents/Home` on macOS)
- Read the compiler errors and fix them before continuing the loop

**Emulator not responding?**
- `adb devices` — nothing listed → `emulator -list-avds`, then `emulator -avd <name> &` and `adb wait-for-device`
- Find the package: `applicationId` (+ `applicationIdSuffix` for debug) in the app module's build file
- App not launching → `adb shell cmd package resolve-activity --brief <applicationId>`

**Unknown design system or icon set?**
- Grep for `MaterialTheme(`, `darkColorScheme`/`lightColorScheme`, `Typography(`, `staticCompositionLocalOf`, `object Dimens`, `designsystem`
- Check `res/values/colors.xml`, `res/font/`, `res/drawable/`
- Check `libs.versions.toml` / build files for `material-icons-extended` or other icon libraries

---

## Examples

**Invocation:**
```
/snap-compose
> Paste Figma link: https://figma.com/design/abc123/MyApp?node-id=42-100
```

**What happens:**
1. Recon: `get_metadata` + `get_variable_defs` + `get_code_connect_map` (0 images). Frame is 360×800 dp; emulator is 1080px @ 420dpi = 411dp → override `wm size` to 945×2100.
2. Figma overview: `get_screenshot` on root (image 1)
3. Study section 1: `get_screenshot` (image 2) + `get_design_context` (text). Memorize.
4. Study section 2: `get_screenshot` (image 3) + `get_design_context` (text). Memorize.
5. Code everything from memory, add colors to the theme (0 images)
6. **Loop round 1**: install + emulator screenshot (image 4) + `uiautomator` audit → 4 mismatches + 1 wrong icon
7. Fix mismatches. Icon check: `get_screenshot` on Figma icon (image 5) — swap `Filled` → `Outlined`.
8. **Loop round 2**: emulator screenshot (image 6) + audit → `padding` placed after `background`
9. Fix modifier order.
10. **Loop round 3**: emulator screenshot (image 7) + audit → zero mismatches.
11. **Final check**: Retake Figma `get_screenshot` (image 8) side-by-side with emulator → match.
12. Cleanup, ask for review. Total: 8 images.

**Bad patterns:**
- Screenshotting every element individually (budget killer)
- Reading raw device-px screenshots (wastes resolution budget, breaks 1:1 comparison)
- Comparing against an emulator whose dp width differs from the Figma frame
- Re-screenshotting the same Figma node (it's static)
- Rebuilding and screenshotting after every small tweak
- Porting `get_design_context` web code literally into Compose
- `get_design_context` on root (token waste)
- `get_variable_defs` more than once (redundant)
- `Color(0xFF...)` or inline `TextStyle` in screens
- Fixed `Modifier.size(247.dp)` on content that should size itself
- Reimplementing the status bar or navigation bar drawn in the Figma frame
- Saying "looks close" without numerical verification
