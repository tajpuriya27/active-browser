# Assets

Source of truth is SVG; the binary formats below are generated from it.

| File | What it is |
| --- | --- |
| `icon.svg` | App icon artwork, 1024×1024 |
| `icon-1024.png` | Flat preview / upload asset |
| `AppIcon.icns` | Generated app icon for `Contents/Resources/` |
| `menubar-icon.svg` | Menu bar glyph, 18×18pt, black + alpha (template image) |
| `menubar/MenuBarIconTemplate{,@2x,@3x}.png` | Generated menu bar images |
| `tools/render.swift` | AppKit-only SVG→PNG rasteriser (no third-party deps) |

## Regenerating

Run these from `assets/`; the paths are relative.

```sh
rm -rf /tmp/AppIcon.iconset && mkdir -p /tmp/AppIcon.iconset
for s in 16 32 128 256 512; do
  swift tools/render.swift icon.svg /tmp/AppIcon.iconset/icon_${s}x${s}.png $s
  swift tools/render.swift icon.svg /tmp/AppIcon.iconset/icon_${s}x${s}@2x.png $((s*2))
done
iconutil -c icns /tmp/AppIcon.iconset -o AppIcon.icns
swift tools/render.swift menubar-icon.svg menubar/MenuBarIconTemplate.png 18
swift tools/render.swift menubar-icon.svg menubar/MenuBarIconTemplate@2x.png 36
swift tools/render.swift menubar-icon.svg menubar/MenuBarIconTemplate@3x.png 54
```

## Design

Both marks are the same idea: browser chrome (title bar + three dots) with the active
pointer inside it. The app icon adds the receding stack and colour; the menu bar glyph is the
monochrome reduction — the app icon's tinted title-bar strip becomes a divider line, since a
template image has only black and alpha to work with.

## How the bundle uses these

`make bundle` copies `AppIcon.icns` and the three `MenuBarIconTemplate` PNGs into
`Contents/Resources/` before it runs `codesign`. `Support/Info.plist` names the app icon
through `CFBundleIconFile` (`AppIcon`), and `MenuBarManager` loads the menu bar glyph by
name, falling back to the `globe` SF Symbol when `Contents/Resources` is missing (an
unbundled build).

The menu bar image must be loaded with `isTemplate = true` so macOS tints it for light/dark
menu bars and for the highlighted state.
