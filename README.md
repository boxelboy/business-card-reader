# Business Card Reader (Vue 3)

Vue 3 app that:

- opens the phone camera
- captures a business card image
- uses OCR (`tesseract.js`) to extract text
- maps likely fields (name/company/title/email/phone/website/address)
- posts JSON to `https://boxel.co.uk/api/cardreader`

## Run

```bash
npm install
npm run dev -- --host
```

Open the URL shown by Vite on your phone (same network) and allow camera access.

## Notes

- Browser must support `navigator.mediaDevices.getUserMedia`.
- For camera on mobile, HTTPS is recommended (or localhost in development).
- OCR quality improves with sharp image, good lighting, and card filling most of the frame.
