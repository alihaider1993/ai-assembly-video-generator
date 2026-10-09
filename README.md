# AI Assembly Video Generator

**Upload a flat-pack furniture manual. Get back an animated 3D video of it being built, step by step.**

[![Live demo](https://img.shields.io/badge/Live%20demo-open%20the%20site-ff5a1f?style=for-the-badge)](https://ai-assembly-video-generator.syed-ali-haider.workers.dev)
[![Examples](https://img.shields.io/badge/Examples-10%20real%20manuals-2d6cdf?style=for-the-badge)](examples/README.md)

![The manual and the generated video, side by side](assets/demo.gif)

*Left: a page of the paper manual. Right: the video generated from it. Every step in the drawings becomes a captioned 3D animation.*

## Contents

- [Why](#why)
- [Try it](#try-it)
- [What it does](#what-it-does)
- [How it works: the AI pipeline](#how-it-works-the-ai-pipeline)
- [Examples](#examples)
- [What it can build](#what-it-can-build)
- [Inside a video](#inside-a-video)
- [The website](#the-website)
- [Limits](#limits)
- [How the project grew](#how-the-project-grew)
- [Lessons learned](#lessons-learned)
- [Roadmap](#roadmap)
- [FAQ](#faq)
- [Tech stack](#tech-stack)
- [Author](#author)

## Why

Flat-pack manuals have no words. A few line drawings carry every step, and it is easy to misread which panel faces which way, which of five similar screws goes where, or what "x4" refers to. A short video that shows each part moving into place, with the step written underneath, is much easier to follow. This project makes that video from the manual itself, for any product, with no hand animation.

It started when I moved to the UK. Almost everything I bought needed self-assembly, and working out the diagrams often took longer than building the thing. One misread step meant undoing the work and starting again. I looked for assembly videos on YouTube, but for many products there were none. So I asked: **what if AI could read any assembly manual and make the video automatically?**

## Try it

**[ai-assembly-video-generator.syed-ali-haider.workers.dev](https://ai-assembly-video-generator.syed-ali-haider.workers.dev)**

- **Watch the examples:** ten real IKEA and Habitat manuals. As a video plays, the manual turns to the matching page and the parts list ticks itself off. Click any step to jump to it.
- **Upload your own manual:** a PDF of up to 10 pages and 2 MB. The video is made in a few minutes, with live progress, while the render machine is online.

## What it does

1. **Reads the manual.** It goes through the drawings page by page: the parts and fittings, how many of each, the sizes where the manual prints them, and what happens on each page.
2. **Works out the finished piece.** It decides where every part sits in the assembled furniture, which parts join which, and the order they go together, including steps that say "repeat x4" or show only one of several identical parts.
3. **Plans the motion.** Each part slides in from a clear direction at its step. Screws and bolts travel into their holes, and pieces built separately (a drawer, a chaise base with its feet) move into place as one unit.
4. **Renders it in 3D.** Every part is modelled to size and shape, the camera circles the piece, and the steps are captioned, e.g. *"Insert 4 x hex head bolt into leg"*.
5. **Delivers an MP4** with the product's name at the top and every step captioned, ready to watch or download.

![Studio: the manual page, the video and the steps in sync](assets/studio.png)

## How it works: the AI pipeline

The pipeline does not jump straight from a PDF to an animation. Over a chain of stages it builds up a structured understanding of how the product goes together. Each stage writes structured output that the next stage reads, so every decision can be inspected and explained.

```mermaid
flowchart TD
    A[Assembly manual PDF] --> B[PDF page extraction]
    B --> C[Vision reading<br/>parts, fittings, counts, sizes, actions per page]

    subgraph U[Understanding]
        D[Assembly delta detection<br/>what changed from page to page]
        E[Assembly action extraction]
        F[Object identity resolution<br/>the same part across pages]
        G[Action replication<br/>Repeat x4, identical parts]
        H[Universal Assembly Graph]
        D --> E --> F --> G --> H
    end

    subgraph P[Animation planning]
        I[Motion planner]
        J[Part shapes and roles]
        K[Scene layout engine]
        L[Sub-assembly staging]
        M[Procedural proxy geometry]
        I --> J --> K --> L --> M
    end

    C --> D
    H --> I
    M --> N[Blender scene generator]
    N --> O[Headless Blender rendering]
    O --> Q[Captioned MP4 video]
```

### The stages

1. **PDF page extraction.** Each page of the manual becomes an image.
2. **Vision reading.** A vision model reads every page and records the parts and fittings drawn, how many of each, the sizes where they are printed, and the action on that page (insert, attach, screw, turn over). Each page is checked against a strict schema, and every part, fitting and diagram gets a stable ID.
3. **Assembly delta detection.** Pages are compared one after another to find what was added at each step.
4. **Assembly action extraction.** The changes become actions: which part or fitting joins which, and how. Where a page shows only the screws, the joined parts are worked out from the evidence.
5. **Object identity resolution.** The same part is tracked across pages, so the "side panel" on page 3 and on page 12 is one object, and each action is pointed at the right instance.
6. **Action replication.** "Repeat x4", or one leg drawn when four get fitted, becomes the right number of actions across the matching parts, capped at what the manual shows.
7. **Universal Assembly Graph.** All parts, fittings and joins go into one graph of what connects to what, and in what order. It is built twice: once from the raw actions and again after identity resolution and replication.
8. **Motion planning.** Each part is given exactly one step at which it moves, plus the direction it comes from.
9. **Part shapes and roles.** Every part gets a role from its name (leg, rail, side panel, shelf, door, drawer front, slat, cushion, ...) and a size from the manual's measurements, or from sensible defaults for that role when none are printed. Curved parts keep the outline traced from the drawings.
10. **Scene layout engine.** One general layout places every product. It treats the piece as a frame of uprights, walls, beams, plates and blocks: walls stand on the floor, rails stretch between the parts they join, shelves sit between side walls, doors and drawers fill the front opening, and brackets go into inside corners. The assembly graph decides where each part goes. Name words like *left*, *front* or *top* only break ties.
11. **Sub-assembly staging.** Parts built separately, such as a drawer or a chaise base with its feet, are put together beside the main piece and then move into place as one unit. A piece built upside down is turned over at the end.
12. **Procedural proxy geometry.** Clean 3D shapes are made to the right size, including extruded outlines for bentwood frames and curved slats.
13. **Blender scene generation.** A Blender Python script is generated with the parts, materials, lighting, a camera that circles the piece, and a timeline that merges repeated moves.
14. **Headless rendering.** Blender renders the frames with no window open.
15. **Video.** MoviePy joins the frames into an MP4, with a caption for each step and the product's name at the top.

### What makes it general

- **Nothing is special-cased.** Every example went through the same stages, with no per-product code or hand-placed parts.
- **Explainable.** Every stage's output is structured, so a wrong part position can be traced back to the page and the action it came from.
- **Robust to missing information.** It estimates sizes when the manual prints none, infers joins when only screws are shown, and expands "repeat" steps.

## Examples

Every example has its own page with the video, a frame from each step, the full step list with the manual page each step comes from, and the parts list. **[Browse all examples →](examples/README.md)**

| Product | Type | Manual pages | Steps | Parts and fittings | Video |
|---|---|---:|---:|---:|---:|
| [IKEA MELLTORP Table](examples/melltorp_table/README.md) | Table | 8 | 8 | 41 | [0:22](examples/melltorp_table/video.mp4) |
| [Habitat Banbury Chair](examples/banbury_chair/README.md) | Chair | 6 | 6 | 35 | [0:17](examples/banbury_chair/video.mp4) |
| [Habitat Kaycie Bed](examples/kaycie_bed/README.md) | Bed | 15 | 15 | 114 | [0:32](examples/kaycie_bed/video.mp4) |
| [IKEA NORDEN Extendable Table](examples/norden_table/README.md) | Table, no sizes printed | 20 | 21 | 128 | [0:43](examples/norden_table/video.mp4) |
| [IKEA ÅLHULT Chair](examples/alhult_chair/README.md) | Chair | 8 | 11 | 36 | [0:26](examples/alhult_chair/video.mp4) |
| [IKEA POÄNG Armchair](examples/poang_chair/README.md) | Bentwood armchair | 12 | 11 | 42 | [0:26](examples/poang_chair/video.mp4) |
| [IKEA KIVIK Chaise Longue](examples/kivik_chaise_longue/README.md) | Upholstered sofa | 16 | 11 | 40 | [0:26](examples/kivik_chaise_longue/video.mp4) |
| [IKEA HEMNES 8-Drawer Dresser](examples/hemnes_dresser/README.md) | Chest of drawers | 48 | 29 | 357 | [0:55](examples/hemnes_dresser/video.mp4) |
| [IKEA BRIMNES Wardrobe](examples/brimnes_wardrobe/README.md) | Wardrobe | 36 | 26 | 176 | [0:50](examples/brimnes_wardrobe/video.mp4) |
| [IKEA RAKKESTAD Wardrobe](examples/rakkestad_wardrobe/README.md) | Wardrobe | 32 | 24 | 154 | [0:46](examples/rakkestad_wardrobe/video.mp4) |

The largest, the HEMNES dresser, goes from a 48-page manual with 357 parts and fittings to a 55-second video:

![HEMNES dresser: frames from across the build](examples/hemnes_dresser/steps.jpg)

## What it can build

The same system handles every product above. Nothing is set up by hand for a particular manual.

- **Tables and chairs:** legs, rails, corner brackets, tabletops, and seats and backs with slats.
- **Beds:** headboards, footboards, side rails, centre supports and slats.
- **Cabinets, dressers and wardrobes:** side panels, dividers, shelves, back panels and plinths; doors with hinges; drawers with their sides, backs, bottoms, runners and handles.
- **Upholstered furniture:** sofa sections, chaises, feet and cushions, with the hardware that joins them.
- **Curved parts:** bentwood frames and curved slats are drawn to their real outline, not as boxes.
- **Fittings:** screws, bolts, cam locks, dowels, plugs and washers, in the counts the manual gives.
- **Things manuals do that a computer finds hard:**
  - "Repeat x4", or one leg drawn when four get fitted
  - pieces built separately and then joined (a drawer, a chaise base)
  - furniture built upside down and turned over at the end
  - manuals that print no sizes at all (NORDEN): sizes are estimated from the drawings

## Inside a video

- **Title band:** the product's name across the top, with the furniture kept clear of it.
- **Captions:** one per step, e.g. *"Attach 2 x drawer side to small drawer front"*, with the step number and the fitting count from the manual.
- **Motion:** the parts for each step are highlighted and slide in. Small fittings are shown larger while they move so they can be seen, then shrink to their real size.
- **Pacing:** consecutive steps that do the same thing (fit four legs, insert eight plugs) are merged into one smooth move.
- **Camera:** circles the piece as it grows and finishes looking at the front of the finished furniture.
- **Finish:** a turn-over when the piece was built upside down, then a last look at the assembled product.

## The website

![Gallery of examples](assets/gallery.png)

- **Gallery:** every example with its poster, page count and length.
- **Studio:** the video, the manual and the parts list side by side and kept in sync. The manual turns to the page for the current step, the parts used so far are ticked off, and clicking a step jumps the video there.
- **Upload:** drop in a PDF manual and follow its progress in plain language ("Reading the manual", "Animating", ...). The finished video plays in the same studio view and can be downloaded.

![Upload your own manual](assets/upload.png)

## Limits

- Uploads are up to **10 pages and 2 MB**. The examples show it working on manuals of up to 48 pages.
- Uploads are rendered on the author's own machine, so they are open only while it is online. The examples are always available.
- Parts are shown as clean, simplified 3D shapes in the right size and place, not photo-realistic models.
- Manuals that are mostly text, or scanned at low quality, give weaker results than clear line drawings.

## How the project grew

The first version was an experimental MVP built on **Azure AI Foundry and Azure OpenAI GPT-4o Vision**, with a Streamlit front end. It proved that the whole chain worked, from PDF to graph to Blender to MP4. But the parts were generic boxes, the motion was functional rather than realistic, and complex manuals confused it.

The current version keeps the staged architecture and rebuilds most of it:

| | First MVP | Now |
|---|---|---|
| Manual reading | Azure GPT-4o Vision | Gemini vision, checked against a strict per-page schema |
| Layout | Exploded ring of parts | One general layout that builds the real frame for every product |
| Geometry | Generic proxy boxes | Parts sized from the manual, with curved outlines traced from the drawings |
| Motion | One part at a time | Repeated moves merged, separately built parts moved as one unit, a final turn-over |
| Video | Plain render | Step captions with fitting counts, product title, a camera that ends at the front |
| Interface | Local Streamlit app | Public website with a gallery, a synced studio view and live uploads |
| Quality checks | Manual | Continuous integration runs every example manual through the pipeline on each change |

## Lessons learned

- **Most of the work is understanding, not rendering.** Drawing 3D boxes is easy. Knowing which box goes where, and when, is the hard part.
- **Structure beats one big prompt.** Splitting the problem into small stages with structured outputs made errors visible and fixable.
- **Generalise, don't special-case.** Each new kind of furniture (beds, wardrobes, sofas) was handled by improving the one general layout, not by adding a layout for that product.
- **Manuals leave things out.** Sizes, counts and joins are often only implied, so the pipeline has to infer them on purpose.

## Roadmap

- **Understanding:** multi-view reasoning, and better handling of dense, text-heavy pages.
- **Graphics:** CAD-aware and parametric parts, and more realistic materials.
- **Animation:** collision-aware motion, and tool animations (an Allen key turning).
- **Experience:** voice narration, step-by-step playback controls, and uploads that are always on.
- **Products:** beyond furniture, to bicycles, gym equipment, toys and DIY kits.

## FAQ

**Does it work with any brand?**
It is built for the wordless line-drawing style that IKEA, Habitat and most flat-pack brands use. The examples include both brands.

**How long does a video take?**
A few minutes for a typical upload. Most of that is 3D rendering.

**Is anything set up by hand for each product?**
No. A new manual goes through the same steps as every example here.

**Can I get the source code?**
The source is private. This repository shows what the project does: the demo, the examples and the website. For a walkthrough, get in touch below.

## Tech stack

- **Python**
- **Blender** for 3D modelling, animation and rendering
- **Google Gemini** (vision) for reading the manuals' drawings (the first MVP used Azure OpenAI GPT-4o Vision on Azure AI Foundry)
- **Blender Python (bpy)** scripts generated by the pipeline
- **MoviePy** for the final video and its captions
- **GitHub Actions** for continuous integration
- **Cloudflare Workers and Durable Objects** for the website and uploads
- **HTML, CSS and JavaScript** for the site, with no framework

## Author

**Syed Ali Haider**: [LinkedIn](https://www.linkedin.com/in/syed-ali-haider1993) · [GitHub](https://github.com/alihaider1993)

---

Manuals shown belong to their makers and are used for demonstration only. © 2026 Syed Ali Haider. All rights reserved.
