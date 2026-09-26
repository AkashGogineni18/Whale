---
name: framer-motion
description: "Framer Motion animation skill for React. Covers motion components, variants, gestures, layout animations, scroll effects, page transitions, AnimatePresence, useAnimation, useScroll, useTransform, spring physics, stagger, orchestration, drag, and performance best practices. Use when adding animations to React apps with Framer Motion (now called Motion for React / motion library)."
---

# Framer Motion — Animation Intelligence for React

Complete reference for building production-grade animations in React using Framer Motion (`motion` / `framer-motion`). Covers every major API, pattern, and pitfall.

## Package Note

- **v10 and below**: `import { motion } from 'framer-motion'`
- **v11+ (Motion for React)**: `import { motion } from 'motion/react'`
- Both APIs are identical; only the import path changed.
- Default to v11 (`motion/react`) for new projects.

---

## Core Concepts

### 1. motion components

Every HTML/SVG element has a `motion.*` counterpart. Wrap it to opt into animation.

```tsx
import { motion } from 'motion/react';

// Basic fade + slide in
<motion.div
  initial={{ opacity: 0, y: 20 }}
  animate={{ opacity: 1, y: 0 }}
  transition={{ duration: 0.35, ease: [0.25, 0, 0.2, 1] }}
/>

// Custom component — use motion.create()
const MotionCard = motion.create(Card);
```

**Rules:**
- Animate `opacity`, `transform` (x, y, scale, rotate) — GPU-composited, no layout thrash
- Never animate `width`, `height`, `top`, `left` directly — use `scaleX`/`scaleY` or layout animations
- `x`, `y` map to `translateX`/`translateY` (no unit needed for px)

---

### 2. Variants

Named animation states for clean, reusable, orchestrated animations.

```tsx
const container = {
  hidden: { opacity: 0 },
  visible: {
    opacity: 1,
    transition: {
      staggerChildren: 0.07,   // each child staggers by 70ms
      delayChildren: 0.1,
    },
  },
};

const item = {
  hidden: { opacity: 0, y: 16 },
  visible: {
    opacity: 1,
    y: 0,
    transition: { duration: 0.3, ease: [0.25, 0, 0.2, 1] },
  },
};

<motion.ul variants={container} initial="hidden" animate="visible">
  {items.map((i) => (
    <motion.li key={i.id} variants={item}>{i.label}</motion.li>
  ))}
</motion.ul>
```

**Key rules:**
- Parent `animate` propagates variant name to all descendants automatically — children only need `variants`, not `animate`
- Override at any level: child `animate="custom"` overrides parent propagation
- `staggerChildren` applies between children; `staggerDirection: -1` reverses the stagger

---

### 3. Transition

Controls timing, easing, and physics.

```tsx
// Duration + easing (CSS-style)
transition={{ duration: 0.3, ease: 'easeOut' }}
transition={{ duration: 0.4, ease: [0.25, 0, 0.2, 1] }}  // custom cubic-bezier

// Spring (preferred for interactive / gesture-response)
transition={{ type: 'spring', stiffness: 300, damping: 30 }}
transition={{ type: 'spring', mass: 0.5, stiffness: 400, damping: 25 }}

// Tween
transition={{ type: 'tween', duration: 0.35 }}

// Delay
transition={{ delay: 0.2, duration: 0.3 }}

// Per-property transitions
transition={{
  opacity: { duration: 0.2 },
  y: { type: 'spring', stiffness: 300, damping: 28 },
}}
```

**Easing reference:**
| Name | Cubic Bezier | Use case |
|------|-------------|----------|
| `easeOut` | `[0, 0, 0.2, 1]` | Elements entering the screen |
| `easeIn` | `[0.4, 0, 1, 1]` | Elements leaving the screen |
| `easeInOut` | `[0.4, 0, 0.2, 1]` | Cross-fades, toggles |
| Apple spring | `type: 'spring', stiffness: 300, damping: 30` | Interactive responses |

---

### 4. AnimatePresence

Required for exit animations. Keeps unmounting components in the DOM until their `exit` animation finishes.

```tsx
import { AnimatePresence, motion } from 'motion/react';

// Modal
<AnimatePresence>
  {isOpen && (
    <motion.div
      key="modal"
      initial={{ opacity: 0, scale: 0.95 }}
      animate={{ opacity: 1, scale: 1 }}
      exit={{ opacity: 0, scale: 0.95 }}
      transition={{ duration: 0.2 }}
    />
  )}
</AnimatePresence>

// List item removal
<AnimatePresence>
  {items.map((item) => (
    <motion.div
      key={item.id}           // key is required — drives mount/unmount
      initial={{ opacity: 0, height: 0 }}
      animate={{ opacity: 1, height: 'auto' }}
      exit={{ opacity: 0, height: 0 }}
    />
  ))}
</AnimatePresence>
```

