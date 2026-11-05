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
- Page books: 800 × 480 pixels, 1 bit per pixel (black / white). Text is drawn
  sharp; pictures are dithered (newspaper-style dots). PDFs are fitted to the
  screen width and cut into screen-high strips so the text stays readable.

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
| Data | `b00c1700-0003-4b1e-8f3a-5e2d6c7a9b10` | write without response, up to ATT_MTU − 3 bytes |

Control messages (little endian):

| Direction | Message |
|---|---|
| page → BookiT | `0x01` START `u32 size` + file name · `0x02` END `u32 crc32` · `0x03` ABORT |
| BookiT → page | `0x81` READY `u16 max chunk` · `0x82` ACK `u32 bytes saved` · `0x83` DONE `u32 size` · `0xE0` ERROR `u8 code` |

The page keeps at most 8 KB un-acknowledged. BookiT ACKs every 4 KB **after**
the bytes are on its SD card, writes to `name.part`, and renames it only when
the CRC32 (IEEE) matches.

**`.bkp` page book format** (little endian): `"BKP1"`, `u16 width`, `u16 height`,
`u32 page count`, `u32 reserved`, `u32 offset[page count + 1]`, then each page
as a PackBits-compressed 48 000-byte frame (100 bytes per row, bit 7 = leftmost
pixel, 1 = white). The bottom 32 rows are left blank for BookiT's page number.

Libraries loaded from cdnjs: [pdf.js](https://mozilla.github.io/pdf.js/) 3.11.174
(Apache-2.0) and [JSZip](https://stuk.github.io/jszip/) 3.10.1 (MIT).
