# How Vector Maker Works

Vector Maker is one web page that turns a written prompt or an uploaded picture into a vector file (SVG). There's no server and no database. Everything runs inside the browser on your device.

## The files

| File | What it does |
|------|--------------|
| `index.html` | The whole app: the layout, the look, and all the logic. |
| `vtracer.js` | VTracer 1.0.0-alpha.4 by visioncortex (free, MIT or Apache-2.0 license), the tool that converts a normal picture into smooth vector shapes. Copied from the npm package `@visioncortex/vtracer`. The only change is the loader at the bottom, so it runs in a web browser instead of on a server. |
| `vtracer.wasm` | The engine behind `vtracer.js`. It's fast compiled code the browser downloads once, the first time you convert something. |
| `ARCHITECTURE.md` | This file. |
| `README.md` | Short instructions for using and hosting the app. |

## The two ways in

**From a prompt.** The prompt gets some style words added to the end (for example "flat vector illustration, solid colors, plain white background") so the AI draws something that converts cleanly. The Style menu picks which words: Flat illustration, Logo or icon, Lettering (bold rounded letters shaped to fit the design, for word-based logos), Sticker, or Line art. The picture comes from one of two AI services:

- **Free (Pollinations.ai):** the page asks one web address for the picture and gets it back. No account or key.
- **KIE.ai (paid):** the page sends the prompt to KIE.ai's `jobs/createTask` address using the `google/nano-banana` model, then checks `jobs/recordInfo` every 2 seconds until the picture is ready, then downloads it. This needs the person's own KIE.ai key.

**From an image.** The person picks a picture from their device. It never leaves the device.

**Upscale (third tab).** This doesn't make a vector. It sends a picture to KIE.ai's Topaz upscaler (the Topaz Gigapixel engine) and gets back a 2x or 4x larger, sharper picture. Topaz needs the picture at a web address, so the app first sends it to KIE's upload service (`file-base64-upload`), then starts a `topaz/image-upscale` job and checks back every 2 seconds, the same way the prompt tab does. Topaz does at most 4x per pass, so 8x, 16x and 32x run it more than once, feeding each result back in (8x is 4x then 2x). Each pass costs a few cents. An Upscaler menu picks the engine: Topaz (default) or Recraft Crisp, which takes no size factor and picks its own (so the "How much bigger" choice only applies to Topaz). The choice is remembered on the device. If KIE answers "internal error" (which it sometimes does for no reason), the app retries Topaz once, then falls back to KIE's Recraft Crisp upscaler (`recraft/crisp-upscale`) so the job still finishes. JPG, PNG and WebP files are sent as they are; other types the browser can open are converted to PNG first. KIE caps the finished picture at 20,000 pixels on the longest side and uploads at 10 MB, and the app checks both before starting. The "From an image" tab also has an "Upscale 2x with Topaz first" checkbox that runs one pass before tracing, which gives the tracer cleaner edges on small or blurry pictures. Pictures must be JPEG, PNG or WebP under 10 MB.

## Converting to vector (the prompt and image tabs end here)