**Rules:**
- Always provide a stable `key` prop on the child
- `mode="wait"` — exit finishes before enter starts (good for page transitions)
- `mode="popLayout"` — exiting element leaves layout flow immediately, others animate into place
- `initial={false}` on `AnimatePresence` — skip entrance animation on first render

---

### 5. Layout Animations

Animate an element when its size or position changes in the DOM — no manual calculation.

```tsx
// Single element
<motion.div layout />

// Layout + content fade
<motion.div layout>
  <motion.p layout="position">{text}</motion.p>
</motion.div>

// Shared layout (hero transition between two screens)
<motion.div layoutId="card-123" />  // same layoutId on both screens
```

**`layout` prop values:**
- `layout={true}` — animate position + size
- `layout="position"` — position only
- `layout="size"` — size only
- `layout="preserve-aspect"` — maintain aspect ratio during resize
- `layoutId="key"` — connect two elements across the tree for shared transitions

**`LayoutGroup`** — required when multiple independent layout-animated elements are on screen:
```tsx
import { LayoutGroup } from 'motion/react';
<LayoutGroup><TabBar /><Content /></LayoutGroup>
```

---

### 6. Gestures

```tsx
// Hover
<motion.button
  whileHover={{ scale: 1.04, backgroundColor: '#333' }}
  whileTap={{ scale: 0.96 }}
  transition={{ type: 'spring', stiffness: 400, damping: 25 }}
/>

// Drag
<motion.div
  drag                          // free drag
  drag="x"                      // constrain to axis
  dragConstraints={{ left: 0, right: 300 }}
  dragElastic={0.1}             // 0 = rigid, 1 = fully elastic
  dragSnapToOrigin              // snap back on release
  onDragEnd={(event, info) => console.log(info.point)}
/>

// Focus (accessibility)
<motion.button whileFocus={{ scale: 1.02 }} />
```

---

### 7. useAnimation (imperative control)

Use when timing is driven by external events, not mount/unmount.

```tsx
import { useAnimation, motion } from 'motion/react';

const controls = useAnimation();

// Trigger from anywhere
await controls.start({ opacity: 1, y: 0 });
controls.stop();
controls.set({ opacity: 0 });   // instant, no animation

<motion.div animate={controls} initial={{ opacity: 0, y: 20 }} />
```

---

### 8. useScroll + useTransform (scroll-driven animations)

```tsx
import { useScroll, useTransform, motion } from 'motion/react';
import { useRef } from 'react';

// Parallax
function ParallaxSection() {
  const ref = useRef(null);
  const { scrollYProgress } = useScroll({
    target: ref,
    offset: ['start end', 'end start'],  // [when target enters, when target leaves]
  });

  const y = useTransform(scrollYProgress, [0, 1], ['0%', '-30%']);
  const opacity = useTransform(scrollYProgress, [0, 0.3, 0.7, 1], [0, 1, 1, 0]);

  return (
    <section ref={ref}>
      <motion.div style={{ y, opacity }} />
    </section>
  );
}

// Sticky progress bar
function ProgressBar() {
  const { scrollYProgress } = useScroll();
  return <motion.div style={{ scaleX: scrollYProgress, transformOrigin: 'left' }} />;
}
```

**`useTransform` clamp + custom mapping:**
```tsx
const scale = useTransform(scrollYProgress, [0, 0.5, 1], [0.8, 1, 1.2]);
// Input values must be monotonically increasing
// Output values can go any direction
```

---

### 9. useSpring

Smooths any `MotionValue` with spring physics — great for cursor followers, smooth counters.

```tsx
import { useSpring, useMotionValue, motion } from 'motion/react';

const x = useMotionValue(0);
const springX = useSpring(x, { stiffness: 200, damping: 20 });

<motion.div style={{ x: springX }} onMouseMove={(e) => x.set(e.clientX)} />
```

---

### 10. Page Transitions (React Router / Next.js)

```tsx
// React Router v6
const pageVariants = {
  initial:  { opacity: 0, x: 24 },
  enter:    { opacity: 1, x: 0, transition: { duration: 0.25, ease: [0.25, 0, 0.2, 1] } },
  exit:     { opacity: 0, x: -24, transition: { duration: 0.18, ease: [0.4, 0, 1, 1] } },
};

function PageWrapper({ children }) {
  return (
    <AnimatePresence mode="wait">
      <motion.div
        key={location.pathname}
        variants={pageVariants}
        initial="initial"
        animate="enter"
        exit="exit"
      >
        {children}
      </motion.div>
    </AnimatePresence>
  );
}

// Next.js App Router — in layout.tsx
// Wrap <main> with motion.main; use usePathname() as key
```

