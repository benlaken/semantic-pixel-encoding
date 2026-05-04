# Semantic Pixel Encoding
A novel architecture for encoding semantic data in AI-generated images, decoded in real-time via WebGL fragment shader for touch interaction. Prior art disclosure 

---
## Prior Art Statement

This document constitutes a public disclosure of the semantic pixel encoding architecture described below, establishing priority of invention by Dr. Benjamin A. Laken on May 4th, 2026.

---
## The Core Idea

AI-generated images are typically treated as presentation assets — static visuals sitting beneath a separate interaction layer. This architecture collapses that separation. Every meaningful pixel in the image carries both a visual meaning and a semantic one simultaneously. The user sees the visual. The machine reads the semantic. They never conflict.

The image IS the data structure.

<img src="test_circle.png" width="300" alt="Validation test image — single high-luminance element on pure black">
*A single high-luminance teal element on pure black — the simplest case for cluster detection and semantic encoding validation.*

---

## The Encoding Scheme

In dark, atmospheric visual aesthetics — the kind produced by modern AI image generators — certain RGB combinations never appear naturally. High R values (≥250) combined with the underwater/bioluminescent palette are effectively impossible without deliberate encoding. That range is reserved.

R = 252   →  encoded pixel marker
G = 0–255 →  element type identifier
B = 0–255 →  numeric value (maps linearly to domain range)

Example: a data point with score 41 on a 0–100 scale:
  RGB(252, 2, 105)
  R=252 → encoded marker
  G=2   → element type: data node, second in series
  B=105 → value: 105/255 × 100 = 41.2

---

## The Pipeline

### 1. Image Generation
Generate the atmospheric image normally via AI image generation (Grok, DALL-E, Midjourney, etc.). No encoding at this stage.

### 2. Post-Processing (encoding step)
A script runs cluster detection on the generated image — identifying high-luminance elements by brightness threshold. For each detected cluster, it writes the encoding color to those pixel regions.

  ```python
  def encode_element(image, centroid, radius, element_type, value, domain_max=100):
      x0, y0 = centroid
      for y in range(image.height):
          for x in range(image.width):
              if distance((x,y), (x0,y0)) <= radius:
                  image.putpixel((x, y), (
                      252,                          # marker
                      element_type,                 # type
                      int((value / domain_max) * 255)  # value
                  ))
      return image
  ```

### 3. WebGL Fragment Shader (decode + render)

Each tile is a <canvas> element. The image loads as a WebGL texture. The fragment shader intercepts encoded pixels before display:

  ```glsl
  uniform sampler2D u_image;
  uniform float u_time;
  varying vec2 v_uv;

  vec3 valueRamp(float t) {
      vec3 low  = vec3(0.78, 0.18, 0.18); // deep red
      vec3 mid  = vec3(0.82, 0.55, 0.12); // amber
      vec3 high = vec3(0.31, 0.82, 0.78); // teal
      if (t < 0.5) return mix(low, mid, t * 2.0);
      return mix(mid, high, (t - 0.5) * 2.0);
  }

  void main() {
      vec4 px = texture2D(u_image, v_uv);

      if (px.r > 0.98) {
          // encoded pixel — decode and render
          float value = px.b; // 0.0–1.0
          vec3 displayColor = valueRamp(value);

          // animate intensity based on value
          float breathe = sin(u_time * (0.8 + value * 0.8)) * 0.5 + 0.5;
          gl_FragColor = vec4(displayColor * (0.7 + breathe * 0.3), 1.0);
      } else {
          // normal rendering pipeline
          float lum = dot(px.rgb, vec3(0.299, 0.587, 0.114));
          float teal = clamp((px.g + px.b) - px.r * 2.5, 0.0, 1.0);
          float breathe = sin(u_time * 1.4) * 0.5 + 0.5;
          float glow = teal * lum * breathe * 0.25;

          // edge vignette to page background
          vec2 d = abs(v_uv - 0.5) * 2.0;
          float vignette = 1.0 - smoothstep(0.55, 1.0, max(d.x, d.y));
          vec3 bg = vec3(0.031, 0.035, 0.059); // #08090f

          gl_FragColor = vec4(mix(bg, px.rgb + vec3(-glow*0.1, glow, glow*0.9),
                                 vignette), 1.0);
      }
  }
  ```

### 4. Semantic Interaction

On touch/click, JavaScript reads the original texture — not the rendered output — at the interaction coordinates:

  ```javascript
  function handleInteraction(canvas, glContext, originalTexture, event) {
      const rect = canvas.getBoundingClientRect();
      const x = Math.floor((event.clientX - rect.left)
                           * (originalTexture.width / rect.width));
      const y = Math.floor((event.clientY - rect.top)
                           * (originalTexture.height / rect.height));

      const pixel = readTexturePixel(glContext, originalTexture, x, y);

      if (pixel.r >= 252) {
          const elementType = pixel.g;
          const value = (pixel.b / 255) * domainMax[elementType];

          dispatch({
              type: 'SEMANTIC_TAP',
              elementType,
              value,
              position: { x, y }
          });
      }
  }
  ```

No coordinate map. No hitbox JSON. No calibration file. Sub-pixel precision across the entire image surface.

---
## The Broader Architecture

This mechanism operates inside a larger closed-loop system:

  ```
  Personal Knowledge Graph
             ↓
     AI Agent Reasoning
     (knowledge graph + user model + session log)
             ↓
     Generative Imagery
     (encodes knowledge graph state as pixel semantics)
             ↓
     WebGL Decoder
     (renders correct display + enables semantic interaction)
             ↓
     Semantic Interaction
     (touch → meaning → knowledge graph update)
             ↓
           [loop]
  ```
The AI agent does not personalise content within a fixed layout. It reasons about what to compose — which visualisations exist, which questions surface, which data is foregrounded. The UI composition itself is generated per user per session from a psychometric or knowledge model. Every interaction closes the loop.

---
## Generalisation

This architecture is domain-agnostic. Any personal knowledge graph can be expressed through this pipeline:

- Psychological health — psychometric scores, patterns, longitudinal state tracking
- Physical health — biomarkers, genetic risk, family history, medication outcomes encoded into anatomical visualisations
- Financial — asset/liability history as navigable landscape, value encoded as elevation, risk as colour
- Learning — knowledge maps where gap analysis driveswhat the agent surfaces next

The interface paradigm: your data, rendered as a world you inhabit and navigate by touch.

---
## Differences from Prior Art

Steganography hides data in low-order bit manipulations for security or watermarking — the goal is concealment from all parties.
Semantic pixel encoding is the opposite: the encoded layer is the primary data source for the rendering pipeline, decoded in real-time by a programmable graphics stage, for the explicit purpose of UI interaction semantics.

QR codes are visible, static, and require a separate scanning step. This mechanism is invisible, dynamic, and decoded continuously at render time within the graphics pipeline.

No prior art is known for this specific combination: reserved color channel encoding → AI-generated imagery → WebGL real-time decode → touch interaction semantic extraction.

---
## Reduction to Practice

A working proof-of-concept was implemented on May 4, 2026 including:
- Cluster detection algorithm identifying high-luminance elements
- Post-processing encoding script
- WebGL fragment shader with decode and display transform
- Touch interaction handler reading original texture semantics

Author: Dr. Benjamin A. Laken

Date: May 4, 2026

Location: Tbilisi, Georgia

If you build on this, say where it came from.
