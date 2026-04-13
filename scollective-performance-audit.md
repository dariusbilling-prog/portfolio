# S Collective — Mobile Performance Audit
**Date:** April 8, 2026  
**URL:** https://scollective.com/  
**Platform:** Webflow  
**Current Mobile Score:** ~45 | LCP ~19s | FCP ~6-7s | TBT ~500ms

---

## SECTION 1 — LCP Diagnosis

### What is the LCP element?

The LCP element is the **Mux poster/thumbnail image** in the hero section:

```
<img src="https://image.mux.com/huFAv01RQJ02yYRoqnauGp8OrrpJdqZ5ODCTZpghvqS024/thumbnail.webp"
     loading="eager"
     fetchpriority="high">
```

- Rendered at full viewport width (~100vw on mobile)
- Natural dimensions: **1920×1080** (full HD)
- **No `srcset`** — mobile devices download the full 1920px image
- **No `sizes`** attribute
- Format: WebP (not AVIF despite your AVIF migration elsewhere)

### Why it takes ~19 seconds

The LCP image itself isn't inherently slow — the problem is everything that happens *before* the browser can even request it. The critical rendering path is catastrophically blocked:

1. **13 synchronous, render-blocking scripts** must download and execute before first paint
2. **7 render-blocking CSS files** (all `media="all"`) must download and parse
3. **Typekit font loader** (`pnn0jmd.js`, sync) triggers a cascade of **10 additional CSS requests** for individual font faces
4. **webfont.js** (sync) adds another blocking font-load step
5. **Google Fonts** (2 more blocking stylesheets)

On a mobile 4G connection (~1.6Mbps effective), this blocking chain alone takes 8-12 seconds before the browser even begins layout and paint. The LCP image can only render *after* all of this completes.

**The LCP is not slow because of the image. The LCP is slow because the entire page is render-blocked for 10+ seconds.**

---

## SECTION 2 — Critical Issues (Ranked by Impact)

### Issue #1: Massive Render-Blocking Script Chain (Impact: ~8-10s on mobile)

**13 synchronous scripts** block rendering:

| Script | Notes |
|--------|-------|
| `webfont.js` (cdnjs) | Sync font loader — blocks render |
| `pnn0jmd.js` (Typekit) | Sync — triggers 10 font CSS fetches |
| `jquery-3.5.1.min.dc5e7f18c8.js` | jQuery #1 |
| `jquery.min.js` (cdnjs) | jQuery #2 — **loaded twice** |
| `webflow.schunk.fa046cbb9c58fdf2.js` | Webflow runtime chunk |
| `webflow.schunk.4f6e7548b6508c8a.js` | Webflow runtime chunk |
| `webflow.206b4860.d3e755e9ec170d7d.js` | Webflow main |
| `owl.carousel.min.js` | Carousel library |
| `slick.min.js` | Another carousel library — **2 carousel libs** |
| 3× custom inline scripts | Embedded in Webflow |

On mobile (simulated slow 4G), each synchronous script serializes: the browser cannot parallelize their execution. Estimated combined download + parse + execute: **4-6 seconds**.

### Issue #2: Render-Blocking CSS + Font Loading Chain (Impact: ~4-6s on mobile)

**7 blocking stylesheets:**

| Stylesheet | Purpose |
|-----------|---------|
| `s-collective-800f5d.webflow.shared.*.css` | Webflow main |
| Google Fonts CSS × 2 | Font declarations |
| `jquery.fancybox.css` | Lightbox — not needed above fold |
| `owl.carousel.min.css` | Carousel — not needed above fold |
| `slick-theme.min.css` | Carousel theme |
| `slick.css` | Carousel base |

**Font loading is a 3-layer waterfall:**

```
Layer 1: webfont.js (sync script, ~16KB)
  └── Layer 2: Typekit pnn0jmd.js (sync script)
       └── Layer 3: 10× individual Typekit font CSS files
            └── Font files download

Parallel: 2× Google Fonts stylesheets → font files
```

