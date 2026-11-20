# 💎 Fresh Tg Store — Telegram Premium UI Kit (Telethon)

Is ek file ko follow karke `bot.py` **premium** ban jayega:

1. 🎨 **Coloured buttons** — Telegram ke 3 official button styles: `primary` (blue) · `success` (green) · `danger` (red)
2. ⚡ `/start` pe **animated loading bar** (0% → 100%, 10-10 ke gap me, fast) → "✅ Successful" → message **auto delete**
3. 🛡 **Image captcha**: 5–6 letters, ek pe **circle**, niche utne hi **coloured circular letter buttons** (Ⓐ Ⓑ Ⓒ…) — circled letter wale button pe tap
4. 🤖 Captcha image jo **OCR / bots se easily solve na ho**
5. 📜 Verify ke baad → Terms & Conditions → main menu (sab coloured buttons ke saath)

---

## 🎨 0. Button styles kaise kaam karte hain

Telegram ne buttons ke liye 3 background styles diye hain:

| Style | Color | Kab use karo |
|---|---|---|
| `primary` | 🔵 Blue | Main actions: Buy, Open, Continue |
| `success` | 🟢 Green | Positive: Accept, Confirm, Add Funds, Verify |
| `danger` | 🔴 Red | Destructive: Cancel, Delete, Decline |

- **Bot API**: `InlineKeyboardButton` / `KeyboardButton` me `"style": "primary" | "success" | "danger"`.
  Style na do to client apna default style dikhata hai.
- **Raw MTProto (Telethon)**: button ke saath `style=KeyboardButtonStyle(bg_primary=True)` (ya `bg_success` / `bg_danger`). **Sirf ek** color flag set hota hai.
- **Optional icon**: Bot API me `icon_custom_emoji_id`, MTProto me `KeyboardButtonStyle(icon=<custom_emoji_id>)`.
  Ye sirf unhi bots ke liye hai jinka username Fragment se kharida gaya ho, ya jinke owner ke paas Telegram Premium ho.
- Raw docs: `keyboardButtonCallback` me `style` field hai (flag bit 10).

> ⚠️ Telethon me ye tabhi chalega jab tumhare installed Telethon me `KeyboardButtonStyle` class ho.
> Check karo:
> ```bash
> pip install -U telethon pillow
> python -c "from telethon.tl.types import KeyboardButtonStyle; print('OK')"
> ```
> `OK` aaya to sab set. `ImportError` aaya to neeche **Section 3.7 (Bot API fallback)** use karo. `ui_kit.py` me already emoji fallback (🔵🟢🔴) bhi hai taaki bot kabhi crash na kare.

---

## 📦 1. Install & folder structure

```bash
pip install -U telethon pillow
```

Fonts: `fonts/` me 3–4 alag `.ttf` fonts rakho (DejaVuSans-Bold, Roboto-Bold, OpenSans-ExtraBold…). Alag fonts OCR ko confuse karte hain.

```
project/
├── bot.py
├── ui_kit.py           ← coloured buttons helper (Section 2)
├── captcha_gen.py      ← captcha image generator (Section 3)
└── fonts/
    ├── font1.ttf
    ├── font2.ttf
    └── font3.ttf
```

---

## 🧰 2. `ui_kit.py` — coloured buttons helper (poore bot me use hoga)

```python
# ui_kit.py
from telethon.tl.types import (
    KeyboardButtonCallback, KeyboardButtonRow, ReplyInlineMarkup,
)

try:
    from telethon.tl.types import KeyboardButtonStyle
    STYLES_OK = True
except ImportError:            # purana Telethon → emoji fallback
    STYLES_OK = False

FALLBACK = {"primary": "🔵", "success": "🟢", "danger": "🔴"}


def cb(text: str, data, style: str = None) -> KeyboardButtonCallback:
    """Inline callback button. style = 'primary' | 'success' | 'danger' | None"""
    if isinstance(data, str):
        data = data.encode()
    if style and STYLES_OK:
        st = KeyboardButtonStyle(**{f"bg_{style}": True})
        return KeyboardButtonCallback(text=text, data=data, style=st)
    if style:                                   # fallback
        text = f"{FALLBACK[style]} {text}"
    return KeyboardButtonCallback(text=text, data=data)


def markup(*rows) -> ReplyInlineMarkup:
    """markup([btn, btn], [btn]) → inline keyboard"""
    return ReplyInlineMarkup([KeyboardButtonRow(list(r)) for r in rows])
```

