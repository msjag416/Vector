# How Vector Maker Works

Vector Maker is one web page that turns a written prompt or an uploaded picture into a vector file (SVG). There's no server and no database. Everything runs inside the browser on your device.

## The files

| File | What it does |
|------|--------------|
| `index.html` | The whole app: the layout, the look, and all the logic. |
| `imagetracer.js` | A free, public-domain tool (ImageTracer 1.2.6 by András Jankovics) that converts a normal picture into vector shapes. It's copied into the project so the app doesn't depend on another website. |
| `ARCHITECTURE.md` | This file. |
| `README.md` | Short instructions for using and hosting the app. |

## The two ways in

**From a prompt.** The prompt gets some style words added to the end (for example "flat vector illustration, solid colors, plain white background") so the AI draws something that converts cleanly. The picture comes from one of two AI services:

- **Free (Pollinations.ai):** the page asks one web address for the picture and gets it back. No account or key.
- **KIE.ai (paid):** the page sends the prompt to KIE.ai's `jobs/createTask` address using the `google/nano-banana` model, then checks `jobs/recordInfo` every 2 seconds until the picture is ready, then downloads it. This needs the person's own KIE.ai key.

**From an image.** The person picks a picture from their device. It never leaves the device.

## Converting to vector (both ways end here)

1. The picture is shrunk if it's large (500, 800 or 1200 pixels wide for Low, Medium or High detail) so it runs fast on an iPad.
2. It's drawn onto a hidden canvas with a white background behind any see-through parts.
3. ImageTracer groups the pixels into the number of colors chosen (2 to 32) and draws smooth outlines around each color area. The Detail setting controls how closely those outlines follow the picture and how small a shape can be before it's thrown away.
4. Each shape gets a 1-pixel outline in its own color so no white cracks show between shapes.
5. If "Remove white background" is ticked, any near-white shapes are deleted so the background is see-through.
6. The result is shown next to the original. "Re-trace" runs steps 1 to 5 again with new settings without asking the AI for a new picture. "Download SVG" saves the file.

## What gets saved on the device

Only two things are remembered, in the browser's own storage: which AI service was picked, and the KIE.ai key. Nothing is sent anywhere except the prompt (to the chosen AI service).

## Known limits

The AI draws a normal picture first and then it's converted, so the best results come from flat art, logos, icons and stickers. Detailed photos turn into poster-style color blocks.
