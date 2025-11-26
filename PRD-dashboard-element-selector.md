# PRD: Dashboard Element Selector

**Feature Name:** Interactive Element Context Selector
**Route:** `/test-dashboard-stagewise`
**Status:** Draft
**Created:** 2025-11-12
**Owner:** Development Team

---

## 1. Executive Summary

Implement an interactive element selector for the dashboard that allows users to click UI elements (charts, metrics, text) to add them to a context collection. Selected elements are displayed in a sidebar with visual highlights, enabling users to build context for AI interactions or data analysis.

### Success Metrics
- Users can select 5+ elements in under 30 seconds
- No visible flickering or lag during hover/selection
- 95%+ accuracy in element detection
- Smooth 30 FPS overlay performance

---

## 2. User Story

**As a** dashboard user
**I want to** click on charts, metrics, and UI elements to add them to a context collection
**So that** I can build a curated set of data points for analysis or AI assistance

### User Flow

```
1. User lands on /test-dashboard-stagewise
2. User sees dashboard with multiple charts/metrics
3. User clicks "Add to Context" button
4. Screen enters selection mode:
   - Cursor changes to crosshair
   - Hovering elements shows blue dotted outline
   - Element label appears on hover
5. User clicks an element (e.g., revenue chart)
6. Element gets green dotted outline
7. Sidebar shows chip with element info
8. User clicks more elements (metrics, cards, etc.)
9. Sidebar updates with all selected elements
10. User hovers over chip in sidebar
    - Corresponding element highlights in blue
11. User clicks X on chip to remove element
    - Element highlight disappears
    - Chip removed from sidebar
12. User clicks "Done" to exit selection mode
13. Selected context persists in sidebar
```

---

## 3. Feature Requirements

### 3.1 Functional Requirements

#### FR-1: Selection Mode Activation
- **FR-1.1:** Button labeled "Add to Context" visible on dashboard
- **FR-1.2:** Clicking button activates selection mode
- **FR-1.3:** In selection mode, button changes to "Done"
- **FR-1.4:** Clicking "Done" deactivates selection mode
- **FR-1.5:** Keyboard shortcut `Ctrl+Alt+.` (or `⌘+⌥+.` on Mac) toggles selection mode
- **FR-1.6:** `Esc` key exits selection mode

#### FR-2: Element Detection & Hover
- **FR-2.1:** In selection mode, hovering over elements shows preview highlight
- **FR-2.2:** Preview highlight is blue dotted border (2px)
- **FR-2.3:** Element tag name displayed in label (e.g., "div", "svg", "section")
- **FR-2.4:** Cursor changes to crosshair in selection mode
- **FR-2.5:** Hover updates smoothly without flickering
- **FR-2.6:** Ignores hover over selector UI itself (overlays, sidebar, button)

#### FR-3: Element Selection
- **FR-3.1:** Clicking hovered element adds it to selection
- **FR-3.2:** Selected elements show green dotted border (2px)
- **FR-3.3:** Clicking already-selected element does nothing (must remove via sidebar)
- **FR-3.4:** Hovering over selected element shows no blue outline
- **FR-3.5:** Multiple elements can be selected simultaneously
- **FR-3.6:** Selection persists when exiting/re-entering selection mode

#### FR-4: Context Sidebar
- **FR-4.1:** Sidebar visible on right side of screen (fixed position)
- **FR-4.2:** Shows list of selected elements as chips
- **FR-4.3:** Each chip displays:
  - Element tag name (e.g., "div")
  - Element ID if available (e.g., "div#revenue-chart")
  - Text preview (first 30 chars) if element has text
  - X button to remove
- **FR-4.4:** Sidebar header shows count: "Context (3)"
- **FR-4.5:** Sidebar scrollable if many elements selected
- **FR-4.6:** Empty state shows "No elements selected"

#### FR-5: Chip Hover Synchronization
- **FR-5.1:** Hovering over chip in sidebar highlights corresponding element
- **FR-5.2:** Highlight changes from green to blue during chip hover
- **FR-5.3:** Highlight returns to green on chip unhover
- **FR-5.4:** Works even when not in selection mode

#### FR-6: Element Removal
- **FR-6.1:** Clicking X on chip removes element from selection
- **FR-6.2:** Highlight overlay disappears immediately
- **FR-6.3:** Chip animates out of sidebar
- **FR-6.4:** In selection mode, hovering over selected element shows red outline
- **FR-6.5:** Clicking red-outlined element removes it from selection