Use example:

```python
from ui_kit import cb, markup

kb = markup(
    [cb("🛒 Buy Account", "buy", "primary"), cb("💰 Add Funds", "funds", "success")],
    [cb("❌ Cancel", "cancel", "danger")],
)
await event.respond("Choose 👇", buttons=kb)
```

---

## 🖼️ 3. Captcha

### 3.1 `captcha_gen.py` — Anti-bot image generator

Anti-bypass techniques:

- Har letter **random font, size, rotation, color**
- Letters thode **jitter/overlap** karte hain (segmentation tough)
- **Wavy distortion** (sine warp) poori image pe
- **Noise dots + curved lines** letters ke upar bhi
- **Circle** thoda imperfect (off-center, random thickness)
- **Decoy arcs** dusre letters pe (bot ko confuse karne ke liye)
- Confusing letters (`O, 0, I, 1, L, S, 5, B, 8`) hata diye gaye
- Captcha **single-use + expiry + attempt limit** (bot.py me)

```python
# captcha_gen.py
import io, math, random, glob
from PIL import Image, ImageDraw, ImageFont, ImageFilter

CHARS = "ACDEFGHJKMNPQRTUVWXYZ"   # confusing letters removed
W, H = 640, 260


def circ(ch: str) -> str:
    """'A' → 'Ⓐ'  (circular letter for the buttons)"""
    return chr(0x24B6 + ord(ch) - ord("A"))


def _rand_color(lo=40, hi=220):
    return tuple(random.randint(lo, hi) for _ in range(3))


def _wave(img: Image.Image) -> Image.Image:
    """Sine-wave distortion — OCR ke liye bada headache."""
    w, h = img.size
    out = Image.new("RGB", (w, h), (255, 255, 255))
    px_in, px_out = img.load(), out.load()
    ax, ay = random.uniform(4, 8), random.uniform(4, 8)
    fx, fy = random.uniform(0.015, 0.03), random.uniform(0.015, 0.03)
    for y in range(h):
        for x in range(w):
            sx = int(x + ax * math.sin(2 * math.pi * fy * y))
            sy = int(y + ay * math.sin(2 * math.pi * fx * x))
            if 0 <= sx < w and 0 <= sy < h:
                px_out[x, y] = px_in[sx, sy]
    return out


def generate_captcha(length: int = None):
    """Returns: (png_bytes, letters: list[str], answer_index: int)"""
    length = length or random.choice([5, 6])
    letters = random.sample(CHARS, length)
    answer_idx = random.randrange(length)

    fonts = glob.glob("fonts/*.ttf")
    bg = Image.new("RGB", (W, H), (random.randint(215, 250),) * 3)
    d = ImageDraw.Draw(bg)
    for _ in range(900):
        d.point((random.randint(0, W), random.randint(0, H)), fill=_rand_color(120, 235))

    slot = W // (length + 1)
    centers = []
    for i, ch in enumerate(letters):
        size = random.randint(86, 112)
        font = ImageFont.truetype(random.choice(fonts), size)
        tile = Image.new("RGBA", (size + 60, size + 60), (0, 0, 0, 0))
        ImageDraw.Draw(tile).text((30, 20), ch, font=font, fill=_rand_color(0, 130) + (255,))
        tile = tile.rotate(random.randint(-35, 35), expand=True, resample=Image.BICUBIC)
        cx = slot * (i + 1) + random.randint(-12, 12)
        cy = H // 2 + random.randint(-22, 22)
        bg.paste(tile, (cx - tile.width // 2, cy - tile.height // 2), tile)
        centers.append((cx, cy))

    # circle around the answer letter (thoda imperfect)
    d = ImageDraw.Draw(bg)
    cx, cy = centers[answer_idx]
    r = random.randint(68, 78)
    ox, oy = random.randint(-5, 5), random.randint(-5, 5)
    d.ellipse((cx - r + ox, cy - r + oy, cx + r + ox, cy + r + oy),
              outline=(220, 30, 60), width=random.randint(4, 6))

    # decoy partial arcs on OTHER letters
    for j in range(length):
        if j != answer_idx and random.random() < 0.5:
            x, y = centers[j]
            rr = random.randint(60, 80)
            a0 = random.randint(0, 300)
            d.arc((x - rr, y - rr, x + rr, y + rr), a0, a0 + random.randint(40, 110),
                  fill=(220, 30, 60), width=3)

    # strike-through curves + noise over everything
    for _ in range(random.randint(4, 7)):
        pts = [(random.randint(0, W), random.randint(0, H)) for _ in range(4)]
        d.line(pts, fill=_rand_color(60, 200), width=random.randint(2, 4), joint="curve")
    for _ in range(500):
        d.point((random.randint(0, W), random.randint(0, H)), fill=_rand_color(0, 255))

    img = _wave(bg).filter(ImageFilter.GaussianBlur(radius=0.8))
    buf = io.BytesIO()
    img.save(buf, "PNG")
    return buf.getvalue(), letters, answer_idx
```

