# Prompt for Antigravity CLI (agy) — Card Redesign

## Task

You are in `/Users/satvikjain/Downloads/Mission-Job-Switch/PWA/monopoly-deal-fix`. Your job is to redesign every card in this Monopoly Deal Indian Edition React app so that instead of being rendered procedurally with CSS (current approach), each card displays a **PNG image** that exactly mirrors the **official Monopoly Deal card design** but with **Indian-themed content** (Indian city names, Hindi action names, ₹Cr values, Indian landmark icons).

## Step 1: Study the Reference Images

Read every PNG in `monopoly-deal-rules-and-images/images/` (52 files). These are scans of the official Monopoly Deal cards. For each card type, extract the exact design system:

### Property cards (brown-property-card.png, light-blue-property-card.png, etc.)
- Color bar at top: exact height, what text goes inside it (property name font/size/weight/color)
- Corner circle: position, size, what value it shows (the cash value of the property if banked)
- Body layout: "RENT" label, the rent ladder (how many rent values per set, the (set) indicator), any icon/illustration
- Bottom area: building costs (house = ₹3M, hotel = ₹4M) — how are these shown?
- Overall card dimensions, border radius, white background

### Money cards (10M-money-card.png, etc.)
- Green gradient background
- Denomination text position, font, size, color
- "Money" label position and style
- Any decorative elements

### Action cards (deal-breaker-action-card.png, etc.)
- Dark blue/purple gradient background
- Action name text position, font, size, color
- Description text position
- Value in corner (top or bottom?)
- Icon/illustration in center
- "ACTION" label

### Rent cards (brown-and-light-blue-rent-card.png, etc.)
- Dual-color split background
- Rent icon (the rupee/hand symbol)
- Color pair label
- "RENT" label

### Wild cards (multicolor-wildcard-card.png, etc.)
- Rainbow gradient or dual-color split
- "WILD" label
- Which colors this wild can represent
- Value (if any)

Output a **design-system-analysis.md** document with all measurements, hex colors, font specs, and layout rules you extracted.

## Step 2: Read the Current App Code

Read these files completely:
- `src/game/constants.js` — has ALL card definitions: 28 properties (Indian cities), 10 wilds, 20 money, 35 actions, 13 rent. Exact names, values, colors, rent values per card.
- `docs/card-image-mapping.md` — maps every official card → Indian card → which reference image to base the design on.
- `src/components/game/Card.jsx` — the current procedural renderer. Understand the layout so your images match the same visual hierarchy.
- `src/components/game/CardArt.jsx` — has Indian landmark SVG icons for all 28 cities (Gateway of India, Charminar, Hawa Mahal, Howrah Bridge, India Gate, etc.)

## Step 3: Generate Indian Card Images

Use your image generation capability to create one PNG per **unique card design**. Cards that are identical except for the name (e.g., all 3 Pink properties use the same layout just different names) should be generated as separate files since each has unique text.

### Naming Convention

Save all images to `public/images/cards/` with this naming:

**Properties:** `prop-{color}-{cityKey}.png`
  - e.g., `prop-brown-indore.png`, `prop-brown-lucknow.png`
  - e.g., `prop-red-bengaluru.png`, `prop-darkBlue-lutyensdelhi.png`

**Wilds:** `wild-{color1}-{color2}.png`
  - e.g., `wild-rainbow.png` (the 2 multicolor wilds)
  - e.g., `wild-railroad-utility.png`, `wild-pink-orange.png`

**Money:** `money-{value}cr.png`
  - e.g., `money-1cr.png`, `money-10cr.png` (one per denomination)

**Actions:** `action-{actionType}.png`
  - e.g., `action-dealBreaker.png`, `action-birthday.png`, `action-justSayNo.png`
  - For Hindi-named ones: `action-passGo.png` (text still says "Pass Go"), `action-birthday.png` (text says "Mera Birthday!")

**Rent:** `rent-{color1}-{color2}.png`
  - e.g., `rent-brown-lightBlue.png`, `rent-railroad-utility.png`
  - Wild rent: `rent-wild.png`

### Image Specs

- Dimensions: 420px × 610px (5:7 ratio, matches standard playing card)
- Resolution: 150 DPI minimum
- Format: PNG with transparency? No — use white/colored background exactly like official cards
- Card corners: rounded rectangle, border-radius ~20px

### Design Requirements for Each Card Type

#### Property Cards (28 unique)
- **Header bar:** Color of the property set. Inside the bar: Indian city name in bold white (or black for light colors), centered.
- **Corner circle:** Top-left, white circle with colored border. Shows the cash value (₹1, ₹2, ₹3, or ₹4).
- **Body area (white):**
  - "RENT" label in small grey uppercase text, left-aligned
  - Below it: the rent ladder. Each row: number of cards (1, 2, 3...) on left, rent value in ₹ on right. Last row marked "(set)" and highlighted in the property color.
  - Small landmark icon next to the rent ladder (use the SVGs from CardArt.jsx as reference — draw a simplified version of each landmark)