### 3.2 Non-Functional Requirements

#### NFR-1: Performance
- **NFR-1.1:** Hover detection runs at 28-30 FPS
- **NFR-1.2:** Overlay position updates at 30 FPS
- **NFR-1.3:** No visible lag or stuttering during mouse movement
- **NFR-1.4:** Page load time unaffected (selector lazy-loaded)

#### NFR-2: Visual Design
- **NFR-2.1:** Overlays use consistent color scheme:
  - Hover: `border-blue-500/70`, `bg-blue-500/5`
  - Selected: `border-green-500/70`, `bg-green-500/5`
  - Hover-to-remove: `border-red-500/70`, `bg-red-500/5`
  - Chip hover: `border-blue-500/70`, `bg-blue-500/5`
- **NFR-2.2:** Smooth transitions (100ms) for color changes
- **NFR-2.3:** Labels have dark background for readability
- **NFR-2.4:** Sidebar follows dashboard design system

#### NFR-3: Accessibility
- **NFR-3.1:** Keyboard shortcuts work consistently
- **NFR-3.2:** Focus management when entering/exiting selection mode
- **NFR-3.3:** Screen reader announces mode changes
- **NFR-3.4:** High contrast mode supported

#### NFR-4: Browser Compatibility
- **NFR-4.1:** Works in Chrome 90+, Firefox 88+, Safari 14+, Edge 90+
- **NFR-4.2:** Graceful degradation for older browsers
- **NFR-4.3:** Mobile responsive (optional for v1)

---

## 4. Technical Specification

### 4.1 Architecture

```
/test-dashboard-stagewise (Next.js page)
├── DashboardContent (charts, metrics, cards)
└── ElementSelectorWrapper
    ├── ElementSelectorProvider (context)
    ├── SelectorOverlay (full-screen capture layer)
    │   ├── HoverHighlight (blue dotted overlay)
    │   └── SelectedHighlight[] (green/blue/red overlays)
    ├── ContextSidebar (right panel)
    │   ├── Header ("Context (3)")
    │   ├── ElementChip[]
    │   └── EmptyState
    └── SelectorControls (Add to Context / Done button)
```

### 4.2 Key Components

#### Component: `ElementSelectorProvider`
**Purpose:** Global state management via React Context
**State:**
```typescript
interface ElementSelectorState {
  isActive: boolean;                    // Selection mode on/off
  hoveredElement: HTMLElement | null;   // Currently hovered element
  selectedElements: SelectedElement[];  // Array of selected elements
  chipHoveredElement: HTMLElement | null; // Element whose chip is hovered
}

interface SelectedElement {
  id: string;                          // Unique ID (random)
  element: HTMLElement;                // DOM reference
  tagName: string;                     // e.g., "div"
  textContent: string;                 // First 100 chars
  rect: DOMRect;                       // Position/size at selection time
  timestamp: number;                   // Selection timestamp
}
```

**Actions:**
```typescript
- toggleActive()
- setHoveredElement(element)
- addSelectedElement(element)
- removeSelectedElement(id)
- setChipHoveredElement(element)
- clearAllSelected()
```

---

#### Component: `SelectorOverlay`
**File:** `components/element-selector/SelectorOverlay.tsx`
**Purpose:** Full-screen transparent layer for mouse event capture

**Props:** None (uses context)

**Key Features:**
- Full-screen fixed-position div (`fixed inset-0 z-40`)
- Captures mouse move, click, leave events
- Velocity-based throttling (>30px/s delays updates)
- Updates hovered element at ~28 FPS
- Changes cursor to `cursor-crosshair`

**Performance Optimizations:**
- Uses refs to track mouse state (no re-renders on movement)
- `setTimeout` for throttled updates (1000/28 ms)
- Clears timeout on fast movement, reschedules when slowing
- Duplicate element check via ref comparison

**Code Pattern:**
```typescript
const handleMouseMove = (event: MouseEvent) => {
  // Calculate velocity
  const velocity = (distance / deltaTime) * 1000;

  // Throttle based on velocity
  if (velocity > 30 || !nextUpdateTimeout.current) {
    clearTimeout(nextUpdateTimeout.current);
    nextUpdateTimeout.current = setTimeout(updateHoveredElement, 1000 / 28);
  }
};
```

