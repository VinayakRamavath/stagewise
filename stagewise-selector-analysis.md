# stagewise Element Selector Analysis

## Executive Summary

stagewise achieves smooth, flicker-free element selection through a combination of:
- **Velocity-based throttling** for mouse tracking
- **Fixed 30 FPS updates** via requestAnimationFrame
- **Direct DOM manipulation** using refs (avoiding React re-renders)
- **Simple fixed-position overlays** with inline styles
- **React Context** for state synchronization between components

The implementation is remarkably simple yet effective, focusing on performance optimization at the event handling level rather than complex rendering strategies.

---

## 1. Architecture Overview

### High-Level Flow

```
User activates selector mode
         ↓
Full-screen overlay captures mouse events
         ↓
Mouse movement → Velocity calculation
         ↓
Throttled updates (28ms intervals, delayed if velocity > 30px/s)
         ↓
getElementAtPoint() detects element under cursor
         ↓
Blue highlight rendered at 30 FPS (HoveredItem)
         ↓
Click → Add to selection array
         ↓
Grey highlights rendered for all selected (SelectedItem)
         ↓
Chips displayed in toolbar (bidirectional hover sync)
```

### Key Design Decisions

1. **Full-screen transparent overlay** - Intercepts all mouse events
2. **Separate components for hover vs selected** - Clear separation of concerns
3. **Ref-based positioning** - Updates happen via `boxRef.current.style.*` (no React re-renders)
4. **Fixed frame rate** - 30 FPS via custom `useCyclicUpdate` hook
5. **Velocity-based throttling** - Delays updates during fast mouse movement
6. **React Context for state** - Global state without prop drilling

### Technology Stack

- **React** for component structure
- **Vanilla JS DOM manipulation** for positioning (refs)
- **requestAnimationFrame** for smooth updates
- **TailwindCSS** for styling (dotted borders, transitions)
- **No Canvas/SVG** - Pure DOM elements with fixed positioning

---

## 2. Key Files & Their Roles

### Core Components

#### `/toolbar/core/src/components/dom-context-selector/selector-canvas.tsx` (178 lines)
**The Main Orchestrator**

**Purpose:** Full-screen overlay that manages the entire selection flow

**Key Responsibilities:**
- Renders transparent full-screen div
- Tracks mouse position and velocity
- Throttles element detection updates
- Manages hover state
- Handles click events to add/remove selections
- Renders both `HoveredItem` and `SelectedItem` components

**Key Functions:**
```typescript
updateHoveredElement() // Detects element at mouse position
handleMouseMove()      // Calculates velocity, schedules updates
handleMouseClick()     // Adds element to selection
```

**Performance Techniques:**
- Velocity calculation: `distance / deltaTime * 1000` pixels/second
- Throttling: Updates scheduled at `1000 / 28` ms (~35 FPS)
- Velocity threshold: Delays update if velocity > 30 px/s
- Ref-based tracking: `lastHoveredElement.current` avoids duplicate updates

---

#### `/toolbar/core/src/components/dom-context-selector/hovered-item.tsx` (82 lines)
**Blue Dotted Preview Overlay**

**Purpose:** Shows blue dotted border around element under cursor

**Key Features:**
- Blue dotted border (`border-blue-600/70`)
- Semi-transparent background (`bg-blue-600/5`)
- Tag name label in top-left corner
- Plugin annotations displayed
- Updates at 30 FPS

**Positioning Logic:**
```typescript
const updateBoxPosition = useCallback(() => {
  if (boxRef.current && refElement) {
    const referenceRect = refElement.getBoundingClientRect();
    boxRef.current.style.top = `${referenceRect.top - 2}px`;
    boxRef.current.style.left = `${referenceRect.left - 2}px`;
    boxRef.current.style.width = `${referenceRect.width + 4}px`;
    boxRef.current.style.height = `${referenceRect.height + 4}px`;
  }
}, [refElement]);

useCyclicUpdate(updateBoxPosition, 30); // 30 FPS
```

**CSS Classes:**
```css
fixed z-10 rounded-sm
border-2 border-blue-600/70 border-dotted
bg-blue-600/5
transition-all duration-100
```

---

#### `/toolbar/core/src/components/dom-context-selector/selected-item.tsx` (54 lines)
**Grey Dotted Overlay for Selected Elements**

**Purpose:** Shows grey dotted border around selected elements, changes to blue when chip hovered, red on hover

