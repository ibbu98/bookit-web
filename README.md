# BookiT Sender

Web page that sends books to the **BookiT** e-reader (a solar-powered BLE e-paper
reader, ECEN 5833 course project) over Bluetooth.

**Open it:** https://ibbu98.github.io/bookit-web/
(or scan the QR code on BookiT: *Settings → Bluetooth*).

- Pick a **PDF, EPUB or TXT** file on your phone or laptop.
- The page converts it **on your device**. Nothing is uploaded anywhere.
- It sends the book **straight to BookiT over Bluetooth**.

## Who talks to whom

BookiT has **no internet**, only Bluetooth (a few meters). GitHub never talks to
BookiT. The **phone is the bridge**:

```
  GitHub  ──(internet, once)──►  Phone  ──(Bluetooth)──►  BookiT
                                   ▲
  BookiT ──(QR code = a printed link, just a picture)──┘
```

1. **BookiT → phone:** the QR code is only a picture of this page's address.
2. **GitHub → phone:** the phone downloads this page once, over its own Wi-Fi or mobile data.
3. **Phone → BookiT:** the page, running in the phone's browser, uses the phone's
   Bluetooth to find BookiT, pair with the code shown on its screen, and send the book.

Once the page has loaded, the transfer works even without internet.

## What happens when you send a book

```
 BookiT                       Phone (Chrome)                    GitHub Pages
 ──────                       ──────────────                    ────────────
 Settings > Bluetooth
 shows QR code  ───scan──►  camera opens link  ──── https ───►  sends the page
 starts advertising                                              (once)
 "BookiT"                    page loads
                             1. you pick a PDF/EPUB/TXT
                                from the phone's files
                             2. page converts it ON THE PHONE
                                English → .txt / other → .bkp pages
                             3. "Connect & send"
           ◄── Bluetooth ──  Chrome lists "BookiT", you tap it
 shows 6-digit code
                             phone asks for the code, you type it
 paired ◄──────────────────
 "Receiving 45%"  ◄── data ─ sends the book in chunks
 writes to SD card, ──ack──► progress bar moves
 confirms every 4 KB
 checks CRC, renames  ─done─► "Done! saved on BookiT"
 "Saved!"
 Library → open → read
```

## Two book formats

| Format | Used for | On BookiT |
|---|---|---|
| **Text** `.txt` | English books | small; font size and line spacing can be changed on BookiT |
| **Pages** `.bkp` | any other language (Arabic, Tamil, ...), scanned PDFs, PDFs, books with pictures | pages are drawn here with your device's fonts, so every script works; fixed size |

**Auto** picks the format: if more than 0.5 % of the letters are not plain
English, or there is almost no text (a scanned PDF), it makes a page book.
You can force **Text** or **Pages** by hand.

- Text books are cleaned to plain ASCII (curly quotes → `"`, dashes → `-`,
  `é` → `e`). Pictures become `[Image]`.
- Page books: 1 bit per pixel (black / white), each page drawn for **portrait
  (480 × 800)** or **landscape (800 × 480)**. Text is drawn sharp; pictures are
  dithered (newspaper-style dots).
- PDFs: **Whole page** (default) fits every page whole on one screen in its own
  shape, so a portrait Quran page stays one portrait page. **Fit width** gives
  bigger text but cuts tall pages into strips.
- On BookiT, *Settings → Orientation* turns the whole device. A page whose shape
  doesn't match is shrunk to fit whole; switch Orientation to see it full size.

## What you need

| Device | Browser |
|---|---|
| Android | **Chrome** (allow "Nearby devices" the first time) |
| Windows / macOS / ChromeOS / Linux | **Chrome** or **Edge** |
| iPhone / iPad | Safari can't use Bluetooth from web pages; use the free **Bluefy** browser |

Locked (DRM) books, e.g. from Kindle or Adobe-protected EPUBs, can't be opened by anyone.

## Privacy and safety

- The book is converted inside your browser and sent only to BookiT.
- BookiT only accepts a book after **pairing with the 6-digit code shown on its
  screen**, so only the person holding it can send books.
- BookiT is only visible over Bluetooth while its Bluetooth screen is open.

## For developers

**BookiT Transfer** GATT service `b00c1700-0001-4b1e-8f3a-5e2d6c7a9b10`

| Characteristic | UUID | Use |
|---|---|---|
| Control | `b00c1700-0002-4b1e-8f3a-5e2d6c7a9b10` | write + notify |
| Data | `b00c1700-0003-4b1e-8f3a-5e2d6c7a9b10` | write without response: `u32 file offset` + bytes, up to ATT_MTU − 3 in total |

Control messages (little endian):

| Direction | Message |
|---|---|
| page → BookiT | `0x01` START `u32 size` + file name · `0x02` END `u32 crc32` · `0x03` ABORT |
| BookiT → page | `0x81` READY `u16 max packet` · `0x82` ACK `u32 bytes saved` · `0x83` DONE `u32 size` · `0x84` RESEND `u32 offset` · `0xE0` ERROR `u8 code` |

Loss-proof transfer:

- The page keeps at most 8 KB un-acknowledged. BookiT ACKs every 4 KB, and
  whenever data stops coming, **after** the bytes are on its SD card.
- Every data packet carries its file offset. BookiT only takes the exact next
  offset: a gap (lost packet) or a full buffer → `RESEND` from where it is;
  a duplicate → it repeats its last ACK (in case that ACK was lost).
- If the page hears nothing for 3 s it goes back to the last ACK and sends again.
- BookiT writes to `name.part` and renames it only when the CRC32 (IEEE) matches.
- A new START always restarts cleanly (the phone's pairing prompt can hold up
  the first answer, so the page retries START for up to 90 s).

**`.bkp` page book format (BKP2)** (little endian): `"BKP2"`, `u16 width (800)`, `u16 height (480)`,
`u32 page count`, `u32 reserved`, `u32 offset[page count + 1]`, `u8 portrait[page count]`
(1 = page drawn as 480 x 800 portrait), padding to 4 bytes, then each page as a
PackBits-compressed 48 000-byte frame in the panel's physical 800 x 480 layout
(100 bytes per row, bit 7 = leftmost pixel, 1 = white). Portrait pages are stored
turned: physical x = y, physical y = 479 - x. The bottom 32 rows of each page (as
seen) are left blank for BookiT's page number. Older `"BKP1"` files (no portrait
table, all landscape) still open.

Libraries loaded from cdnjs: [pdf.js](https://mozilla.github.io/pdf.js/) 3.11.174
(Apache-2.0) and [JSZip](https://stuk.github.io/jszip/) 3.10.1 (MIT).