---

#### Component: `HoverHighlight`
**File:** `components/element-selector/HoverHighlight.tsx`
**Purpose:** Blue dotted overlay for hovered element

**Props:**
```typescript
interface HoverHighlightProps {
  element: HTMLElement;
}
```

**Key Features:**
- Fixed-position overlay tracking element bounds
- Updates position at 30 FPS via `useCyclicUpdate(updatePosition, 30)`
- Direct DOM manipulation via refs (no React re-renders)
- Tag name label in top-left corner
- Semi-transparent blue background

**Code Pattern:**
```typescript
const updatePosition = useCallback(() => {
  if (boxRef.current && element) {
    const rect = element.getBoundingClientRect();
    boxRef.current.style.top = `${rect.top - 2}px`;
    boxRef.current.style.left = `${rect.left - 2}px`;
    boxRef.current.style.width = `${rect.width + 4}px`;
    boxRef.current.style.height = `${rect.height + 4}px`;
  }
}, [element]);

useCyclicUpdate(updatePosition, 30);
```

---

#### Component: `SelectedHighlight`
**File:** `components/element-selector/SelectedHighlight.tsx`
**Purpose:** Green/blue/red dotted overlay for selected elements

**Props:**
```typescript
interface SelectedHighlightProps {
  element: HTMLElement;
  isChipHovered: boolean;      // Chip is being hovered
  isHoveredToRemove: boolean;  // In selection mode, user hovering element
  onRemove: () => void;
}
```

**Key Features:**
- Updates position at 30 FPS
- Dynamic border color based on state:
  - Default: green (`border-green-500/70`)
  - Chip hovered: blue (`border-blue-500/70`)
  - Hover to remove: red (`border-red-500/70`)
- Clickable to remove (when in selection mode)
- Smooth color transitions (100ms)

---

#### Component: `ContextSidebar`
**File:** `components/element-selector/ContextSidebar.tsx`
**Purpose:** Right sidebar showing selected elements

**Layout:**
```typescript
<aside className="fixed top-0 right-0 h-screen w-80
                  bg-white border-l shadow-lg z-50">
  <header>
    <h2>Context ({selectedElements.length})</h2>
    <button onClick={clearAll}>Clear All</button>
  </header>

  <div className="overflow-y-auto">
    {selectedElements.length === 0 ? (
      <EmptyState />
    ) : (
      selectedElements.map(el => (
        <ElementChip key={el.id} element={el} />
      ))
    )}
  </div>
</aside>
```

---

#### Component: `ElementChip`
**File:** `components/element-selector/ElementChip.tsx`
**Purpose:** Individual chip representing selected element

**Props:**
```typescript
interface ElementChipProps {
  element: SelectedElement;
  onRemove: () => void;
  onHover: (element: HTMLElement) => void;
  onUnhover: () => void;
}
```

**Layout:**
```typescript
<div
  className="flex items-center gap-2 p-2 border rounded hover:bg-blue-50"
  onMouseEnter={() => onHover(element.element)}
  onMouseLeave={() => onUnhover()}
>
  <SquareDashedMousePointer className="w-4 h-4 text-gray-500" />
  <div className="flex-1 min-w-0">
    <div className="font-medium text-sm">{element.tagName}</div>
    <div className="text-xs text-gray-500 truncate">
      {element.textContent}
    </div>
  </div>
  <button onClick={onRemove}>
    <X className="w-4 h-4 text-gray-400 hover:text-red-500" />
  </button>
</div>
```

---

#### Hook: `useCyclicUpdate`
**File:** `hooks/useCyclicUpdate.ts`
**Purpose:** requestAnimationFrame loop with frame rate limiting

**Signature:**
```typescript
function useCyclicUpdate(func: () => void, frameRate?: number): void
```

**Implementation:**
```typescript
export function useCyclicUpdate(func: () => void, frameRate?: number) {
  const animationFrameHandle = useRef<number>();
  const timeBetweenFrames = frameRate ? 1000 / frameRate : 0;
  const lastCallFrameTime = useRef<number>(0);

  const update = useCallback((frameTime: number) => {
    if (frameTime - lastCallFrameTime.current >= timeBetweenFrames) {
      func();
      lastCallFrameTime.current = frameTime;
    }
    animationFrameHandle.current = requestAnimationFrame(update);
  }, [func, timeBetweenFrames]);

  useEffect(() => {
    animationFrameHandle.current = requestAnimationFrame(update);
    return () => {
      if (animationFrameHandle.current) {
        cancelAnimationFrame(animationFrameHandle.current);
      }
    };
  }, [update]);
}
```