**Key Features:**
- Grey dotted border (`border-zinc-600/70`)
- Changes to blue when corresponding chip is hovered (`isChipHovered` prop)
- Changes to red on hover (`hover:border-rose-600/70`)
- Clickable to remove from selection
- Updates at 30 FPS

**State-Dependent Styling:**
```typescript
<button
  className={cn(
    'fixed border-2 border-zinc-600/70 border-dotted',
    'hover:border-rose-600/70 hover:bg-rose-600/5',
    isChipHovered && 'border-blue-600/70 bg-blue-600/5'
  )}
  onClick={props.onRemoveClick}
/>
```

---

### Performance Optimization Hooks

#### `/toolbar/core/src/hooks/use-cyclic-update.tsx` (42 lines)
**requestAnimationFrame Loop with Frame Rate Control**

**Purpose:** Calls a function cyclically at a specified frame rate

**How It Works:**
```typescript
export function useCyclicUpdate(func: () => void, frameRate?: number) {
  const timeBetweenFrames = frameRate ? 1000 / frameRate : 0;
  const lastCallFrameTime = useRef<number>(0);

  const update = useCallback((frameTime: number) => {
    // Only call func if enough time has passed
    if (frameTime - lastCallFrameTime.current >= timeBetweenFrames) {
      func();
      lastCallFrameTime.current = frameTime;
    }
    // Schedule next frame
    animationFrameHandle.current = requestAnimationFrame(update);
  }, [func, timeBetweenFrames]);

  useEffect(() => {
    animationFrameHandle.current = requestAnimationFrame(update);
    return () => cancelAnimationFrame(animationFrameHandle.current);
  }, [update]);
}
```

**Usage Pattern:**
```typescript
useCyclicUpdate(updateBoxPosition, 30); // Caps at 30 FPS
```

**Why This Works:**
- Uses RAF for smoothness
- Throttles to 30 FPS to reduce CPU usage
- Auto-cleanup on unmount
- Timestamp-based throttling (more accurate than setTimeout)

---

### State Management Hooks

#### `/toolbar/core/src/hooks/use-chat-state.tsx` (362 lines)
**Central State Management**

**Purpose:** Global state for selection mode and selected elements

**State Structure:**
```typescript
interface ChatContext {
  // Selection state
  domContextElements: {
    element: HTMLElement;
    pluginContext: any[];
  }[];
  isContextSelectorActive: boolean;

  // Actions
  addChatDomContext: (element: HTMLElement) => void;
  removeChatDomContext: (element: HTMLElement) => void;
  startContextSelector: () => void;
  stopContextSelector: () => void;
}
```

**Key Behaviors:**
- Auto-stops selector when chat closes or toolbar minimizes
- Integrates with plugin system
- Collects element metadata when sending messages
- Clears selection after message sent

---

#### `/toolbar/core/src/hooks/use-context-chip-hover.tsx` (59 lines)
**Synchronizes Hover Between Chips and Overlays**

**Purpose:** When you hover a chip in the toolbar, the corresponding overlay highlights in blue

**How It Works:**
```typescript
interface ContextChipHoverState {
  hoveredElement: HTMLElement | null;
  setHoveredElement: (element: HTMLElement | null) => void;
}

// Used in chip component:
<button
  onMouseEnter={() => setHoveredElement(element)}
  onMouseLeave={() => setHoveredElement(null)}
/>

// Used in overlay component:
<SelectedItem
  isChipHovered={hoveredElement === element}
/>
```

**Cleanup Logic:**
- Auto-clears hover state if element is removed from selection

---

### Utility Functions

#### `/toolbar/core/src/utils.tsx` (607 lines)
**Core Utility Functions**

**Key Functions:**

1. **`getElementAtPoint(x, y)`** - Element detection
```typescript
export function getElementAtPoint(x: number, y: number) {
  const elementsBelowAnnotation =
    iframeWindow.document.elementsFromPoint(x, y);

  // Filter out SVG, toolbar, and elements not at point
  const refElement = elementsBelowAnnotation.find(element =>
    !element.closest('svg') &&
    !element.closest('STAGEWISE-TOOLBAR') &&
    isElementAtPoint(element as HTMLElement, x, y)
  ) || document.body;

  return refElement;
}
```

2. **`getXPathForElement(element, useId)`** - Element identification
   - Generates XPath for element
   - Used as React key for selected items
   - Optionally uses element ID for shorter path

