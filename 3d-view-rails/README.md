# 3D Card Animations with Stimulus in Rails

Interactive card animations without heavy JavaScript frameworks - perfect for Rails apps.

## The Vercel Badge Inspiration

Vercel's conference badge uses physics-based animations with these libraries:

- **react-spring** or **framer-motion** - Physics-based animations (springs, inertia, drag)
- **react-use-gesture** - Cursor/touch input tracking
- **three.js** (optional) - True 3D effects and lighting

But for Rails apps, we can achieve great results with much simpler approaches!

## Library Comparison for Rails

| Approach             | Best For                           | Complexity                 | Bundle Size    |
| -------------------- | ---------------------------------- | -------------------------- | -------------- |
| Vanilla JS + CSS     | Simple drag + spring feel          | ✅ Low                     | ✅ Minimal     |
| Stimulus             | Rails integration, maintainability | ✅ Low                     | ✅ Minimal     |
| three.js             | True 3D, lighting, depth           | ❌ Overkill for most cases | ❌ Large       |
| React + react-spring | Best physics fidelity              | ⚠️ High complexity         | ⚠️ Medium size |

## Simple 3D Card Flip (No JavaScript Required)

### Basic HTML Structure

```erb
<div class="card-container">
  <div class="card">
    <div class="card-face card-front">
      <img src="<%= asset_path('card-front.png') %>" alt="Front of card" />
    </div>
    <div class="card-face card-back">
      <img src="<%= asset_path('card-back.png') %>" alt="Back of card" />
    </div>
  </div>
</div>
```

### CSS for 3D Effect

```css
.card-container {
  perspective: 1000px;
  width: 300px;
  height: 400px;
}

.card {
  width: 100%;
  height: 100%;
  position: relative;
  transform-style: preserve-3d;
  transition: transform 0.6s;
  cursor: pointer;
}

.card-container:hover .card {
  transform: rotateY(180deg);
}

.card-face {
  position: absolute;
  width: 100%;
  height: 100%;
  backface-visibility: hidden;
  border-radius: 16px;
  overflow: hidden;
}

.card-front {
  z-index: 2;
}

.card-back {
  transform: rotateY(180deg);
}
```

### Tailwind Version

```erb
<div class="relative w-[300px] h-[400px] [perspective:1000px] group">
  <div class="w-full h-full transition-transform duration-500 [transform-style:preserve-3d] group-hover:rotate-y-180">
    <div class="absolute inset-0 [backface-visibility:hidden] rounded-xl overflow-hidden z-10">
      <img src="<%= asset_path('card-front.png') %>" class="w-full h-full object-cover" />
    </div>
    <div class="absolute inset-0 [backface-visibility:hidden] rotate-y-180 rounded-xl overflow-hidden">
      <img src="<%= asset_path('card-back.png') %>" class="w-full h-full object-cover" />
    </div>
  </div>
</div>
```

## Enhanced with Stimulus Controller

### 1. Create the Controller

**File:** `app/javascript/controllers/card_flip_controller.js`

```javascript
import { Controller } from "@hotwired/stimulus";

export default class extends Controller {
  static targets = ["card"];

  flip() {
    this.cardTarget.classList.toggle("rotate-y-180");
  }
}
```

### 2. HTML with Stimulus

```erb
<div
  data-controller="card-flip"
  data-action="click->card-flip#flip"
  class="relative w-[300px] h-[400px] [perspective:1000px]"
>
  <div
    data-card-flip-target="card"
    class="w-full h-full transition-transform duration-500 [transform-style:preserve-3d]"
  >
    <div class="absolute inset-0 [backface-visibility:hidden] rounded-xl overflow-hidden z-10">
      <img src="<%= asset_path('card-front.png') %>" class="w-full h-full object-cover" />
    </div>
    <div class="absolute inset-0 [backface-visibility:hidden] rotate-y-180 rounded-xl overflow-hidden">
      <img src="<%= asset_path('card-back.png') %>" class="w-full h-full object-cover" />
    </div>
  </div>
</div>
```

### 3. Tailwind Custom Classes

If using `rotate-y-180`, add to `tailwind.config.js`:

```javascript
module.exports = {
  theme: {
    extend: {
      transform: {
        "rotate-y-180": "rotateY(180deg)",
      },
    },
  },
};
```

Or use arbitrary values:

```html
class="[transform:rotateY(180deg)]"
```

## Advanced Drag & Physics (Vanilla JS)

For more interactive cards with drag physics:

```html
<div
  id="draggable-badge"
  style="position: absolute; width: 300px; height: 400px; background: url('/badge.png'); background-size: cover; border-radius: 16px; cursor: grab;"
></div>

<script>
  const el = document.getElementById("draggable-badge");
  let isDragging = false;
  let pos = { x: 0, y: 0 };
  let velocity = { x: 0, y: 0 };
  let last = { x: 0, y: 0 };
  let animationFrame;

  el.addEventListener("pointerdown", (e) => {
    isDragging = true;
    last = { x: e.clientX, y: e.clientY };
    cancelAnimationFrame(animationFrame);
  });

  window.addEventListener("pointermove", (e) => {
    if (!isDragging) return;
    const dx = e.clientX - last.x;
    const dy = e.clientY - last.y;
    velocity = { x: dx, y: dy };
    pos.x += dx;
    pos.y += dy;
    last = { x: e.clientX, y: e.clientY };
    el.style.transform = `translate(${pos.x}px, ${pos.y}px)`;
  });

  window.addEventListener("pointerup", () => {
    isDragging = false;
    animatePhysics();
  });

  function animatePhysics() {
    velocity.x *= 0.95; // Friction
    velocity.y *= 0.95;
    pos.x += velocity.x;
    pos.y += velocity.y;
    el.style.transform = `translate(${pos.x}px, ${pos.y}px)`;

    if (Math.abs(velocity.x) > 0.5 || Math.abs(velocity.y) > 0.5) {
      animationFrame = requestAnimationFrame(animatePhysics);
    }
  }
</script>
```