### 3.2 `bot.py` — imports & state

```python
import asyncio, time, random
from telethon import TelegramClient, events
from captcha_gen import generate_captcha, circ
from ui_kit import cb, markup

# user_id -> {"answer": "K", "expires": ts, "tries": 0}
PENDING = {}
VERIFIED = set()          # production me sqlite me save karo (users table: `verified` column)

CAPTCHA_TTL = 120         # seconds
MAX_TRIES = 3
```

### 3.3 Premium loading bar (0 → 100, 10-10 step, fast)

```python
async def loading_bar(event):
    msg = await event.respond("⏳ **Initializing...**\n\n`▱▱▱▱▱▱▱▱▱▱` **0%**")
    for pct in range(10, 101, 10):
        bar = "▰" * (pct // 10) + "▱" * (10 - pct // 10)
        try:
            await msg.edit(f"⚡ **Loading...**\n\n`{bar}` **{pct}%**")
        except Exception:
            pass
        await asyncio.sleep(0.25)          # 10 steps ≈ 2.5 sec

    await msg.edit("✅ **Successful!**")
    await asyncio.sleep(0.7)
    await msg.delete()                      # message delete
```

> Aur fast chahiye to `0.25` → `0.15`. Isse kam mat karo, Telegram edit rate-limit laga deta hai.

### 3.4 Captcha send (coloured circular buttons)

Buttons ke colors **random** assign hote hain (answer se koi connection nahi), aur teeno styles use hote hain:

```python
async def send_captcha(event, user_id, tries=0):
    png, letters, idx = generate_captcha()
    answer = letters[idx]

    order = letters[:]
    random.shuffle(order)                                       # position se guess na ho
    styles = (["primary", "success", "danger"] * 2)[:len(order)]
    random.shuffle(styles)                                      # color se guess na ho

    btns = [cb(circ(ch), f"cap:{ch}", st) for ch, st in zip(order, styles)]
    rows = [btns[i:i + 3] for i in range(0, len(btns), 3)]      # 3 per row

    await event.client.send_file(
        user_id,
        file=png,
        caption=(
            "🛡 **Human Verification**\n\n"
            "Image me jis letter pe **circle** hai,\n"
            "niche se wahi letter wale button pe tap karo 👇\n\n"
            f"⏱ Time: {CAPTCHA_TTL}s  •  ❤️ Tries: {MAX_TRIES - tries}"
        ),
        buttons=markup(*rows),
    )
    PENDING[user_id] = {"answer": answer, "expires": time.time() + CAPTCHA_TTL, "tries": tries}
```