3. **`getSelectedElementInfo(element)`** - Element metadata extraction
   - Collects attributes, properties, bounding rect
   - Recursively includes parent hierarchy (up to 10 levels)
   - Truncates data to prevent schema violations

4. **`isElementAtPoint(element, x, y)`** - Validates element is at coordinates
   - Checks if point is within bounding rect
   - Used to filter false positives from `elementsFromPoint`

---

### Supporting Components

#### `/toolbar/core/src/components/context-elements-chips-flexible.tsx` (104 lines)
**Interactive Chips in Toolbar**

**Purpose:** Displays chips for selected elements with hover sync

**Features:**
- Shows tag name or plugin annotation
- X button to remove
- Hover highlights corresponding overlay
- Icon from plugin if available

---

## 3. Core Implementation Patterns

### Pattern 1: Velocity-Based Throttling

**The Problem:** Mouse events fire very frequently, causing performance issues

**The Solution:**
```typescript
// selector-canvas.tsx:93-126
const handleMouseMove = useCallback<MouseEventHandler>((event) => {
  const currentTimestamp = performance.now();

  // Calculate velocity
  const deltaX = event.clientX - (mouseState.current?.lastX ?? event.clientX);
  const deltaY = event.clientY - (mouseState.current?.lastY ?? event.clientY);
  const deltaTime = currentTimestamp - (mouseState.current?.lastTimestamp ?? currentTimestamp);
  const distance = Math.hypot(deltaX, deltaY);

  mouseState.current = {
    lastX: event.clientX,
    lastY: event.clientY,
    velocity: deltaTime > 0 ? (distance / deltaTime) * 1000 : 0,
    lastTimestamp: currentTimestamp
  };

  // Delay update if moving fast
  if (mouseState.current.velocity > 30) {
    if (nextUpdateTimeout.current) {
      clearTimeout(nextUpdateTimeout.current);
    }
    nextUpdateTimeout.current = setTimeout(updateHoveredElement, 1000 / 28);
  } else if (!nextUpdateTimeout.current) {
    nextUpdateTimeout.current = setTimeout(updateHoveredElement, 1000 / 28);
  }
}, [updateHoveredElement]);
```

**Why This Works:**
- Reduces unnecessary element lookups during fast mouse movement
- Uses setTimeout, not RAF (simpler scheduling)
- ~28 FPS for element detection (1000/28 ≈ 35ms)
- Only schedules update if none pending
- Clears pending timeout when velocity exceeds threshold

---

### Pattern 2: Ref-Based Position Updates

**The Problem:** Re-rendering React components on every frame is expensive

**The Solution:**
```typescript
// hovered-item.tsx:31-47
const boxRef = useRef<HTMLDivElement>(null);

const updateBoxPosition = useCallback(() => {
  if (boxRef.current && refElement) {
    const referenceRect = refElement.getBoundingClientRect();

    // Direct DOM manipulation - no React re-render!
    boxRef.current.style.top = `${referenceRect.top - 2}px`;
    boxRef.current.style.left = `${referenceRect.left - 2}px`;
    boxRef.current.style.width = `${referenceRect.width + 4}px`;
    boxRef.current.style.height = `${referenceRect.height + 4}px`;
  }
}, [refElement]);

useCyclicUpdate(updateBoxPosition, 30); // Called at 30 FPS
```

**Why This Works:**
- Bypasses React's reconciliation entirely
- Updates DOM directly via ref
- Only re-renders when `refElement` prop changes
- Position updates happen at 30 FPS without triggering React

---

### Pattern 3: Fixed Frame Rate with RAF

**The Problem:** Need smooth updates but not at full 60 FPS (wasteful)

**The Solution:**
```typescript
// use-cyclic-update.tsx:7-41
export function useCyclicUpdate(func: () => void, frameRate?: number) {
  const animationFrameHandle = useRef<number>();
  const timeBetweenFrames = frameRate ? 1000 / frameRate : 0;
  const lastCallFrameTime = useRef<number>(0);

  const update = useCallback((frameTime: number) => {
    // Throttle to target frame rate
    if (frameTime - lastCallFrameTime.current >= timeBetweenFrames) {
      func();
      lastCallFrameTime.current = frameTime;
    }

    // Always schedule next frame
    animationFrameHandle.current = requestAnimationFrame(update);
  }, [func, timeBetweenFrames]);

  useEffect(() => {
    animationFrameHandle.current = requestAnimationFrame(update);
    return () => cancelAnimationFrame(animationFrameHandle.current);
  }, [update]);
}
```