1. Large pictures are shrunk (800, 1024 or 1280 pixels on the longest side for Low, Medium or High detail) so it runs fast on an iPad. Small pictures are left as they are.
2. The picture is drawn onto a hidden canvas with a white background behind any see-through parts.
3. If "Clean up colors" is ticked (it is by default), the picture is first resized to 2048 pixels on its longest side, then the app finds the picture's real colors and snaps every pixel to the nearest one. AI pictures have soft, blended edges around letters and shapes, and without this step those blends become pale halos and smeared edges in the vector. The app counts colors in coarse steps so grain doesn't split one color into dozens, merges near-duplicates, drops colors covering less than 0.05% of the picture, and drops any color that sits between two bigger colors (pink between red and white is an edge blend, not a real color). The surviving colors are handed to VTracer as a fixed palette. When Colors is set to 32 or 64 (meant for painterly pictures with shading), the halo filter is skipped and near-duplicate colors are merged less eagerly, so real shades survive, and colors are counted in finer steps (32 levels per channel instead of 16) so smooth gradients keep more bands. Those two settings make bigger files (a few MB) and take longer (around 10 seconds). Known limit: very small or faint text may take on a neighboring color.
4. The Shading menu has three choices. "Flat colors" is the classic traced look. "Smooth gradients" fits a real gradient to each traced shape afterwards (step 6 below). "Exact" keeps the traced outlines for crisp edges and shows the original picture inside them (also step 6). Both non-flat choices turn on color clean-up and use cutout mode so each shape is measured from its own pixels only, and choosing either drops a color count above 8 down to 2.
5. VTracer groups similar colors (up to the number chosen in Colors), then draws smooth curves around each color area. Shapes are stacked on top of each other, so there are no cracks between them. The Detail setting controls how small a shape can be before it's thrown away and how closely two colors must match to be treated as one.
6. Shading. "Exact": the app builds a `clipPath` from all the remaining traced shapes, stores the working-size picture inside the SVG as a PNG (`<image href="data:...">`) with everything within 2 px of the outlines made see-through (so the soft edge pixels of the source never peek out), and draws it clipped to those shapes. The edges are vector and stay crisp at any size; the inside is a picture, which is fine for soft shading because a gradient has no fine detail to lose. The file is bigger (a few hundred KB) and the colors inside can't be edited in a vector app. "Smooth gradients": for each shape, the app draws the shape on a small hidden canvas to find which pixels are inside it, reads those pixels from the original picture, and fits "color = base + u*x + v*y" for red, green and blue by least squares. The direction the three color slopes agree on becomes the gradient direction. The average color is measured at six points along that direction and written into the SVG as a `linearGradient` with six stops, and the shape's fill (and its seam-covering outline) point at it. Shapes whose color changes by less than about 24 (out of 765) keep their flat fill, so text and flat art are untouched. Works best with 2 to 8 colors, so each shape stays one region; at higher color counts a gradient shape splits into several regions that each get their own gradient, which can leave a visible seam.
7. The file gets a `viewBox` (the scaling info vector apps need to enlarge it cleanly) and a starting size of 2048 points on its longest side.
8. If "Remove background" is ticked, the app finds the background color (the most common color along the picture's four edges) and deletes every shape close to that color, so the background is see-through. This works for white, black, cream or any other solid background. In this case VTracer runs in "cutout" mode (real holes in shapes) instead of "stacked" mode, so the hole in a letter O stays open instead of showing the color beneath it. Cutout mode can leave hairline gaps where shapes meet, so each remaining shape also gets a thin outline in its own color to cover them.
9. The result is shown next to the original. A Zoom slider (1x to 20x) makes the preview larger than its box so you can scroll around and check edges. "Re-trace" runs steps 1 to 5 again with new settings without asking the AI for a new picture. "Download SVG" saves the file.
10. "Download PNG" draws the vector onto a hidden canvas at the chosen size (1024, 2048, 4096 or 8192 pixels on the longest side; 8192 may fail on older iPads because of browser memory limits) and saves that as a normal picture. If the background was removed, the PNG is see-through there too.

Why VTracer: the first version used a tool called ImageTracer, which copied the tiny pixel steps along every edge. That looked rough when enlarged. VTracer fits smooth curves instead.

## What gets saved on the device

Three things are remembered, in the browser's own storage: which AI service was picked, which upscaler was picked, and the KIE.ai key. Nothing is sent anywhere except the prompt (to the chosen AI service) and, on the Upscale tab, the picture being upscaled (to KIE.ai).

## Known limits

The AI draws a normal picture first and then it's converted, so the best results come from flat art, logos, icons and stickers. With Flat colors shading, smooth gradients become bands of flat color (more bands at 32 or 64 colors), since traced shapes are single flat colors. With Smooth gradients, simple linear fades come back as real gradients; radial or multi-direction fades are approximated by the closest straight-line gradient. With Exact, the shading always matches the source. Detailed photos turn into poster-style color blocks, and photos on High detail can make large files (a few MB) that are slower to open.