This creates a **minimum 3-hop waterfall** before fonts render. On mobile, this alone adds 3-5 seconds to text rendering.

Additionally: **~37KB of inline `<style>` blocks** in the document, which the parser must process before rendering.

### Issue #3: Hero Poster Image Has No Responsive Sizing (Impact: ~1-2s on mobile)

The Mux thumbnail serves **1920×1080 to ALL devices** — including phones that only need ~400px wide. On mobile:

- Downloading ~1920px WebP when ~400px would suffice = **~4x unnecessary bytes**
- No `srcset` means the browser cannot pick a smaller variant
- No `sizes` means even if srcset existed, the browser couldn't make an informed choice

Mux's image API supports width parameters (`?width=400`) but this isn't being used.

### Issue #4: Third-Party Script Bloat (Impact: ~500-800ms TBT)

Third-party domains observed (20+ unique hosts):

| Service | Requests | Notes |
|---------|----------|-------|
| Typekit (use.typekit.net) | 12 | Font loading |
| Cloudflare Challenges | 4 | Bot detection |
| HubSpot (3 subdomains) | 5 | Forms + tracking |
| Stripe (js.stripe.com) | 4 | Payment — homepage? |
| Mux (stream + image + manifest) | 6 | Video |
| cdnjs.cloudflare.com | 8 | Libraries |
| Google (ajax.googleapis.com) | 1 | jQuery CDN |
| Analytics trackers | 4+ | statsdata, sevendata, lottingem, singleview |
| Behold.so | 1 | Instagram feed widget |

**Stripe JS on the homepage** is particularly wasteful — payment processing scripts should only load on checkout/cart pages.

### Issue #5: Duplicate Libraries and Unnecessary Below-Fold Assets

- **jQuery loaded twice** (Webflow bundle + CDN copy)
- **Two carousel libraries** (Owl Carousel + Slick) — pick one
- **3 S3-hosted `<video>` elements** at y=5043+ (far below fold) with `preload="metadata"` — one has `autoplay=true`
- **98 total `<img>` elements** on a single page
- Mux `@mux/mux-background-video` loaded synchronously as ES module from jsdelivr

---

## SECTION 3 — Exact Fixes (Dev-Ready)

### Fix #1: Defer All Non-Critical Scripts (Expected: -6-8s LCP)

**In Webflow Custom Code → Head:**

Remove the sync webfont.js and replace with `font-display: swap` in CSS. For scripts you can control:

```html
<!-- REMOVE this: -->
<script src="https://cdnjs.cloudflare.com/ajax/libs/webfont/1.6.26/webfont.js"></script>

<!-- ADD preconnects in <head> before anything else: -->
<link rel="preconnect" href="https://use.typekit.net" crossorigin>
<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
<link rel="preconnect" href="https://image.mux.com" crossorigin>
<link rel="preconnect" href="https://stream.mux.com" crossorigin>
```

**For third-party scripts, add `defer` or load after interaction:**

```html
<!-- Defer HubSpot until after page load -->
<script>
  window.addEventListener('load', function() {
    setTimeout(function() {
      var s = document.createElement('script');
      s.src = 'https://js-na2.hs-scripts.com/YOUR_ID.js';
      document.body.appendChild(s);
    }, 3000);
  });
</script>

<!-- Defer Stripe to cart/checkout pages only -->
<!-- Remove from homepage entirely, or: -->
<script>
  if (window.location.pathname.includes('/cart') || window.location.pathname.includes('/checkout')) {
    var s = document.createElement('script');
    s.src = 'https://js.stripe.com/v3/';
    document.body.appendChild(s);
  }
</script>
```

**For Owl Carousel and Slick CSS — load them async:**

```html
<!-- Convert render-blocking carousel CSS to non-blocking -->
<link rel="preload" href="owl.carousel.min.css" as="style" onload="this.onload=null;this.rel='stylesheet'">
<link rel="preload" href="slick.css" as="style" onload="this.onload=null;this.rel='stylesheet'">
<link rel="preload" href="jquery.fancybox.css" as="style" onload="this.onload=null;this.rel='stylesheet'">
```