**Why This Works:**
- RAF provides smooth timing, aligned with browser repaints
- Timestamp-based throttling caps frame rate
- 30 FPS is sweet spot (smooth enough, efficient enough)
- Auto-cleanup on unmount prevents memory leaks

---

### Pattern 4: Duplicate Update Prevention

**The Problem:** Don't want to update state if hovering same element

**The Solution:**
```typescript
// selector-canvas.tsx:58-87
const lastHoveredElement = useRef<HTMLElement | null>(null);

const updateHoveredElement = useCallback(() => {
  if (!mouseState.current) return;

  const refElement = getElementAtPoint(
    mouseState.current.lastX,
    mouseState.current.lastY
  );

  // Skip if already selected
  if (selectedItems.includes(refElement)) {
    setHoversAddable(false);
    lastHoveredElement.current = null;
    setHoveredElement(null);
    return;
  }

  // Only update if element changed
  if (lastHoveredElement.current !== refElement) {
    lastHoveredElement.current = refElement;
    setHoveredElement(refElement);
    setHoversAddable(true);
  }
}, [selectedItems]);
```

**Why This Works:**
- Ref comparison is cheaper than state update
- Prevents unnecessary React re-renders
- Also prevents hovering already-selected elements

---

### Pattern 5: Overlay Rendering Strategy

**The Problem:** Need overlays that track moving/resizing elements

**The Solution:**
```typescript
// Fixed positioning with inline styles
<div
  ref={boxRef}
  className="fixed z-10 border-2 border-blue-600/70 border-dotted
             bg-blue-600/5 transition-all duration-100"
>
  {/* Label in top-left corner */}
  <div className="absolute top-0.5 left-0.5">
    <div className="bg-zinc-700/80 px-1 py-0 rounded-md">
      <span>{refElement.tagName.toLowerCase()}</span>
    </div>
  </div>
</div>
```

**CSS Choices:**
- `fixed` positioning - relative to viewport
- `z-10` - above app content, below toolbar
- `border-dotted` - clear visual feedback
- `bg-blue-600/5` - subtle fill
- `transition-all duration-100` - smooth color changes
- Inline styles for position/size (updated via ref)

**Why This Works:**
- Fixed positioning works across all layouts
- Tailwind classes for styling (no re-renders)
- Inline styles for dynamic values (updated via ref)
- Simple DOM elements, no Canvas/SVG overhead

---

## 4. Performance Secrets

### Secret #1: Two-Tier Update Strategy

**Element Detection: ~28 FPS**
```typescript
// selector-canvas.tsx
setTimeout(updateHoveredElement, 1000 / 28) // ~35ms
```

**Overlay Positioning: 30 FPS**
```typescript
// hovered-item.tsx
useCyclicUpdate(updateBoxPosition, 30) // ~33ms
```

**Why Different Rates?**
- Element detection (DOM query) is more expensive
- Position updates are cheap (just getBoundingClientRect)
- Decoupling allows independent optimization

---

### Secret #2: Velocity-Based Delay

```typescript
if (mouseState.current.velocity > 30) {
  clearTimeout(nextUpdateTimeout.current);
  nextUpdateTimeout.current = setTimeout(updateHoveredElement, 1000 / 28);
}
```

**Effect:**
- During fast mouse movement, detection is delayed
- Reduces wasted CPU on elements user is skipping over
- Once mouse slows down, detection resumes

**Result:** Feels responsive without burning CPU

---

### Secret #3: Ref-Based Updates

```typescript
boxRef.current.style.top = `${top}px`;    // ✅ Direct DOM
setPosition({ top, left });               // ❌ React re-render
```

**Performance Gain:**
- Avoids React reconciliation
- Avoids virtual DOM diffing
- Avoids re-rendering child components
- Direct DOM updates are ~10x faster for animations

---

### Secret #4: Smart Element Filtering

```typescript
const elementsBelowAnnotation = document.elementsFromPoint(x, y);
const refElement = elementsBelowAnnotation.find(element =>
  !element.closest('svg') &&              // Skip SVG internals
  !element.closest('STAGEWISE-TOOLBAR') && // Skip own toolbar
  isElementAtPoint(element, x, y)          // Double-check bounds
);
```

**Why This Matters:**
- `elementsFromPoint` returns ALL elements (including nested SVG)
- Filtering reduces false positives
- Double-checking bounds catches edge cases