- **Bottom strip:** House/Hotel cost info. "House ₹3M" / "Hotel ₹4M" in small text. (Railroads and utilities don't have this.)
- **Colors exactly as in constants.js:**
  - Brown: #955436, Light Blue: #55C3F0, Pink: #D93A96, Orange: #F7941D
  - Red: #ED1C24, Yellow: #FEF200 (use dark text on this), Green: #1FB25A
  - Dark Blue: #003F9E, Railroad: #2C2C2C, Utility: #00796B

#### Money Cards (6 unique — one per denomination)
- Full green gradient background (#2E7D32 → #43A047)
- Large "₹{value}Cr" text in white, bold, centered
- Small "Money" label below in lighter white
- A subtle ₹ icon or decoration

#### Action Cards (10 unique types)
- Dark blue gradient background (#1A237E → #283593)
- Large icon in center (use reference images for which icon goes where)
- Card name at top or middle: "Deal Breaker", "Mera Birthday!", "Nahi!", "Double Rent!", "Ghar", "Hotel", etc.
- Description text (in Hindi): from ACTION_DESC in Card.jsx
- Value in bottom-right or top-right circle: ₹5Cr, ₹3Cr, etc.
- **Special:** "Just Say No" card has a unique red accent in official version — replicate that.

#### Rent Cards (6 unique)
- Split background: 50/50 diagonal or vertical split of the two colors
- White ₹ icon (CurrencyRupeeIcon) centered, with drop shadow
- Card name below: "Rent: Brown/Light Blue", "Rent: Pink/Orange", etc.
- Wild Rent: green gradient background, says "Wild Rent" and "Kisi se bhi rent lo"

#### Wild Cards (9 unique)
- Full multicolor (2): rainbow gradient (use colors: #FF6B6B, #FFE66D, #4ECDC4, #45B7D1, #A855F7)
- Dual-color (7): 50/50 split of the two colors
- Text: "Wild" and description in white with text shadow
- Value shown for non-rainbow wilds

## Step 4: Update Card.jsx to Use Images

Modify `src/components/game/Card.jsx` so that:
- For **full-size cards** (!mini): render the PNG image instead of the procedural layout
- For **mini cards** (mini): keep the current procedural rendering (it's small, images won't look good at 52×74px)
- Build a `cardImage` mapping function that takes a card object and returns the correct image path:
  ```js
  function getCardImage(card) {
    if (card.type === 'property') return `/images/cards/prop-${card.color}-${card.landmark}.png`
    if (card.type === 'money') return `/images/cards/money-${card.value}cr.png`
    if (card.type === 'action') return `/images/cards/action-${card.actionType}.png`
    if (card.type === 'rent') {
      if (card.wild) return `/images/cards/rent-wild.png`
      return `/images/cards/rent-${card.colors[0]}-${card.colors[1]}.png`
    }
    if (card.type === 'wildProperty') {
      if (card.colors[0] === 'wild') return `/images/cards/wild-rainbow.png`
      return `/images/cards/wild-${card.colors.join('-')}.png`
    }
    return null
  }
  ```
- Import `getCardImage` and render an `<img>` tag inside the Paper component for full-size cards
- Keep the existing `mini` rendering path unchanged
- Make sure image cards have proper `alt` text (card name)
- Cards that fail to load an image should fall back to the current procedural rendering

## Step 5: Update HomeScreen (Optional but Nice)

Check `src/components/screens/HomeScreen.jsx`. It currently uses the 6 generic PNGs in `public/images/cards/`. Update it to use actual card images from your new set.

## Step 6: Verify

1. Run `npm run build` — must succeed with no errors
2. Run `npm test` — all tests must pass
3. Start the dev server with `npm run dev` and visually confirm cards display correctly

## Quality Checklist

- [ ] Every card has the EXACT same layout as its official counterpart
- [ ] Colors match the official palette exactly (from constants.js)
- [ ] Text is the Indian version (city name, Hindi action name)
- [ ] Values are in ₹Cr (not ₹M)
- [ ] All 106 cards have unique images (not one image per color — each card is distinct)
- [ ] Images are sharp at 420×610px
- [ ] Card.jsx image fallback works (if image missing, procedural rendering shows)
- [ ] Build passes with no warnings
- [ ] Mini cards still use procedural rendering (they're too small for images)

## Critical Rules

1. DO NOT delete or modify `src/game/constants.js` — card data must stay the same
2. DO NOT delete `CardArt.jsx` — mini cards still need the SVGs
3. Each card gets its own image file — no sharing images between cards with different names
4. The "Just Say No" (Nahi!) card must have the red accent exactly like the official version
5. Pass Go cards: official shows a running figure with "₹2M" — yours should say "₹1Cr" and show a running figure
6. Images must be production quality — not blurry, not distorted
7. If you cannot generate an image for a card (technical limitation), leave the procedural rendering as fallback

## File Structure After Completion

```
public/images/cards/
├── prop-brown-indore.png
├── prop-brown-lucknow.png
├── prop-lightBlue-chandigarh.png
├── ... (28 property cards)
├── wild-rainbow.png
├── wild-rainbow.png (2 copies or 1 shared for the 2 multicolor wilds)
├── wild-railroad-utility.png
├── ... (9 wild cards)
├── money-1cr.png
├── money-2cr.png
├── ... (6 money denominations)
├── action-dealBreaker.png
├── action-birthday.png
├── ... (10 action types)
├── rent-brown-lightBlue.png
├── rent-wild.png
├── ... (6 rent types)
```