---

#### Utility: `getElementAtPoint`
**File:** `utils/elementDetection.ts`
**Purpose:** Get element at coordinates, filtering out selector UI

**Signature:**
```typescript
function getElementAtPoint(x: number, y: number): HTMLElement
```

**Implementation:**
```typescript
export function getElementAtPoint(x: number, y: number): HTMLElement {
  // Validate coordinates
  if (!Number.isFinite(x) || !Number.isFinite(y)) {
    return document.body;
  }

  const elements = document.elementsFromPoint(x, y);

  // Filter out selector UI
  const element = elements.find(el =>
    !el.closest('[data-selector-overlay]') &&
    !el.closest('[data-selector-sidebar]') &&
    !el.closest('[data-selector-controls]') &&
    !el.closest('svg') // Optionally skip SVG internals
  );

  return (element as HTMLElement) || document.body;
}
```

---

### 4.3 Data Flow

```
User Clicks "Add to Context"
  ↓
toggleActive() → isActive = true
  ↓
SelectorOverlay renders (full-screen capture)
  ↓
User Moves Mouse
  ↓
handleMouseMove() → Calculate velocity
  ↓
velocity > 30px/s? → Clear & reschedule timeout
  ↓
setTimeout(updateHoveredElement, 1000/28)
  ↓
getElementAtPoint(x, y) → Find element
  ↓
lastHoveredElement.current !== newElement?
  ↓ (yes)
setHoveredElement(newElement)
  ↓
HoverHighlight renders
  ↓
useCyclicUpdate(updatePosition, 30) → Update overlay at 30 FPS
  ↓
User Clicks Element
  ↓
handleClick() → addSelectedElement(element)
  ↓
selectedElements array updated
  ↓
SelectedHighlight renders
  ↓
ElementChip renders in sidebar
  ↓
User Hovers Chip
  ↓
setChipHoveredElement(element)
  ↓
SelectedHighlight receives isChipHovered=true
  ↓
Border changes from green to blue
```

---

### 4.4 File Structure

```
/app/test-dashboard-stagewise/
├── page.tsx                                    # Main dashboard page
└── components/
    ├── DashboardContent.tsx                    # Charts, metrics, cards
    └── element-selector/
        ├── index.tsx                           # Main export
        ├── ElementSelectorProvider.tsx         # Context provider
        ├── SelectorOverlay.tsx                 # Mouse event capture
        ├── HoverHighlight.tsx                  # Blue preview overlay
        ├── SelectedHighlight.tsx               # Green/blue/red overlay
        ├── ContextSidebar.tsx                  # Right sidebar
        ├── ElementChip.tsx                     # Individual chip
        ├── SelectorControls.tsx                # Add to Context button
        └── hooks/
            └── useCyclicUpdate.ts              # RAF loop
        └── utils/
            └── elementDetection.ts             # Element finding logic
```

---

## 5. UI/UX Design

### 5.1 Visual States

#### State 1: Inactive (Default)
```
┌─────────────────────────────────────────────────────────┐
│ Dashboard                                               │
│                                                         │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐ │
│  │ Revenue      │  │ Users        │  │ Conversion   │ │
│  │ $45,234      │  │ 12,456       │  │ 3.2%         │ │
│  └──────────────┘  └──────────────┘  └──────────────┘ │
│                                                         │
│  ┌─────────────────────────────────────────────────┐  │
│  │                                                 │  │
│  │        [Bar Chart: Monthly Revenue]            │  │
│  │                                                 │  │
│  └─────────────────────────────────────────────────┘  │
│                                                         │
│  [Add to Context] ← Button visible                     │
└─────────────────────────────────────────────────────────┘
```