---

### Secret #5: Fixed Positioning

```css
.overlay {
  position: fixed; /* ✅ Relative to viewport */
  top: 42px;       /* ✅ Direct coordinates */
  left: 100px;
}

/* NOT: */
.overlay {
  position: absolute; /* ❌ Relative to parent */
  transform: translate(100px, 42px); /* ❌ Extra reflow */
}
```

**Why Fixed?**
- No need to account for scroll position
- No parent stacking context issues
- Simpler calculations
- Consistent across all layouts

---

### Secret #6: Minimal React Re-renders

**When Overlay Component Re-renders:**
1. When `refElement` prop changes (new element hovered)
2. When `isChipHovered` prop changes
3. When window size changes

**When Overlay Does NOT Re-render:**
1. ✅ When element moves (ref updates handle it)
2. ✅ When element resizes (ref updates handle it)
3. ✅ Every animation frame (capped at 30 FPS, ref-based)

**How They Achieve This:**
- Position updates via ref in effect
- No position/size in React state
- `useCallback` for update functions
- Stable dependencies

---

## 5. Recommendations for Our Project

### ✅ What to Adopt

#### 1. **Velocity-Based Throttling**
```typescript
// Essential pattern for smooth mouse tracking
const velocity = (distance / deltaTime) * 1000;
if (velocity > 30) {
  // Delay update
}
```
**Why:** Prevents unnecessary work during fast mouse movement

---

#### 2. **Ref-Based Position Updates**
```typescript
// Update overlay position without React re-renders
const updatePosition = () => {
  boxRef.current.style.top = `${rect.top}px`;
  boxRef.current.style.left = `${rect.left}px`;
};
useCyclicUpdate(updatePosition, 30);
```
**Why:** Dramatically reduces rendering overhead

---

#### 3. **Fixed 30 FPS Update Loop**
```typescript
// Custom hook for frame-rate-limited updates
useCyclicUpdate(updateFunction, 30);
```
**Why:** Perfect balance of smoothness and performance

---

#### 4. **Simple Fixed-Position Overlays**
```tsx
<div
  ref={boxRef}
  className="fixed border-2 border-blue-500/70 border-dotted"
/>
```
**Why:** Simplest solution that works across all layouts

---

#### 5. **Duplicate Element Check**
```typescript
const lastHoveredElement = useRef<HTMLElement | null>(null);
if (lastHoveredElement.current !== newElement) {
  // Only update if changed
}
```
**Why:** Prevents unnecessary state updates

---

### ❌ What to Skip/Modify

#### 1. **Plugin System Integration**
- stagewise has extensive plugin hooks
- We don't need this complexity
- **Alternative:** Simple context object with metadata

---

#### 2. **iframe Window Handling**
```typescript
// stagewise operates in iframe
const iframeWindow = getIFrameWindow();
```
- We're running directly in the dashboard
- **Alternative:** Use `window` and `document` directly

---

#### 3. **XPath Element Identification**
```typescript
// stagewise uses XPath for element tracking
const xpath = getXPathForElement(element);
```
- Overkill for runtime-only selection
- **Alternative:** Use `WeakMap` or simple index

---

#### 4. **Complex Metadata Collection**
```typescript
// stagewise collects extensive element metadata
export const getSelectedElementInfo = (element) => {
  // 100+ lines of metadata extraction
}
```
- We only need basic info for context
- **Alternative:** Collect `tagName`, `textContent`, `boundingRect`, custom attributes

---

#### 5. **Auto-Stop on App State Changes**
- stagewise stops selector when chat closes/minimizes
- Dashboard might want persistent selection
- **Alternative:** User-controlled on/off toggle

---

### 🔄 What to Adapt

#### 1. **Element Filtering**
```typescript
// stagewise filters SVG and toolbar
const element = elementsFromPoint(x, y).find(el =>
  !el.closest('svg') &&
  !el.closest('STAGEWISE-TOOLBAR')
);

// We should filter our own UI
const element = elementsFromPoint(x, y).find(el =>
  !el.closest('[data-selector-ui]') &&
  !el.closest('[data-dashboard-chrome]')
);
```

---

#### 2. **Visual Style**
```css
/* stagewise: blue for hover, grey for selected */
.hovered { border: 2px dotted rgb(59 130 246 / 0.7); }
.selected { border: 2px dotted rgb(82 82 91 / 0.7); }

/* We might want: */
.hovered { border: 2px solid rgb(59 130 246); }
.selected { border: 2px solid rgb(34 197 94); }
```