> **Webflow limitation:** You can't directly control `defer`/`async` on Webflow-injected scripts. You'll need to:
> 1. Move third-party scripts to "Before </body>" custom code section
> 2. Use the lazy-load-script pattern above for HubSpot, Stripe, analytics
> 3. For Webflow's own jQuery/runtime — these are harder to defer (Webflow controls them). Focus on everything else.

### Fix #2: Fix Font Loading (Expected: -3-5s FCP)

**Option A: Replace Typekit + webfont.js with direct `@font-face` + `font-display: swap`**

```html
<!-- In Webflow Custom Code → Head, REMOVE: -->
<script src="https://cdnjs.cloudflare.com/ajax/libs/webfont/1.6.26/webfont.js"></script>
<script>WebFont.load({typekit: {id: 'pnn0jmd'}});</script>

<!-- REPLACE WITH preloading your primary font only: -->
<link rel="preload" href="https://use.typekit.net/af/28b120/000000000000000077359e3c/31/l?primer=..." as="font" type="font/woff2" crossorigin>

<!-- And in CSS, ensure all @font-face rules have: -->
<style>
  @font-face {
    font-family: 'YourTypekitFont';
    font-display: swap; /* Critical — prevents FOIT */
    /* ... */
  }
</style>
```

**Option B (simpler): Keep Typekit but make it async:**

```html
<link rel="preload" href="https://use.typekit.net/pnn0jmd.css" as="style">
<link rel="stylesheet" href="https://use.typekit.net/pnn0jmd.css" media="print" onload="this.media='all'">
```

This changes the 3-hop waterfall (sync JS → CSS → fonts) into a single async CSS load.

### Fix #3: Add Responsive Sizing to LCP Image (Expected: -1-2s LCP)

Mux's image API supports `width` and `height` params. Inject this via Webflow custom code or embed:

```html
<!-- Replace the Mux poster with responsive srcset -->
<style>
  .sc-hero-bg-vid img,
  mux-background-video img {
    /* Ensure the poster image gets responsive treatment */
  }
</style>

<script>
  // After DOM ready, fix the Mux poster image
  document.querySelectorAll('mux-background-video img, .sc-hero-section img[src*="mux"]').forEach(img => {
    const baseUrl = img.src.replace('/thumbnail.webp', '');
    img.srcset = `
      ${baseUrl}/thumbnail.webp?width=480 480w,
      ${baseUrl}/thumbnail.webp?width=768 768w,
      ${baseUrl}/thumbnail.webp?width=1280 1280w,
      ${baseUrl}/thumbnail.webp?width=1920 1920w
    `;
    img.sizes = '100vw';
  });
</script>
```

**Better approach — preload the mobile-sized poster in `<head>`:**

```html
<link rel="preload" as="image" 
  href="https://image.mux.com/huFAv01RQJ02yYRoqnauGp8OrrpJdqZ5ODCTZpghvqS024/thumbnail.webp?width=480" 
  media="(max-width: 768px)">
<link rel="preload" as="image"
  href="https://image.mux.com/huFAv01RQJ02yYRoqnauGp8OrrpJdqZ5ODCTZpghvqS024/thumbnail.webp?width=1280"
  media="(min-width: 769px)">
```

### Fix #4: Remove Duplicate Libraries (Expected: -200-400ms TBT)

- **Remove one jQuery** — the Webflow-bundled one should suffice. Remove the cdnjs copy.
- **Remove either Owl Carousel or Slick** — using both is pointless. Consolidate to one.
- **Remove `jquery.fancybox.css`** if FancyBox isn't used above the fold — load it on interaction.

### Fix #5: Lazy-Load Below-Fold Videos (Expected: -500ms+ load time)

The 3 S3-hosted `<video>` elements are at y=5043+ (deeply below fold). One has `autoplay=true` which triggers download:

```html
<!-- Change hero-slide-video (the S3 one, NOT the Mux hero) -->
<video class="hero-slide-video" preload="none" autoplay="false" muted>
  <!-- Remove src/source until visible -->
</video>
```