#### State 2: Selection Mode - Hovering
```
┌─────────────────────────────────────────────┬───────────┐
│ Dashboard                                   │ Context   │
│                                             │ ───────── │
│  ┌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┐  ┌──────────────┐        │ (0)       │
│  ╎ Revenue      ╎  │ Users        │        │           │
│  ╎ $45,234      ╎  │ 12,456       │        │ No        │
│  └╌╌╌╌╌╌╌╌╌╌╌╌╌╌┘  └──────────────┘        │ elements  │
│   ^ Blue hover                              │ selected  │
│                                             │           │
│  ┌─────────────────────────────────────┐   │           │
│  │                                     │   │           │
│  │    [Bar Chart: Monthly Revenue]    │   │           │
│  │                                     │   │           │
│  └─────────────────────────────────────┘   │           │
│                                             │           │
│  [Done]  ← Button changed                  │           │
│  ✕ (cursor: crosshair)                     │           │
└─────────────────────────────────────────────┴───────────┘
```

#### State 3: Elements Selected
```
┌─────────────────────────────────────────────┬───────────┐
│ Dashboard                                   │ Context   │
│                                             │ ───────── │
│  ┌──────────────┐  ┌──────────────┐        │ (2)       │
│  ├╌╌╌╌╌╌╌╌╌╌╌╌╌╌┤  ├╌╌╌╌╌╌╌╌╌╌╌╌╌╌┤        │           │
│  ╎ Revenue      ╎  ╎ Users        ╎        │ ┌───────┐ │
│  ╎ $45,234      ╎  ╎ 12,456       ╎        │ │ ▭ div │ │
│  └╌╌╌╌╌╌╌╌╌╌╌╌╌╌┘  └╌╌╌╌╌╌╌╌╌╌╌╌╌╌┘        │ │ Reven │ │
│   ^ Green selected                          │ │ ...  X│ │
│                                             │ └───────┘ │
│  ┌─────────────────────────────────────┐   │           │
│  │                                     │   │ ┌───────┐ │
│  │    [Bar Chart: Monthly Revenue]    │   │ │ ▭ div │ │
│  │                                     │   │ │ Users │ │
│  └─────────────────────────────────────┘   │ │ ...  X│ │
│                                             │ └───────┘ │
│  [Done]                                     │           │
└─────────────────────────────────────────────┴───────────┘
```

#### State 4: Chip Hover (Selection Mode Inactive)
```
┌─────────────────────────────────────────────┬───────────┐
│ Dashboard                                   │ Context   │
│                                             │ ───────── │
│  ┌──────────────┐  ┌──────────────┐        │ (2)       │
│  ├╌╌╌╌╌╌╌╌╌╌╌╌╌╌┤  │ Users        │        │           │
│  ╎ Revenue      ╎  │ 12,456       │        │ ┏━━━━━━━┓ │
│  ╎ $45,234      ╎  │              │        │ ┃ ▭ div ┃ │
│  └╌╌╌╌╌╌╌╌╌╌╌╌╌╌┘  └──────────────┘        │ ┃ Reven ┃ │
│   ^ Blue (chip hover)                       │ ┃ ...  X┃ │
│                                             │ ┗━━━━━━━┛ │
│  ┌─────────────────────────────────────┐   │  ^ hovered│
│  │                                     │   │           │
│  │    [Bar Chart: Monthly Revenue]    │   │ ┌───────┐ │
│  │                                     │   │ │ ▭ div │ │
│  └─────────────────────────────────────┘   │ │ Users │ │
│                                             │ │ ...  X│ │
│  [Add to Context]                           │ └───────┘ │
└─────────────────────────────────────────────┴───────────┘
```

### 5.2 Color Scheme

| State | Border Color | Background | Purpose |
|-------|-------------|------------|---------|
| Hover (Preview) | `#3b82f6` (blue-500/70) | `#3b82f6` (blue-500/5) | Indicates hoverable |
| Selected | `#22c55e` (green-500/70) | `#22c55e` (green-500/5) | Indicates selected |
| Chip Hover | `#3b82f6` (blue-500/70) | `#3b82f6` (blue-500/5) | Highlights active chip |
| Hover to Remove | `#ef4444` (red-500/70) | `#ef4444` (red-500/5) | Indicates removal |

### 5.3 Animations

| Element | Transition | Duration | Easing |
|---------|-----------|----------|--------|
| Overlay border color | `border-color` | 100ms | ease-in-out |
| Overlay background | `background-color` | 100ms | ease-in-out |
| Chip hover | `background-color` | 150ms | ease-in-out |
| Chip removal | `opacity, height` | 200ms | ease-out |
| Sidebar slide | `transform` | 300ms | cubic-bezier |

---

## 6. Implementation Phases