---

#### 3. **Tooltip Content**
```tsx
// stagewise shows: tag name + plugin annotations
<span>{tagName}</span>

// We might show: tag name + text preview + custom label
<span>{tagName}</span>
<span className="text-xs opacity-60">
  {textContent.slice(0, 30)}...
</span>
```

---

## 6. Minimal Implementation Blueprint

### Step 1: Core Hook (`useElementSelector.ts`)

```typescript
import { useState, useRef, useCallback } from 'react';

interface SelectedElement {
  element: HTMLElement;
  id: string;
  label: string;
}

export function useElementSelector() {
  const [isActive, setIsActive] = useState(false);
  const [hoveredElement, setHoveredElement] = useState<HTMLElement | null>(null);
  const [selectedElements, setSelectedElements] = useState<SelectedElement[]>([]);

  const lastHoveredElement = useRef<HTMLElement | null>(null);
  const mouseState = useRef<{
    lastX: number;
    lastY: number;
    velocity: number;
    lastTimestamp: number;
  } | null>(null);
  const nextUpdateTimeout = useRef<NodeJS.Timeout | null>(null);

  // Get element at point, filtering our own UI
  const getElementAtPoint = useCallback((x: number, y: number) => {
    const elements = document.elementsFromPoint(x, y);
    return elements.find(el =>
      !el.closest('[data-selector-overlay]') &&
      !el.closest('[data-selector-controls]')
    ) as HTMLElement || document.body;
  }, []);

  // Update hovered element
  const updateHoveredElement = useCallback(() => {
    if (!mouseState.current) return;

    const element = getElementAtPoint(
      mouseState.current.lastX,
      mouseState.current.lastY
    );

    // Skip if already selected
    if (selectedElements.some(s => s.element === element)) {
      lastHoveredElement.current = null;
      setHoveredElement(null);
      return;
    }

    // Only update if changed
    if (lastHoveredElement.current !== element) {
      lastHoveredElement.current = element;
      setHoveredElement(element);
    }
  }, [selectedElements, getElementAtPoint]);

  // Mouse move handler with velocity throttling
  const handleMouseMove = useCallback((event: React.MouseEvent) => {
    const currentTimestamp = performance.now();

    const deltaX = event.clientX - (mouseState.current?.lastX ?? event.clientX);
    const deltaY = event.clientY - (mouseState.current?.lastY ?? event.clientY);
    const deltaTime = currentTimestamp - (mouseState.current?.lastTimestamp ?? currentTimestamp);
    const distance = Math.hypot(deltaX, deltaY);

    mouseState.current = {
      lastX: event.clientX,
      lastY: event.clientY,
      velocity: deltaTime > 0 ? (distance / deltaTime) * 1000 : 0,
      lastTimestamp: currentTimestamp
    };

    // Throttle based on velocity
    if (mouseState.current.velocity > 30) {
      if (nextUpdateTimeout.current) {
        clearTimeout(nextUpdateTimeout.current);
      }
      nextUpdateTimeout.current = setTimeout(updateHoveredElement, 1000 / 28);
    } else if (!nextUpdateTimeout.current) {
      nextUpdateTimeout.current = setTimeout(updateHoveredElement, 1000 / 28);
    }
  }, [updateHoveredElement]);

  // Click handler
  const handleClick = useCallback((event: React.MouseEvent) => {
    event.preventDefault();
    event.stopPropagation();

    if (!lastHoveredElement.current) return;
    if (selectedElements.some(s => s.element === lastHoveredElement.current)) return;

    const element = lastHoveredElement.current;
    setSelectedElements(prev => [...prev, {
      element,
      id: Math.random().toString(36).slice(2),
      label: element.tagName.toLowerCase()
    }]);
  }, [selectedElements]);

  // Remove element
  const removeElement = useCallback((id: string) => {
    setSelectedElements(prev => prev.filter(s => s.id !== id));
  }, []);

  // Toggle active state
  const toggle = useCallback(() => {
    setIsActive(prev => !prev);
    if (isActive) {
      setHoveredElement(null);
      lastHoveredElement.current = null;
    }
  }, [isActive]);

  return {
    isActive,
    hoveredElement,
    selectedElements,
    handleMouseMove,
    handleClick,
    removeElement,
    toggle,
    setIsActive
  };
}
```

---

