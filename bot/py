import os
import re
import logging
import asyncio
import requests
from telegram import Update
from telegram.constants import ParseMode
from telegram.ext import (
    Application,
    CommandHandler,
    MessageHandler,
    ContextTypes,
    filters,
)

logging.basicConfig(
    format="%(asctime)s - %(name)s - %(levelname)s - %(message)s",
    level=logging.INFO,
)
logger = logging.getLogger(__name__)

TELEGRAM_BOT_TOKEN = os.environ.get("TELEGRAM_BOT_TOKEN")
PAGESPEED_API_KEY = os.environ.get("PAGESPEED_API_KEY")  # optional but recommended

PSI_ENDPOINT = "https://www.googleapis.com/pagespeedonline/v5/runPagespeed"

URL_REGEX = re.compile(r"^https?://", re.IGNORECASE)


def normalize_url(text: str) -> str:
    text = text.strip()
    if not URL_REGEX.match(text):
        text = "https://" + text
    return text


def score_emoji(score: float) -> str:
    if score >= 0.9:
        return "🟢"
    if score >= 0.5:
        return "🟠"
    return "🔴"


def fetch_pagespeed(url: str, strategy: str) -> dict:
    """Calls Google PageSpeed Insights API (Lighthouse) for a given URL + strategy."""
    params = {
        "url": url,
        "strategy": strategy,  # 'mobile' or 'desktop'
        "category": ["PERFORMANCE", "ACCESSIBILITY", "BEST_PRACTICES", "SEO"],
    }
    if PAGESPEED_API_KEY:
        params["key"] = PAGESPEED_API_KEY

    resp = requests.get(PSI_ENDPOINT, params=params, timeout=60)
    resp.raise_for_status()
    return resp.json()


def format_report(url: str, mobile: dict, desktop: dict) -> str:
    def extract(data: dict) -> dict:
        lh = data.get("lighthouseResult", {})
        categories = lh.get("categories", {})
        audits = lh.get("audits", {})

        def cat_score(key):
            c = categories.get(key)
            return c["score"] if c and c.get("score") is not None else None

        return {
            "performance": cat_score("performance"),
            "accessibility": cat_score("accessibility"),
            "best-practices": cat_score("best-practices"),
            "seo": cat_score("seo"),
            "lcp": audits.get("largest-contentful-paint", {}).get("displayValue", "N/A"),
            "cls": audits.get("cumulative-layout-shift", {}).get("displayValue", "N/A"),
            "tbt": audits.get("total-blocking-time", {}).get("displayValue", "N/A"),
            "fcp": audits.get("first-contentful-paint", {}).get("displayValue", "N/A"),
            "speed_index": audits.get("speed-index", {}).get("displayValue", "N/A"),
        }

    m = extract(mobile)
    d = extract(desktop)

    def block(label, r):
        def pct(s):
            return f"{round(s * 100)}" if s is not None else "N/A"

        lines = [f"*{label}*"]
        for name, key in [
            ("Performance", "performance"),
            ("Accessibility", "accessibility"),
            ("Best Practices", "best-practices"),
            ("SEO", "seo"),
        ]:
            s = r[key]
            emoji = score_emoji(s) if s is not None else "⚪"
            lines.append(f"{emoji} {name}: *{pct(s)}*/100")
        lines.append("")
        lines.append(f"⏱ LCP: `{r['lcp']}`")
        lines.append(f"📐 CLS: `{r['cls']}`")
        lines.append(f"🚧 TBT: `{r['tbt']}`")
        lines.append(f"🎨 FCP: `{r['fcp']}`")
        lines.append(f"📊 Speed Index: `{r['speed_index']}`")
        return "\n".join(lines)

    header = f"🔍 *PageSpeed Report*\n{url}\n"
    body = block("📱 Mobile", m) + "\n\n" + block("🖥 Desktop", d)
    return header + "\n" + body


async def start(update: Update, context: ContextTypes.DEFAULT_TYPE) -> None:
    await update.message.reply_text(
        "👋 Welcome to *PageSpeed Checker Bot*!\n\n"
        "Send me a URL, or use:\n"
        "`/check https://example.com`\n\n"
        "I'll analyze Performance, Accessibility, Best Practices, SEO, "
        "and Core Web Vitals for both mobile and desktop.",
        parse_mode=ParseMode.MARKDOWN,
    )


async def help_command(update: Update, context: ContextTypes.DEFAULT_TYPE) -> None:
    await update.message.reply_text(
        "*How to use:*\n"
        "`/check <url>` — full report (mobile + desktop)\n"
        "Or just paste a URL directly.\n\n"
        "Example: `/check example.com`",
        parse_mode=ParseMode.MARKDOWN,
    )


async def run_check(update: Update, raw_url: str) -> None:
    url = normalize_url(raw_url)
    status_msg = await update.message.reply_text(f"⏳ Analyzing {url} (mobile + desktop)...")

    try:
        loop = asyncio.get_event_loop()
        mobile_task = loop.run_in_executor(None, fetch_pagespeed, url, "mobile")
        desktop_task = loop.run_in_executor(None, fetch_pagespeed, url, "desktop")
        mobile, desktop = await asyncio.gather(mobile_task, desktop_task)
    except requests.exceptions.HTTPError as e:
        await status_msg.edit_text(
            f"❌ Failed to analyze `{url}`.\nError: {e}\n\n"
            "Make sure the URL is valid and publicly accessible.",
            parse_mode=ParseMode.MARKDOWN,
        )
        return
    except Exception as e:
        logger.exception("Error fetching pagespeed")
        await status_msg.edit_text(f"❌ Something went wrong: {e}")
        return

    report = format_report(url, mobile, desktop)
    await status_msg.edit_text(report, parse_mode=ParseMode.MARKDOWN, disable_web_page_preview=True)


async def check(update: Update, context: ContextTypes.DEFAULT_TYPE) -> None:
    if not context.args:
        await update.message.reply_text("Please provide a URL. Example:\n`/check example.com`", parse_mode=ParseMode.MARKDOWN)
        return
    await run_check(update, " ".join(context.args))


async def handle_message(update: Update, context: ContextTypes.DEFAULT_TYPE) -> None:
    text = update.message.text.strip()
    if URL_REGEX.match(text) or re.match(r"^[\w.-]+\.[a-z]{2,}(/.*)?$", text, re.IGNORECASE):
        await run_check(update, text)
    else:
        await update.message.reply_text(
            "Send me a valid URL, e.g. `example.com` or `/check https://example.com`",
            parse_mode=ParseMode.MARKDOWN,
        )


def main() -> None:
    if not TELEGRAM_BOT_TOKEN:
        raise RuntimeError("TELEGRAM_BOT_TOKEN environment variable is not set")

    app = Application.builder().token(TELEGRAM_BOT_TOKEN).build()

    app.add_handler(CommandHandler("start", start))
    app.add_handler(CommandHandler("help", help_command))
    app.add_handler(CommandHandler("check", check))
    app.add_handler(MessageHandler(filters.TEXT & ~filters.COMMAND, handle_message))

    logger.info("Bot starting (polling mode)...")
    app.run_polling(allowed_updates=Update.ALL_TYPES)


if __name__ == "__main__":
    main()
