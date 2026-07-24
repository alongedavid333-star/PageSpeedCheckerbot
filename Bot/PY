import os
import base64
import logging
from io import BytesIO

from telegram import Update, InlineKeyboardButton, InlineKeyboardMarkup
from telegram.constants import ParseMode
from telegram.ext import (
    Application,
    CommandHandler,
    MessageHandler,
    CallbackQueryHandler,
    ContextTypes,
    filters,
)
from anthropic import Anthropic

logging.basicConfig(
    format="%(asctime)s - %(name)s - %(levelname)s - %(message)s",
    level=logging.INFO,
)
logger = logging.getLogger(__name__)

TELEGRAM_BOT_TOKEN = os.environ.get("TELEGRAM_BOT_TOKEN")
ANTHROPIC_API_KEY = os.environ.get("ANTHROPIC_API_KEY")

# Cheaper/faster: "claude-haiku-4-5-20251001". Higher quality: "claude-sonnet-5".
CLAUDE_MODEL = os.environ.get("CLAUDE_MODEL", "claude-sonnet-5")

client = Anthropic(api_key=ANTHROPIC_API_KEY)

TONES = ["Funny", "Aesthetic", "Professional", "Romantic", "Motivational"]

# In-memory per-user pending job: either {"type": "photo", "bytes": ..., "media_type": ...}
# or {"type": "topic", "text": ...}. Cleared once a tone is picked and captions are generated.
PENDING: dict[int, dict] = {}


def tone_keyboard() -> InlineKeyboardMarkup:
    row1 = [InlineKeyboardButton(t, callback_data=f"tone:{t}") for t in TONES[:3]]
    row2 = [InlineKeyboardButton(t, callback_data=f"tone:{t}") for t in TONES[3:]]
    return InlineKeyboardMarkup([row1, row2])


def build_prompt(tone: str, topic: str | None) -> str:
    base = (
        f"Write 3 different social media captions in a {tone.lower()} tone"
    )
    if topic:
        base += f" about: {topic}."
    else:
        base += " for the attached photo."
    base += (
        " Each caption should be 1-3 sentences, include 3-6 relevant hashtags at the end, "
        "and 1-2 fitting emojis worked naturally into the text. "
        "Number them 1, 2, 3. Do not add any extra commentary before or after the captions."
    )
    return base


async def start(update: Update, context: ContextTypes.DEFAULT_TYPE) -> None:
    await update.message.reply_text(
        "✨ Welcome to *CaptionGeniusBot*!\n\n"
        "Send me a *photo* or a *topic* (just type it) and I'll write you "
        "scroll-stopping captions with hashtags.\n\n"
        "You can also use `/caption <topic>` directly.",
        parse_mode=ParseMode.MARKDOWN,
    )


async def help_command(update: Update, context: ContextTypes.DEFAULT_TYPE) -> None:
    await update.message.reply_text(
        "*How to use:*\n"
        "📸 Send a photo → pick a tone → get 3 captions\n"
        "✍️ Type a topic, or `/caption <topic>` → pick a tone → get 3 captions\n\n"
        "Example: `/caption a rainy coffee shop evening`",
        parse_mode=ParseMode.MARKDOWN,
    )


async def caption_command(update: Update, context: ContextTypes.DEFAULT_TYPE) -> None:
    if not context.args:
        await update.message.reply_text(
            "Give me a topic. Example:\n`/caption sunset hike with friends`",
            parse_mode=ParseMode.MARKDOWN,
        )
        return
    topic = " ".join(context.args)
    PENDING[update.effective_user.id] = {"type": "topic", "text": topic}
    await update.message.reply_text("Pick a tone:", reply_markup=tone_keyboard())


async def handle_text(update: Update, context: ContextTypes.DEFAULT_TYPE) -> None:
    topic = update.message.text.strip()
    if not topic:
        return
    PENDING[update.effective_user.id] = {"type": "topic", "text": topic}
    await update.message.reply_text("Pick a tone:", reply_markup=tone_keyboard())


async def handle_photo(update: Update, context: ContextTypes.DEFAULT_TYPE) -> None:
    photo = update.message.photo[-1]  # highest resolution
    file = await photo.get_file()
    buf = BytesIO()
    await file.download_to_memory(out=buf)
    image_bytes = buf.getvalue()

    PENDING[update.effective_user.id] = {
        "type": "photo",
        "bytes": image_bytes,
        "media_type": "image/jpeg",
    }
    await update.message.reply_text("Nice photo! Pick a tone:", reply_markup=tone_keyboard())


async def handle_tone_choice(update: Update, context: ContextTypes.DEFAULT_TYPE) -> None:
    query = update.callback_query
    await query.answer()
    user_id = query.from_user.id
    job = PENDING.get(user_id)

    if not job:
        await query.edit_message_text("This request expired — send a new photo or topic.")
        return

    tone = query.data.split(":", 1)[1]
    await query.edit_message_text(f"✍️ Writing {tone.lower()} captions...")

    try:
        if job["type"] == "photo":
            b64_image = base64.b64encode(job["bytes"]).decode("utf-8")
            prompt = build_prompt(tone, topic=None)
            message = client.messages.create(
                model=CLAUDE_MODEL,
                max_tokens=500,
                messages=[
                    {
                        "role": "user",
                        "content": [
                            {
                                "type": "image",
                                "source": {
                                    "type": "base64",
                                    "media_type": job["media_type"],
                                    "data": b64_image,
                                },
                            },
                            {"type": "text", "text": prompt},
                        ],
                    }
                ],
            )
        else:
            prompt = build_prompt(tone, topic=job["text"])
            message = client.messages.create(
                model=CLAUDE_MODEL,
                max_tokens=500,
                messages=[{"role": "user", "content": prompt}],
            )

        result_text = "".join(
            block.text for block in message.content if block.type == "text"
        ).strip()

        await query.edit_message_text(
            f"✨ *{tone} captions:*\n\n{result_text}",
            parse_mode=ParseMode.MARKDOWN,
        )
    except Exception as e:
        logger.exception("Error generating caption")
        await query.edit_message_text(f"❌ Something went wrong: {e}")
    finally:
        PENDING.pop(user_id, None)


def main() -> None:
    if not TELEGRAM_BOT_TOKEN:
        raise RuntimeError("TELEGRAM_BOT_TOKEN environment variable is not set")
    if not ANTHROPIC_API_KEY:
        raise RuntimeError("ANTHROPIC_API_KEY environment variable is not set")

    app = Application.builder().token(TELEGRAM_BOT_TOKEN).build()

    app.add_handler(CommandHandler("start", start))
    app.add_handler(CommandHandler("help", help_command))
    app.add_handler(CommandHandler("caption", caption_command))
    app.add_handler(MessageHandler(filters.PHOTO, handle_photo))
    app.add_handler(CallbackQueryHandler(handle_tone_choice, pattern=r"^tone:"))
    app.add_handler(MessageHandler(filters.TEXT & ~filters.COMMAND, handle_text))

    logger.info("CaptionGeniusBot starting (polling mode)...")
    app.run_polling(allowed_updates=Update.ALL_TYPES)


if __name__ == "__main__":
    main()