### Step 2: Cyclic Update Hook (`useCyclicUpdate.ts`)

```typescript
import { useEffect, useRef, useCallback } from 'react';

export function useCyclicUpdate(func: () => void, frameRate?: number) {
  const animationFrameHandle = useRef<number>();
  const timeBetweenFrames = frameRate && frameRate > 0 ? 1000 / frameRate : 0;
  const lastCallFrameTime = useRef<number>(0);

  const update = useCallback((frameTime: number) => {
    if (frameTime - lastCallFrameTime.current >= timeBetweenFrames) {
      func();
      lastCallFrameTime.current = frameTime;
    }
    animationFrameHandle.current = requestAnimationFrame(update);
  }, [func, timeBetweenFrames]);

  useEffect(() => {
    if (!frameRate || frameRate > 0) {
      animationFrameHandle.current = requestAnimationFrame(update);
    }
    return () => {
      if (animationFrameHandle.current) {
        cancelAnimationFrame(animationFrameHandle.current);
      }
    };
  }, [frameRate, update]);
}
```

---

### Step 3: Hover Overlay Component (`HoverOverlay.tsx`)

```typescript
import { useRef, useCallback } from 'react';
import { useCyclicUpdate } from './useCyclicUpdate';

interface HoverOverlayProps {
  element: HTMLElement;
}

export function HoverOverlay({ element }: HoverOverlayProps) {
  const boxRef = useRef<HTMLDivElement>(null);

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

  return (
    <div
      ref={boxRef}
      data-selector-overlay
      className="fixed z-50 pointer-events-none
                 border-2 border-blue-500/70 border-dotted
                 bg-blue-500/5 rounded-sm
                 transition-colors duration-100"
    >
      <div className="absolute top-0.5 left-0.5
                      bg-zinc-800/90 text-white text-xs
                      px-2 py-0.5 rounded">
        {element.tagName.toLowerCase()}
      </div>
    </div>
  );
}
```

---

### Step 4: Selected Overlay Component (`SelectedOverlay.tsx`)

```typescript
import { useRef, useCallback } from 'react';
import { useCyclicUpdate } from './useCyclicUpdate';

interface SelectedOverlayProps {
  element: HTMLElement;
  onRemove: () => void;
}

export function SelectedOverlay({ element, onRemove }: SelectedOverlayProps) {
  const boxRef = useRef<HTMLButtonElement>(null);

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

  return (
    <button
      ref={boxRef}
      data-selector-overlay
      onClick={onRemove}
      className="fixed z-50
                 border-2 border-green-500/70 border-dotted
                 bg-green-500/5 rounded-sm
                 hover:border-red-500/70 hover:bg-red-500/5
                 transition-colors duration-100
                 cursor-pointer"
    />
  );
}
```

---

### Step 5: Main Selector Component (`ElementSelector.tsx`)

```typescript
import { useElementSelector } from './useElementSelector';
import { HoverOverlay } from './HoverOverlay';
import { SelectedOverlay } from './SelectedOverlay';

export function ElementSelector() {
  const {
    isActive,
    hoveredElement,
    selectedElements,
    handleMouseMove,
    handleClick,
    removeElement
  } = useElementSelector();

  if (!isActive) return null;

  return (
    <>
      {/* Full-screen overlay */}
      <div
        data-selector-overlay
        className="fixed inset-0 z-40 cursor-crosshair"
        onMouseMove={handleMouseMove}
        onClick={handleClick}
      />

      {/* Hover overlay */}
      {hoveredElement && (
        <HoverOverlay element={hoveredElement} />
      )}

      {/* Selected overlays */}
      {selectedElements.map(({ element, id }) => (
        <SelectedOverlay
          key={id}
          element={element}
          onRemove={() => removeElement(id)}
        />
      ))}
    </>
  );
}
```

---

### Step 6: Controls Component (`SelectorControls.tsx`)

```typescript
import { useElementSelector } from './useElementSelector';

export function SelectorControls() {
  const { isActive, selectedElements, toggle } = useElementSelector();

  return (
    <div data-selector-controls className="fixed top-4 right-4 z-50">
      <button
        onClick={toggle}
        className="px-4 py-2 bg-blue-500 text-white rounded-lg
                   hover:bg-blue-600 transition-colors"
      >
        {isActive ? 'Done' : 'Select Elements'}
      </button>

      {selectedElements.length > 0 && (
        <div className="mt-2 bg-white rounded-lg shadow-lg p-2">
          <div className="text-sm font-medium mb-1">
            Selected: {selectedElements.length}
          </div>
          {selectedElements.map(({ id, label }) => (
            <div key={id} className="text-xs text-gray-600">
              {label}
            </div>
          ))}
        </div>
      )}
    </div>
  );
}
```