**Better: use Intersection Observer to load videos only when visible:**

```javascript
const observer = new IntersectionObserver((entries) => {
  entries.forEach(entry => {
    if (entry.isIntersecting) {
      const video = entry.target;
      const source = video.querySelector('source');
      if (source && source.dataset.src) {
        source.src = source.dataset.src;
        video.load();
        if (video.autoplay) video.play();
      }
      observer.unobserve(video);
    }
  });
}, { rootMargin: '200px' });

document.querySelectorAll('video:not(.sc-hero-bg-vid)').forEach(v => observer.observe(v));
```

### Fix #6: Remove Stripe from Homepage

```javascript
// Only load Stripe on pages that need it
if (document.querySelector('[data-stripe]') || /\/(cart|checkout|shop)/.test(location.pathname)) {
  const s = document.createElement('script');
  s.src = 'https://js.stripe.com/v3/';
  s.async = true;
  document.head.appendChild(s);
}
```

---

## SECTION 4 — Expected Impact

| Fix | Current Impact | Expected Improvement | New Estimated |
|-----|---------------|---------------------|---------------|
| Defer render-blocking scripts | ~8-10s blocked | -6-8s LCP | Major |
| Fix font loading chain | ~4-6s FOIT/blocked | -3-5s FCP | Major |
| Responsive LCP image | ~1-2s wasted download | -1-2s LCP | Moderate |
| Remove duplicate libs | ~200-400ms TBT | -200ms TBT | Minor |
| Lazy-load below-fold videos | ~500ms+ load | -500ms load | Minor |
| Remove Stripe from homepage | ~200ms TBT | -200ms TBT | Minor |

**Combined realistic estimate:**
- LCP: 19s → **3-5s** (with all fixes), potentially **<2.5s** if font loading is fully optimized
- FCP: 6-7s → **1.5-2.5s**
- TBT: 500ms → **200-300ms**
- Score: 45 → **70-85**

To break under 2.5s LCP, you need fixes #1 + #2 + #3 as a minimum.

---

## SECTION 5 — Quick Wins vs. Structural Fixes

### Quick Wins (can do today in Webflow)

1. **Add `<link rel="preload">` for the mobile hero poster** in Webflow Head code — 5 minutes, saves ~1-2s
2. **Move HubSpot, Stripe, analytics to Before `</body>`** and wrap in `setTimeout` — 15 minutes, saves ~1-2s TBT
3. **Remove webfont.js** and use Typekit CSS link with `font-display: swap` instead — 10 minutes, saves ~1-2s FCP
4. **Add `preload="none"` to below-fold S3 videos** — 5 minutes
5. **Preconnect to critical origins** (Typekit, Mux, Google Fonts) in `<head>` — 5 minutes, saves ~200-500ms per origin

### Structural Fixes (requires more effort)

1. **Eliminate one carousel library** (Owl or Slick) — requires refactoring carousel markup
2. **Remove duplicate jQuery** — need to verify nothing depends on the CDN copy
3. **Convert all render-blocking CSS to async** (fancybox, carousel styles) — Webflow may fight you on this
4. **Implement responsive Mux poster via srcset** — requires custom embed code
5. **Conditionally load Stripe** — requires modifying e-commerce integration
6. **Long-term: migrate off Webflow** — Webflow's mandatory sync script loading (jQuery + runtime chunks) is an inherent bottleneck you cannot fully eliminate. A static site or Next.js build would remove ~4-5 forced render-blocking scripts.

---

## Architecture Note

The fundamental constraint is **Webflow**. It force-loads jQuery + its runtime chunks synchronously, and you can't change that. This means even with all optimizations, you'll always have ~3-4 render-blocking scripts you can't remove. The ceiling on Webflow for mobile performance is roughly a score of 80-85 — getting above 90 would likely require migrating to a different platform.

Focus your effort on the things you *can* control: font loading (biggest bang for buck), third-party scripts, and the LCP image sizing.