---

### 11. Common Patterns

#### Staggered list entrance
```tsx
const list = { hidden: { opacity: 0 }, show: { opacity: 1, transition: { staggerChildren: 0.06 } } };
const item = { hidden: { opacity: 0, y: 12 }, show: { opacity: 1, y: 0 } };

<motion.ul variants={list} initial="hidden" animate="show">
  {data.map((d) => <motion.li key={d.id} variants={item}>{d.label}</motion.li>)}
</motion.ul>
```

#### Accordion / height animation
```tsx
<motion.div
  initial={false}
  animate={{ height: isOpen ? 'auto' : 0, opacity: isOpen ? 1 : 0 }}
  transition={{ duration: 0.25, ease: [0.25, 0, 0.2, 1] }}
  style={{ overflow: 'hidden' }}
/>
// Note: 'auto' in height requires layout animations for best results
// Alternative: use layout prop and conditional render inside AnimatePresence
```

#### Number counter
```tsx
import { useSpring, useTransform, motion } from 'motion/react';

function Counter({ value }: { value: number }) {
  const spring = useSpring(value, { stiffness: 100, damping: 20 });
  const display = useTransform(spring, (v) => Math.round(v).toLocaleString());
  return <motion.span>{display}</motion.span>;
}
```

#### Skeleton shimmer
```tsx
const shimmer = {
  animate: {
    backgroundPosition: ['200% 0', '-200% 0'],
    transition: { duration: 1.5, ease: 'linear', repeat: Infinity },
  },
};

<motion.div
  {...shimmer}
  style={{
    background: 'linear-gradient(90deg, #1a1a1a 25%, #2a2a2a 50%, #1a1a1a 75%)',
    backgroundSize: '400% 100%',
  }}
/>
```

#### Drag-to-dismiss card
```tsx
const THRESHOLD = 100;

<motion.div
  drag="x"
  dragConstraints={{ left: 0, right: 0 }}
  onDragEnd={(_, { offset, velocity }) => {
    if (Math.abs(offset.x) > THRESHOLD || Math.abs(velocity.x) > 500) {
      onDismiss();
    }
  }}
  animate={{ x: 0 }}
  exit={{ x: 300, opacity: 0 }}
/>
```

---

### 12. Performance Rules

- Animate **only** `opacity` and `transform` properties (x, y, scale, rotate, skew) — these run on the GPU compositor thread and never trigger layout
- **Never** animate: `width`, `height`, `top`, `left`, `margin`, `padding` — use `scaleX`/`scaleY` + `layout` prop instead
- Use `will-change: transform` sparingly on elements that animate continuously (scroll-driven, infinite)
- `layoutRoot` on a scrolling container prevents layout animations from propagating outside it (performance)
- Use `useReducedMotion()` hook to respect OS accessibility settings:
```tsx
import { useReducedMotion } from 'motion/react';

function AnimatedComponent() {
  const prefersReduced = useReducedMotion();
  return (
    <motion.div
      animate={{ opacity: 1, y: prefersReduced ? 0 : -20 }}
    />
  );
}
```
- Avoid `animate` objects recreated every render — define outside component or with `useMemo`
- Don't use `motion.div` for every element — only the ones that actually animate

---

### 13. TypeScript Types

```tsx
import type { Variants, Transition, MotionProps } from 'motion/react';

const variants: Variants = {
  hidden: { opacity: 0 },
  visible: { opacity: 1 },
};

const transition: Transition = {
  type: 'spring',
  stiffness: 300,
  damping: 28,
};

// Extend component props with motion
interface CardProps extends MotionProps {
  title: string;
}
```

---

### 14. Installation

```bash
# New projects (Motion for React, v11+)
npm install motion

# Legacy / existing framer-motion projects
npm install framer-motion

# Import (v11+)
import { motion, AnimatePresence, useScroll } from 'motion/react';

# Import (v10 and below)
import { motion, AnimatePresence, useScroll } from 'framer-motion';
```

---

### 15. Anti-Patterns to Avoid

| Don't | Do instead |
|-------|-----------|
| Animate `height: 0` to `height: auto` directly | Use `layout` prop + `AnimatePresence` |
| Put `variants` object inline in JSX | Define outside component or use `useMemo` |
| Use `motion.div` everywhere | Only on elements that actually animate |
| Animate `backgroundColor` on hover | Use CSS `transition` for color, Framer Motion for transforms |
| Forget `key` on `AnimatePresence` children | Always provide stable `key` |
| Nest `AnimatePresence` unnecessarily | One `AnimatePresence` wraps all conditional children |
| Use `initial` on elements deep in a tree | Let variants propagate from the nearest `motion` ancestor |
| Ignore `useReducedMotion` | Always respect `prefers-reduced-motion` |
