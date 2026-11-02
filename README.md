# HPOS - Hollywood Ice Creams car order pad

Single-file web app (`index.html`) for taking car-side orders on an Android tablet or phone.

- Menu with sizes and prices, pictures, Arabic names
- Tap to add, +/- on each item, order review, total in AED
- Orders list with Edit, Cancel and Mark done, plus a "To enter" summary for keying into the main system
- Menu editor (Menu tab, default PIN 1234): items, sizes, prices, pictures, sold out
- Data is saved in the browser on the device only

Open `index.html` in Chrome and use "Add to Home screen".

## Android APK

Every push to `main` builds an APK with GitHub Actions (Capacitor, debug-signed).
Download `hollywood-orders.apk` from the **Releases** page (release `apk-latest`), or from the workflow run's Artifacts.
On the tablet, allow "Install unknown apps" for the browser, open the file and install.

## POS (sample)

`pos/index.html` is a sample counter POS web app (billing, VAT, receipts, sales, stock). It is published at `/pos/` on GitHub Pages. Data is saved in the browser only.
