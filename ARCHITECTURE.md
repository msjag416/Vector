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

**From a prompt.** The prompt gets some style words added to the end (for example "flat vector illustration, solid colors, plain white background") so the AI draws something that converts cleanly. The picture comes from one of two AI services:

- **Free (Pollinations.ai):** the page asks one web address for the picture and gets it back. No account or key.
- **KIE.ai (paid):** the page sends the prompt to KIE.ai's `jobs/createTask` address using the `google/nano-banana` model, then checks `jobs/recordInfo` every 2 seconds until the picture is ready, then downloads it. This needs the person's own KIE.ai key.

**From an image.** The person picks a picture from their device. It never leaves the device.

## Converting to vector (both ways end here)

1. Large pictures are shrunk (800, 1024 or 1280 pixels on the longest side for Low, Medium or High detail) so it runs fast on an iPad. Small pictures are left as they are.
2. The picture is drawn onto a hidden canvas with a white background behind any see-through parts.
3. VTracer groups similar colors (up to the number chosen in Colors), then draws smooth curves around each color area. Shapes are stacked on top of each other, so there are no cracks between them. The Detail setting controls how small a shape can be before it's thrown away and how closely two colors must match to be treated as one.
4. The file gets a `viewBox` (the scaling info vector apps need to enlarge it cleanly) and a starting size of 2048 points on its longest side.
5. If "Remove white background" is ticked, any near-white shapes are deleted so the background is see-through.
6. The result is shown next to the original. "Re-trace" runs steps 1 to 5 again with new settings without asking the AI for a new picture. "Download SVG" saves the file.

Why VTracer: the first version used a tool called ImageTracer, which copied the tiny pixel steps along every edge. That looked rough when enlarged. VTracer fits smooth curves instead.

## What gets saved on the device

Only two things are remembered, in the browser's own storage: which AI service was picked, and the KIE.ai key. Nothing is sent anywhere except the prompt (to the chosen AI service).

## Known limits

The AI draws a normal picture first and then it's converted, so the best results come from flat art, logos, icons and stickers. Detailed photos turn into poster-style color blocks, and photos on High detail can make large files (a few MB) that are slower to open.