### 3.5 `/start` handler

```python
@client.on(events.NewMessage(pattern=r"^/start"))
async def start(event):
    uid = event.sender_id
    if uid in VERIFIED:
        return await show_main_menu(event)

    await loading_bar(event)                      # 0→100 bar + success + delete
    await send_captcha(event, uid)
```

### 3.6 Button click handler

```python
@client.on(events.CallbackQuery(pattern=rb"^cap:"))
async def captcha_cb(event):
    uid = event.sender_id
    data = PENDING.get(uid)

    if not data or time.time() > data["expires"]:
        PENDING.pop(uid, None)
        await event.answer("⌛ Captcha expire ho gaya, /start dobara karo", alert=True)
        return await event.delete()

    picked = event.data.decode().split(":", 1)[1]

    if picked == data["answer"]:
        PENDING.pop(uid, None)
        VERIFIED.add(uid)                         # DB me bhi save karo
        await event.answer("✅ Verified!")
        await event.delete()
        return await show_terms(event)

    tries = data["tries"] + 1
    if tries >= MAX_TRIES:
        PENDING.pop(uid, None)
        await event.answer("❌ Too many wrong attempts. /start karo", alert=True)
        return await event.delete()

    # galat → NAYA captcha (same image reuse nahi, brute-force band)
    await event.answer(f"❌ Galat! {MAX_TRIES - tries} tries left", alert=True)
    await event.delete()
    await send_captcha(event, uid, tries=tries)
```

### 3.7 (Fallback) Bot API se styled buttons

Agar `KeyboardButtonStyle` tumhare Telethon me nahi hai, to captcha message sirf **HTTP Bot API** se bhejo. Button clicks phir bhi Telethon ke `CallbackQuery` handler me aayenge (same bot token).

```python
import json, httpx

async def send_photo_styled(token, chat_id, png, caption, rows):
    """rows = [[(text, callback_data, style), ...], ...]"""
    kb = {"inline_keyboard": [
        [{"text": t, "callback_data": d, "style": s} for t, d, s in row] for row in rows
    ]}
    async with httpx.AsyncClient(timeout=20) as http:
        r = await http.post(
            f"https://api.telegram.org/bot{token}/sendPhoto",
            data={"chat_id": chat_id, "caption": caption,
                  "parse_mode": "Markdown", "reply_markup": json.dumps(kb)},
            files={"photo": ("captcha.png", png, "image/png")},
        )
    return r.json()
```

`pip install httpx` — phir `send_captcha` me `send_file(...)` ki jagah is function ko call karo.

### 3.8 Terms & Conditions (mini app) after verify

```python
from telethon.tl.types import KeyboardButtonWebView, KeyboardButtonRow, ReplyInlineMarkup

async def show_terms(event):
    await event.client.send_message(
        event.sender_id,
        "📜 **Terms & Conditions**\n\nContinue karne se pehle padh ke accept karo 👇",
        buttons=ReplyInlineMarkup([
            KeyboardButtonRow([KeyboardButtonWebView("📖 Read Terms", "https://tg-store-t-c.vercel.app")]),
            KeyboardButtonRow([
                cb("✅ Accept", "tnc:yes", "success"),
                cb("❌ Decline", "tnc:no", "danger"),
            ]),
        ]),
    )


@client.on(events.CallbackQuery(pattern=rb"^tnc:"))
async def tnc_cb(event):
    if event.data == b"tnc:yes":
        # DB me accepted mark karo
        await event.delete()
        await show_main_menu(event)
    else:
        await event.answer("Terms accept kiye bina bot use nahi ho sakta", alert=True)
```

---

## ✨ 4. Premium UI rules (poore bot ke liye)

### Color rules

