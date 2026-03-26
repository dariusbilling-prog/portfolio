# How to Import Into Figma

## Step 1 — Install Tokens Studio
1. Open Figma
2. Go to Plugins → Browse Plugins
3. Search "Tokens Studio for Figma"
4. Install (free)

## Step 2 — Import tokens.json
1. Open any Figma file
2. Run Tokens Studio plugin
3. Click the settings icon (bottom left of plugin)
4. Choose "Import" → select `tokens.json`
5. Click "Import Tokens"

## Step 3 — Apply to Figma Styles + Variables
1. In Tokens Studio, click "Styles & Variables" (top right)
2. Click "Export Styles" → this creates Figma Color Styles, Text Styles
3. Click "Export Variables" → this creates Figma Variables for spacing, radius, etc.

## Step 4 — Use in Figma
- **Colors** appear under Local Styles → Color Styles
- **Text Styles** appear under Local Styles → Text Styles
- **Spacing + Radius** appear under Local Variables

---

## Token Architecture

```
primitive/     → raw values (hex colors, px numbers)
semantic/      → meaningful aliases (background/page → neutral.150)
component/     → specific component rules (button.primary.background → coral.400)
```

Always reference semantic tokens in your components, not primitive ones directly.
This means changing one primitive value updates the whole system automatically.

---

## Fonts Needed
Install these before using text styles:
- Półtawski Nowy → https://fonts.google.com/specimen/Pol%C5%82tawski+Nowy
- Inter → https://fonts.google.com/specimen/Inter

Or use the Google Fonts plugin inside Figma to install them directly.
