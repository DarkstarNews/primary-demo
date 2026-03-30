# Card Flip — Factual Context on Tap

## The Idea
User taps the card → it flips (3D Y-axis rotation) → back side shows factual context to help form an informed opinion. No editorializing. Just facts. Tap again to flip back and swipe.

## Current State
- `SwipeCardView.swift` handles the front face (category, headline, stancePrompt, source, swipe hints)
- `PolicyCard.swift` model already has `caseFor` / `caseAgainst` but those live in the RevealView (post-swipe)
- The pagination dots (top right of card) suggest multi-page was planned — the flip replaces or complements this
- Swipe gesture is on the card via `highPriorityGesture` — tap needs to coexist without conflicting

## Interaction Design

### Gestures
- **Tap** (no drag) → flip card 180° on Y-axis
- **Drag** → swipe as usual (agree/disagree/skip) — unchanged
- The `DragGesture` already uses `highPriorityGesture`, so a `TapGesture` can coexist naturally
- When flipped, **tap again** → flip back
- When flipped, **swipe still works** (user can swipe from the back too)

### Animation
- `rotation3DEffect(.degrees(isFlipped ? 180 : 0), axis: (x: 0, y: 1, z: 0))`
- Duration: ~0.5s spring
- Card front gets `opacity(isFlipped ? 0 : 1)` at 90° to hide during backface
- Card back gets the inverse

### Haptics
- `Haptics.light()` on flip

## Back Face Content

### Data Model Addition to `PolicyCard`
```swift
struct FactContext: Codable {
    let keyFacts: [String]           // 3-5 bullet points — numbers, dates, outcomes
    let timeline: [TimelineEntry]?   // Chronological events (optional)
    let primarySources: [Source]     // Links to actual documents
    let unknowns: [String]?         // What's still unclear (optional)
    
    struct TimelineEntry: Codable {
        let date: String
        let event: String
    }
    
    struct Source: Codable {
        let label: String
        let url: String?
    }
}

// Add to PolicyCard:
var factContext: FactContext?
```

### Back Face Layout (matches card dimensions exactly)
```
┌─────────────────────────────────────┐
│  [Category pill]          [↻ Flip]  │
│                                     │
│  KEY FACTS                          │
│  • Fact with specific numbers       │
│  • Fact with dates                  │
│  • Fact with outcomes               │
│                                     │
│  ─────────────────                  │
│                                     │
│  TIMELINE                           │
│  ● Mar 15 — What happened           │
│  │                                  │
│  ● Mar 20 — What happened next      │
│  │                                  │
│  ● Mar 28 — Current state           │
│                                     │
│  ─────────────────                  │
│                                     │
│  WHAT'S STILL UNKNOWN               │
│  ? Unanswered question 1            │
│  ? Unanswered question 2            │
│                                     │
│  ─────────────────                  │
│                                     │
│  SOURCES                            │
│  [CBO Report]  [HR-4521 Text]      │
│                                     │
└─────────────────────────────────────┘
```

### Typography & Colors (from DesignSystem.swift)
- Section headers: `Typography.labelSmall`, `.tracking(2.5)`, `.textTertiary`
- Fact bullets: `Typography.bodySmall` (13pt regular), `.textPrimary.opacity(0.85)`
- Timeline dates: `Typography.labelMedium`, `.primaryAccent`
- Timeline events: `Typography.bodySmall`, `.textSecondary`
- Unknowns: `Typography.bodySmall`, `.textTertiary`
- Source pills: `Typography.labelMedium`, accent/10 background, accent text
- Flip icon: SF Symbol `arrow.trianglehead.2.counterclockwise.rotate.90`, `.textTertiary`
- Back has same glass gradient background as front
- Scrollable via `ScrollView` if content overflows

### What's NOT on the Back
- No "case for / case against" — that's in RevealView post-swipe
- No opinion language, no adjectives implying judgment
- No "analysts say" or "experts believe" — just what happened
- No agree/disagree hints (you can still swipe from the back)

## Implementation

### New File: `CardBackView.swift`
Renders the back face content from `PolicyCard.factContext`.

### Modified: `SwipeCardView.swift`
- Add `@State private var isFlipped = false`
- Wrap front content + back content in a ZStack with rotation3DEffect
- Add `.onTapGesture { isFlipped.toggle(); Haptics.light() }`
- Both faces get the same card chrome (glass gradient, border, glow)

### Modified: `PolicyCard.swift`
- Add `factContext: FactContext?` property

### Modified: API / Mock Data
- Server sends `factContext` in the card payload
- Fallback: if `factContext` is nil, tap does nothing (or shows a minimal "Context coming soon" state)

## Pagination Dots
The dots in the top-right currently show 3 positions. Options:
1. **Replace with flip:** Dots become a flip indicator (filled = front, hollow = back)
2. **Keep as pages:** Dots represent front / context / sources as horizontal swipe pages WITHIN the card (alternative to flip)
3. **Remove:** If flip handles it, dots aren't needed

**Recommendation:** Option 1 — simplify to a single flip indicator. Two states (front/back) is cleaner than three pages.

## Priority
This is the core "informed opinion" mechanic. Users should never have to swipe blind — the context is always one tap away.
