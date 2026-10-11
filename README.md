# QR Code Generator

Make styled QR codes in the browser and download them as PNG, JPG or SVG.

Live site: https://cytsai1008.github.io/qrcode-generator/

- Live preview while you edit
- Dot, corner square and corner dot styles and colors
- Error correction level (L, M, Q, H)
- Transparent background (PNG and SVG; JPG falls back to white)

Built with [Nuxt UI](https://ui.nuxt.com) and [qr-code-styling](https://github.com/kozakdenys/qr-code-styling).

## Development

```bash
npm install
npm run dev        # http://localhost:3000
npm run lint
npm run typecheck
npm run generate   # static site in .output/public
```

## Deployment

Every push to `main` builds the site and deploys it to GitHub Pages
(`.github/workflows/deploy.yml`).