### Phase 1: Core Selection (MVP)
**Goal:** Basic hover and click selection with overlays
**Duration:** 4-6 hours

**Tasks:**
1. ✅ Create page `/app/test-dashboard-stagewise/page.tsx` with mock dashboard
2. ✅ Implement `ElementSelectorProvider` context
3. ✅ Build `SelectorOverlay` with velocity-based throttling
4. ✅ Create `HoverHighlight` component with 30 FPS updates
5. ✅ Create `SelectedHighlight` component
6. ✅ Implement `useCyclicUpdate` hook
7. ✅ Add basic button to toggle selection mode
8. ✅ Test on mock dashboard

**Acceptance Criteria:**
- ✓ Button toggles selection mode
- ✓ Hovering shows blue outline
- ✓ Clicking adds green outline
- ✓ No flickering or lag

---

### Phase 2: Context Sidebar
**Goal:** Display selected elements in sidebar
**Duration:** 3-4 hours

**Tasks:**
1. ✅ Create `ContextSidebar` component
2. ✅ Create `ElementChip` component
3. ✅ Implement chip removal functionality
4. ✅ Add empty state
5. ✅ Style sidebar to match dashboard
6. ✅ Make sidebar scrollable

**Acceptance Criteria:**
- ✓ Sidebar shows selected elements
- ✓ Chips display tag name and text preview
- ✓ X button removes elements
- ✓ Empty state shows when no selections

---

### Phase 3: Hover Synchronization
**Goal:** Sync chip hover with element highlights
**Duration:** 2-3 hours

**Tasks:**
1. ✅ Add `chipHoveredElement` to context
2. ✅ Implement chip hover handlers
3. ✅ Update `SelectedHighlight` to show blue on chip hover
4. ✅ Test bidirectional sync

**Acceptance Criteria:**
- ✓ Hovering chip highlights element in blue
- ✓ Works when selection mode inactive
- ✓ Returns to green on unhover

---

### Phase 4: Polish & Optimization
**Goal:** Smooth animations, keyboard shortcuts, edge cases
**Duration:** 3-4 hours

**Tasks:**
1. ✅ Add keyboard shortcuts (Ctrl+Alt+., Esc)
2. ✅ Implement hover-to-remove (red outline)
3. ✅ Add smooth transitions
4. ✅ Handle edge cases (element removed from DOM, scroll, resize)
5. ✅ Optimize performance (profiling, lazy loading)
6. ✅ Add loading states
7. ✅ Test in different browsers

**Acceptance Criteria:**
- ✓ Keyboard shortcuts work
- ✓ Smooth color transitions
- ✓ No console errors
- ✓ Works in Chrome, Firefox, Safari

---

### Phase 5: Advanced Features (Optional)
**Goal:** Enhanced UX and additional functionality
**Duration:** 4-6 hours

**Tasks:**
1. ⬜ Persist selection to localStorage
2. ⬜ Add element metadata (dimensions, position, attributes)
3. ⬜ Export selected context as JSON
4. ⬜ Add screenshot capture of selected elements
5. ⬜ Implement undo/redo
6. ⬜ Add element search/filter in sidebar
7. ⬜ Custom labels for elements
8. ⬜ Group elements by type

---

## 7. Testing Strategy

### 7.1 Unit Tests

**Test Coverage:**
- `useCyclicUpdate` hook behavior
- `getElementAtPoint` filtering logic
- Context state management (add, remove, hover)
- Velocity calculation accuracy

**Example Tests:**
```typescript
describe('useCyclicUpdate', () => {
  it('calls function at specified frame rate', () => {
    const mockFn = jest.fn();
    renderHook(() => useCyclicUpdate(mockFn, 30));
    // Advance time, verify call frequency
  });
});

describe('getElementAtPoint', () => {
  it('filters out selector overlay elements', () => {
    // Create DOM with overlay
    const element = getElementAtPoint(100, 100);
    expect(element).not.toHaveAttribute('data-selector-overlay');
  });
});
```

### 7.2 Integration Tests

**Scenarios:**
1. Click button → Selection mode activates
2. Hover element → Blue outline appears
3. Click element → Green outline appears, chip added to sidebar
4. Hover chip → Element highlights in blue
5. Click chip X → Element removed, outline disappears
6. Press Esc → Selection mode deactivates

