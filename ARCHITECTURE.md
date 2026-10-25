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

**Upscale (third tab).** This doesn't make a vector. It sends a picture to KIE.ai's Topaz upscaler (the Topaz Gigapixel engine) and gets back a 2x or 4x larger, sharper picture. Topaz needs the picture at a web address, so the app first sends it to KIE's upload service (`file-base64-upload`), then starts a `topaz/image-upscale` job and checks back every 2 seconds, the same way the prompt tab does. Needs the person's KIE.ai key and costs a few cents per picture. Pictures must be JPEG, PNG or WebP under 10 MB.

## Converting to vector (the prompt and image tabs end here)

1. Large pictures are shrunk (800, 1024 or 1280 pixels on the longest side for Low, Medium or High detail) so it runs fast on an iPad. Small pictures are left as they are.
2. The picture is drawn onto a hidden canvas with a white background behind any see-through parts.
3. If "Clean up colors" is ticked (it is by default), the picture is first resized to 2048 pixels on its longest side, then the app finds the picture's real colors and snaps every pixel to the nearest one. AI pictures have soft, blended edges around letters and shapes, and without this step those blends become pale halos and smeared edges in the vector. The app counts colors in coarse steps so grain doesn't split one color into dozens, merges near-duplicates, drops colors covering less than 0.05% of the picture, and drops any color that sits between two bigger colors (pink between red and white is an edge blend, not a real color). The surviving colors are handed to VTracer as a fixed palette. When Colors is set to 32 or 64 (meant for painterly pictures with shading), the halo filter is skipped and near-duplicate colors are merged less eagerly, so real shades survive. Those two settings make bigger files (a few MB) and take longer (around 10 seconds). Known limit: very small or faint text may take on a neighboring color.
4. VTracer groups similar colors (up to the number chosen in Colors), then draws smooth curves around each color area. Shapes are stacked on top of each other, so there are no cracks between them. The Detail setting controls how small a shape can be before it's thrown away and how closely two colors must match to be treated as one.
5. The file gets a `viewBox` (the scaling info vector apps need to enlarge it cleanly) and a starting size of 2048 points on its longest side.
6. If "Remove background" is ticked, the app finds the background color (the most common color along the picture's four edges) and deletes every shape close to that color, so the background is see-through. This works for white, black, cream or any other solid background. In this case VTracer runs in "cutout" mode (real holes in shapes) instead of "stacked" mode, so the hole in a letter O stays open instead of showing the color beneath it.
7. The result is shown next to the original. A Zoom slider (1x to 20x) makes the preview larger than its box so you can scroll around and check edges. "Re-trace" runs steps 1 to 5 again with new settings without asking the AI for a new picture. "Download SVG" saves the file.
8. "Download PNG" draws the vector onto a hidden canvas at the chosen size (1024, 2048 or 4096 pixels on the longest side) and saves that as a normal picture. If the background was removed, the PNG is see-through there too.

Why VTracer: the first version used a tool called ImageTracer, which copied the tiny pixel steps along every edge. That looked rough when enlarged. VTracer fits smooth curves instead.

## What gets saved on the device

Only two things are remembered, in the browser's own storage: which AI service was picked, and the KIE.ai key. Nothing is sent anywhere except the prompt (to the chosen AI service) and, on the Upscale tab, the picture being upscaled (to KIE.ai).

## Known limits

The AI draws a normal picture first and then it's converted, so the best results come from flat art, logos, icons and stickers. Detailed photos turn into poster-style color blocks, and photos on High detail can make large files (a few MB) that are slower to open.