| Button type | Style |
|---|---|
| Buy / Open / Next / Main action | `primary` 🔵 |
| Accept / Confirm / Pay / Add Funds / Verify | `success` 🟢 |
| Cancel / Delete / Decline / Back-from-payment | `danger` 🔴 |
| Normal info buttons (Help, FAQ) | bina style (default) |

Ek screen pe **zyada se zyada 1–2 colors** rakho, sab kuch rangeen karoge to premium nahi lagega.

### Main menu example

```python
async def show_main_menu(event, name="User", balance=0):
    text = (
        "✦ **Fresh Tg Store** ✦\n"
        "━━━━━━━━━━━━━━\n"
        f"👤 Welcome, **{name}**\n"
        f"💰 Balance: **₹{balance}**\n"
        "━━━━━━━━━━━━━━\n"
        "Neeche se option chuno 👇"
    )
    kb = markup(
        [cb("🛒 Buy Account", "buy", "primary")],
        [cb("💰 Add Funds", "funds", "success"), cb("📦 My Orders", "orders", "primary")],
        [cb("🆘 Support", "support"), cb("ℹ️ Help", "help")],
    )
    await event.client.send_message(event.sender_id, text, buttons=kb)
```

### Confirm screen example

```python
kb = markup([cb("✅ Confirm", f"buy:ok:{item}", "success"),
             cb("❌ Cancel", "buy:no", "danger")])
```

### Baaki tips

| Cheez | Kaise |
|---|---|
| Consistent header | Har message ke top pe `✦ **Fresh Tg Store** ✦` |
| Dividers | `━━━━━━━━━━━━━━` se sections alag karo |
| Edit, don't spam | Naya msg bhejne ke bajay `event.edit()` use karo |
| Loading feel | Heavy actions pe chhota `▱▱▱` bar ya "⏳ Processing..." |
| Auto-delete | Temporary msgs (loading, captcha, alerts) delete kar do |
| Alerts | Errors ke liye `event.answer(..., alert=True)` |
| Short button text | Emoji + 1–2 words, max 2 buttons per row |
| Custom emoji icon | Optional: `KeyboardButtonStyle(bg_primary=True, icon=<custom_emoji_id>)` (Fragment username ya owner Premium chahiye) |

---

## 🛡 5. Security checklist

- [ ] Answer **sirf server memory/DB** me, button data me sirf letter (shuffle ke saath)
- [ ] Button **color answer se independent** (random)
- [ ] Captcha **single-use**, 120s expiry, 3 tries
- [ ] Galat pe **naya captcha** (same image reuse nahi)
- [ ] Fonts/colors/noise **har baar random**
- [ ] Verified users DB me save (restart ke baad dobara na puche)
- [ ] **Flood limit** (e.g. 5 captcha / 10 min per user)
- [ ] Zyada security: `length=6`, noise lines badhao, wave amplitude badhao

> ⚠️ Koi bhi captcha 100% unbreakable nahi hota (AI solvers exist). Ye setup normal OCR/script bots ko rokta hai; tries limit + expiry + naya captcha har galti pe extra layers hain.

---

## ✅ 6. Final flow

```
/start
  └─ ▰▰▰▰▰▰▰▰▰▰ 100%  →  ✅ Successful  →  (msg delete)
       └─ 🖼 Captcha image + 🔵🟢🔴 coloured circular letter buttons
            ├─ ✅ sahi → T&C (🟢 Accept / 🔴 Decline) → 💎 Premium Main Menu
            └─ ❌ galat → naya captcha (max 3 tries)
```

## 🧪 7. Quick test

1. `python -c "from telethon.tl.types import KeyboardButtonStyle"` → error nahi aana chahiye
2. Bot chalao → `/start` → bar 0→100 → "Successful" gayab → captcha aaye
3. Niche 5–6 buttons me **blue, green, red** teeno dikhne chahiye
4. Galat tap → naya captcha; 3 galat → `/start` dobara
5. Sahi tap → T&C → Accept → main menu