**Example Test:**
```typescript
describe('Element Selection Flow', () => {
  it('completes full selection cycle', async () => {
    render(<ElementSelector />);

    // Activate selection mode
    const button = screen.getByText('Add to Context');
    await userEvent.click(button);
    expect(button).toHaveTextContent('Done');

    // Hover element
    const chart = screen.getByTestId('revenue-chart');
    await userEvent.hover(chart);
    expect(screen.getByTestId('hover-highlight')).toBeInTheDocument();

    // Click element
    await userEvent.click(chart);
    expect(screen.getByTestId('selected-highlight')).toBeInTheDocument();
    expect(screen.getByText('div')).toBeInTheDocument(); // Chip

    // Remove element
    const removeButton = screen.getByLabelText('Remove');
    await userEvent.click(removeButton);
    expect(screen.queryByTestId('selected-highlight')).not.toBeInTheDocument();
  });
});
```

### 7.3 Performance Tests

**Metrics to Track:**
- Frame rate during hover (should be ~30 FPS)
- Time to detect element (<35ms)
- Memory usage with 20+ selected elements
- Re-render count during mouse movement (should be minimal)

**Tools:**
- React DevTools Profiler
- Chrome Performance tab
- Lighthouse

### 7.4 Manual Testing Checklist

- [ ] Selection mode activates/deactivates smoothly
- [ ] Hover shows correct element (not nested SVG, not selector UI)
- [ ] Overlays track element position during scroll
- [ ] Overlays track element position during window resize
- [ ] Multiple selections work correctly
- [ ] Chip hover highlights correct element
- [ ] Removal works from both sidebar and selection mode
- [ ] Keyboard shortcuts work (Ctrl+Alt+., Esc)
- [ ] No console errors or warnings
- [ ] Works with different chart types (bar, line, pie, etc.)
- [ ] Works with text elements (headings, paragraphs)
- [ ] Works with cards and containers
- [ ] Cursor changes correctly (crosshair in selection mode)
- [ ] No flickering during fast mouse movement

---

## 8. Edge Cases & Error Handling

### 8.1 Edge Cases

| Scenario | Expected Behavior | Implementation |
|----------|-------------------|----------------|
| Element removed from DOM | Highlight disappears, chip remains with stale data | Track element in WeakMap, show "Element removed" in chip |
| Page scrolls | Highlights move with elements | Use `getBoundingClientRect()` which accounts for scroll |
| Window resizes | Highlights adjust to new positions | Update on window resize event |
| Fast mouse movement | No lag or stuttering | Velocity-based throttling delays updates |
| Hovering nested elements | Selects most specific element | Use `elementsFromPoint()` with filtering |
| Hovering SVG charts | Selects parent container, not SVG internals | Filter out elements with `closest('svg')` |
| Selection mode activated multiple times | Clears previous state or maintains it | Maintain state (selection persists) |
| Many elements selected (50+) | Sidebar scrollable, no performance degradation | Virtualized list if >50 elements |

### 8.2 Error Handling

```typescript
// Protect against invalid coordinates
export function getElementAtPoint(x: number, y: number): HTMLElement {
  if (!Number.isFinite(x) || !Number.isFinite(y)) {
    console.warn('Invalid coordinates:', { x, y });
    return document.body;
  }
  // ... rest of implementation
}

// Protect against missing elements
const updatePosition = useCallback(() => {
  try {
    if (boxRef.current && element) {
      const rect = element.getBoundingClientRect();
      boxRef.current.style.top = `${rect.top - 2}px`;
      // ...
    }
  } catch (error) {
    console.error('Error updating highlight position:', error);
  }
}, [element]);
```

---

## 9. Acceptance Criteria

### Must-Have (P0)

- [ ] "Add to Context" button visible on dashboard
- [ ] Clicking button activates selection mode (cursor: crosshair)
- [ ] Hovering elements shows blue dotted outline with tag name
- [ ] Clicking elements adds green dotted outline
- [ ] Selected elements appear as chips in right sidebar
- [ ] Chips show element tag name and text preview
- [ ] Clicking X on chip removes element from selection
- [ ] Hovering chip highlights corresponding element in blue
- [ ] "Done" button deactivates selection mode
- [ ] No flickering or lag during mouse movement
- [ ] Works with charts, cards, text, and containers

### Should-Have (P1)