## Rails 8 Stimulus Auto-Registration

**Good news:** In Rails 8, Stimulus controllers are auto-registered!

### What Happens Automatically:

- Any `.js` file in `app/javascript/controllers/` is auto-discovered
- File name becomes controller name (`card_flip_controller.js` → `card-flip`)
- No manual registration needed

### Required Setup:

1. **Layout includes JavaScript:**

   ```erb
   <%= javascript_importmap_tags %>
   ```

2. **Application.js imports controllers:**

   ```javascript
   import "controllers";
   ```

3. **File structure:**
   ```
   app/
   ├─ javascript/
   │  ├─ application.js         # imports "controllers"
   │  ├─ controllers/
   │  │  ├─ index.js            # auto-loader
   │  │  └─ card_flip_controller.js
   ```

### Auto-Registration Checklist:

| Task                                   | Required in Rails 8 |
| -------------------------------------- | ------------------- |
| Manually register controllers          | ❌ No               |
| Create controller files                | ✅ Yes              |
| Use `data-controller` in HTML          | ✅ Yes              |
| Import "controllers" in application.js | ✅ Yes              |

## Advanced Stimulus Card Controller

For more complex interactions:

```javascript
import { Controller } from "@hotwired/stimulus";

export default class extends Controller {
  static targets = ["card", "front", "back"];
  static values = {
    flipped: Boolean,
    autoFlip: Boolean,
    flipDuration: Number,
  };

  connect() {
    this.flipDurationValue = this.flipDurationValue || 500;
    if (this.autoFlipValue) {
      this.startAutoFlip();
    }
  }

  disconnect() {
    this.stopAutoFlip();
  }

  flip() {
    this.flippedValue = !this.flippedValue;
    this.updateCardState();
  }

  flipToFront() {
    this.flippedValue = false;
    this.updateCardState();
  }

  flipToBack() {
    this.flippedValue = true;
    this.updateCardState();
  }

  updateCardState() {
    this.cardTarget.style.transitionDuration = `${this.flipDurationValue}ms`;

    if (this.flippedValue) {
      this.cardTarget.classList.add("rotate-y-180");
    } else {
      this.cardTarget.classList.remove("rotate-y-180");
    }
  }

  startAutoFlip() {
    this.autoFlipInterval = setInterval(() => {
      this.flip();
    }, 3000);
  }

  stopAutoFlip() {
    if (this.autoFlipInterval) {
      clearInterval(this.autoFlipInterval);
    }
  }

  // Handle hover effects
  mouseEnter() {
    this.cardTarget.style.transform += " scale(1.05)";
  }

  mouseLeave() {
    this.cardTarget.style.transform = this.cardTarget.style.transform.replace(
      " scale(1.05)",
      ""
    );
  }
}
```

**Usage:**

```erb
<div
  data-controller="card-flip"
  data-card-flip-auto-flip-value="true"
  data-card-flip-flip-duration-value="600"
  data-action="click->card-flip#flip mouseenter->card-flip#mouseEnter mouseleave->card-flip#mouseLeave"
  class="relative w-[300px] h-[400px] [perspective:1000px]"
>
  <!-- card content -->
</div>
```

## CSS 3D Fundamentals

**Key properties for 3D effects:**

```css
/* Container needs perspective */
.container {
  perspective: 1000px; /* Distance from viewer */
}

/* Element needs 3D context */
.card {
  transform-style: preserve-3d; /* Enable 3D for children */
  transition: transform 0.5s; /* Smooth animations */
}

/* Hide back faces */
.face {
  backface-visibility: hidden; /* Hide when rotated away */
}

/* 3D transforms */
.front {
  transform: rotateY(0deg);
}
.back {
  transform: rotateY(180deg);
}
```

## Performance Tips

1. **Use `transform` instead of changing `top/left`** - GPU accelerated
2. **Add `will-change: transform`** for smoother animations
3. **Use `transform3d()` to trigger hardware acceleration**
4. **Avoid animating `box-shadow`** - use pseudo-elements instead
5. **Debounce drag events** for better performance

## Common Issues & Solutions

**Card appears flat:**

- Check `perspective` on container
- Ensure `transform-style: preserve-3d` on card

**Back shows through front:**

- Add `backface-visibility: hidden` to both faces
- Check z-index values

**Animation feels choppy:**

- Add `will-change: transform`
- Use `transform3d(0,0,0)` to trigger GPU acceleration

**Mobile touch issues:**

- Use `touch-action: none` to prevent scrolling
- Handle `touchstart/touchmove/touchend` events