---

### Step 7: Usage in App

```typescript
import { ElementSelector } from './components/ElementSelector';
import { SelectorControls } from './components/SelectorControls';

function App() {
  return (
    <>
      {/* Your dashboard content */}
      <div className="dashboard">
        {/* ... */}
      </div>

      {/* Element selector */}
      <ElementSelector />
      <SelectorControls />
    </>
  );
}
```

---

## 7. Key Takeaways

### What Makes stagewise's Implementation Smooth?

1. **Velocity-based throttling** - Reduces work during fast mouse movement
2. **Fixed 30 FPS updates** - Smooth enough, efficient enough
3. **Ref-based positioning** - Bypasses React re-renders entirely
4. **Simple DOM overlays** - No Canvas/SVG complexity
5. **Smart element filtering** - Prevents false positives
6. **Duplicate checking** - Avoids unnecessary updates

### Simplicity Over Complexity

stagewise's implementation is refreshingly simple:
- No complex state machines
- No animation libraries
- No virtualization
- No memoization everywhere
- Just smart event handling + ref updates + RAF

### The Core Insight

**The secret to flicker-free selection isn't fancy rendering - it's smart event handling.**

By throttling at the input level (mouse events) rather than the output level (rendering), they achieve smooth performance without complex optimizations.

---

## 8. Comparison: stagewise vs Typical Approach

### Typical Approach (❌ Causes Flickering)

```typescript
// Update state on every mouse move
const [hoverPosition, setHoverPosition] = useState({ x: 0, y: 0 });

<div onMouseMove={(e) => setHoverPosition({ x: e.clientX, y: e.clientY })}>
  <Overlay position={hoverPosition} />
</div>

// Result:
// - React re-renders on every mouse move
// - Overlay re-renders constantly
// - Visible flickering and lag
```

### stagewise Approach (✅ Smooth)

```typescript
// Update ref on throttled schedule
const boxRef = useRef<HTMLDivElement>(null);

const updatePosition = () => {
  boxRef.current.style.top = `${rect.top}px`;
};

useCyclicUpdate(updatePosition, 30);

// Result:
// - No React re-renders during mouse movement
// - Overlay updates at fixed 30 FPS
// - Smooth, flicker-free experience
```

---

## 9. Performance Metrics (Estimated)

Based on code analysis:

| Metric | Value | Explanation |
|--------|-------|-------------|
| Mouse event frequency | ~100-200/sec | Browser default |
| Element detection rate | ~28 FPS | `setTimeout(fn, 1000/28)` |
| Overlay position rate | 30 FPS | `useCyclicUpdate(fn, 30)` |
| React re-renders (hover) | 1-5/sec | Only when element changes |
| React re-renders (position) | 0/sec | Ref-based updates |
| CPU usage (idle hover) | <5% | Efficient throttling |
| CPU usage (fast movement) | ~10% | Velocity-based delay |

---

## 10. Recommended Implementation Order

1. ✅ **Start simple:** Basic overlay with `onMouseMove` + state
2. ✅ **Add ref updates:** Replace state-based position with ref updates
3. ✅ **Add cyclic update:** Implement 30 FPS update loop
4. ✅ **Add velocity throttling:** Optimize mouse event handling
5. ✅ **Add duplicate checking:** Prevent unnecessary updates
6. ✅ **Polish UI:** Transitions, tooltips, styling

Don't try to implement everything at once. Start with the basics and add optimizations incrementally.

---

## Conclusion

stagewise's element selector is a masterclass in **pragmatic performance optimization**:

- **Not over-engineered** - Uses simple patterns effectively
- **Performance where it matters** - Optimizes hot paths (mouse events, rendering)
- **Maintainable** - Clear separation of concerns, readable code
- **Extensible** - Plugin system for custom behavior

The key insight: **Optimize at the input (events) rather than output (rendering).**

For your dashboard selector, adopt the core patterns (velocity throttling, ref updates, 30 FPS loop) but skip the complexity you don't need (iframe handling, XPath, extensive metadata).

**Estimated implementation time:** 4-6 hours for basic version, 8-12 hours with polish and edge cases.