- [ ] Keyboard shortcut Ctrl+Alt+. toggles selection mode
- [ ] Esc key exits selection mode
- [ ] Hovering selected element in selection mode shows red outline
- [ ] Clicking red-outlined element removes it
- [ ] Smooth color transitions (100ms)
- [ ] Sidebar shows count: "Context (3)"
- [ ] Sidebar scrollable when many elements
- [ ] Empty state when no selections
- [ ] Works during scroll and resize

### Nice-to-Have (P2)

- [ ] Selection persists across page navigation
- [ ] Export context as JSON
- [ ] Custom labels for elements
- [ ] Element metadata in sidebar (dimensions, position)
- [ ] Undo/redo functionality
- [ ] Mobile responsive

---

## 10. Success Metrics

### Quantitative Metrics

| Metric | Target | Measurement |
|--------|--------|-------------|
| Selection Time | <5s for 5 elements | User testing |
| Frame Rate | 30 FPS | Chrome DevTools Performance |
| Element Detection Accuracy | 95%+ | Manual testing with various elements |
| Page Load Impact | <100ms | Lighthouse |
| Time to Interactive | <3s | Lighthouse |
| Memory Usage | <50MB for 20 elements | Chrome Task Manager |

### Qualitative Metrics

- User feedback: "Feels smooth and responsive"
- No reported flickering issues
- Intuitive UI (users understand without instructions)
- Works reliably across different dashboard layouts

---

## 11. Dependencies

### Technical Dependencies

- **Next.js** 13+ (App Router)
- **React** 18+
- **TypeScript** 5+
- **Tailwind CSS** 3+
- **lucide-react** (for icons)

### New Packages (if needed)

```json
{
  "dependencies": {
    "clsx": "^2.0.0",           // Conditional classNames
    "tailwind-merge": "^2.0.0"  // Merge Tailwind classes
  }
}
```

---

## 12. Risks & Mitigations

| Risk | Impact | Probability | Mitigation |
|------|--------|-------------|------------|
| Performance issues with complex charts | High | Medium | Profile early, optimize hot paths, lazy load |
| Browser compatibility issues | Medium | Low | Test in all major browsers, use polyfills |
| Conflicts with existing page interactions | High | Medium | Use high z-index, careful event handling |
| Element detection inaccuracy | High | Medium | Thorough testing with various element types |
| State management complexity | Medium | Low | Keep state simple, use established patterns |

---

## 13. Future Enhancements

### Version 2.0
- Persistent selection across sessions (localStorage)
- Export context as JSON/CSV
- Custom labels and notes for elements
- Screenshot capture of selected elements
- Element grouping and categorization

### Version 3.0
- AI-powered element suggestions ("You might want to select these charts")
- Collaborative selection (multi-user)
- Integration with AI chat (send context to Claude API)
- Advanced filtering and search in sidebar
- Custom selection templates ("Select all revenue metrics")

---

## 14. Appendix

### A. Reference Implementation

See `stagewise-selector-analysis.md` for detailed analysis of stagewise's implementation.

### B. Key Learnings from stagewise

1. **Velocity-based throttling** is crucial for smooth performance
2. **Ref-based position updates** avoid React re-render overhead
3. **Fixed 30 FPS** is perfect balance of smoothness and efficiency
4. **Simple fixed-position overlays** are better than complex Canvas/SVG
5. **Duplicate element checking** prevents unnecessary state updates

### C. Glossary

- **Selection Mode:** State where user can click elements to add to context
- **Hover Highlight:** Blue dotted overlay shown during hover
- **Selected Highlight:** Green/blue/red dotted overlay for selected elements
- **Context Sidebar:** Right panel showing selected elements as chips
- **Element Chip:** Small card representing a selected element
- **Chip Hover:** Hovering over chip in sidebar to highlight element
- **Velocity Throttling:** Delaying updates when mouse moves fast
- **RAF:** requestAnimationFrame, browser API for smooth animations
- **Ref-based Update:** Directly manipulating DOM via refs, bypassing React

---

## 15. Sign-off

### Stakeholders

| Role | Name | Sign-off | Date |
|------|------|----------|------|
| Product Owner | TBD | [ ] | |
| Tech Lead | TBD | [ ] | |
| UX Designer | TBD | [ ] | |
| QA Lead | TBD | [ ] | |

### Approval

This PRD is approved for implementation once all stakeholders have signed off.

---

**Document Version:** 1.0
**Last Updated:** 2025-11-12
**Next Review:** After Phase 1 completion
