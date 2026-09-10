import os
import aiohttp
from telegram import Update
from telegram.ext import Application, CommandHandler, MessageHandler, ContextTypes, filters

TOKEN = os.getenv("BOT_TOKEN")

if not TOKEN:
    raise RuntimeError("BOT_TOKEN ёфт нашуд!")


async def find_anime(image_bytes):
    url = "https://api.trace.moe/search"

    form = aiohttp.FormData()
    form.add_field(
        "image",
        image_bytes,
        filename="image.jpg",
        content_type="image/jpeg"
    )

    async with aiohttp.ClientSession() as session:
        async with session.post(url, data=form) as response:
            if response.status != 200:
                return None

            data = await response.json()

    results = data.get("result", [])

    if not results:
        return None

    best = results[0]

    return {
        "anime": best.get("filename", "Номаълум"),
        "episode": best.get("episode"),
        "similarity": best.get("similarity", 0),
    }


async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    await update.message.reply_text(
        "👋 Салом!\n\n"
        "🖼️ Скриншоти анимеро фирист, ман кӯшиш мекунам онро муайян кунам."
    )


async def handle_photo(update: Update, context: ContextTypes.DEFAULT_TYPE):
    await update.message.reply_text("🔎 Анимеро ҷустуҷӯ карда истодаам...")

    photo = update.message.photo[-1]
    file = await photo.get_file()
    data = await file.download_as_bytearray()

    result = await find_anime(bytes(data))

    if not result:
        await update.message.reply_text(
            "❌ Анимеро муайян карда натавонистам."
        )
        return

    text = f"🎌 Аниме: {result['anime']}\n"

    if result["episode"]:
        text += f"📺 Серия: {result['episode']}\n"

    text += f"🔎 Эътимод: {result['similarity'] * 100:.1f}%"

    await update.message.reply_text(text)


def main():
    app = Application.builder().token(TOKEN).build()

    app.add_handler(CommandHandler("start", start))
    app.add_handler(MessageHandler(filters.PHOTO, handle_photo))

    print("🤖 Bot started!")
    app.run_polling()


if __name__ == "__main__":
    main()
