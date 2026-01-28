import logging
import asyncio
import json
from pathlib import Path
import time
from collections import deque

from aiogram import Bot, Dispatcher, Router, F
from aiogram.client.default import DefaultBotProperties
from aiogram.filters import Command
from aiogram.types import Message
from aiogram.enums import ChatType, ParseMode

# === НАСТРОЙКИ ===
# ВАЖНО: не вставляйте токен в переписки/публикуемые места.
# Если токен уже "светился" — смените его в BotFather и вставьте новый сюда.
BOT_TOKEN = "8271639521:AAGVLZhlb01rBby-Y0qgHmA5rVhxOxXb4fk"

# Telegram ID админов (числа). Можно узнать через @userinfobot
# Добавляйте сюда сколько угодно ID.
ADMIN_IDS = {1590954977, 5810417179}

# ID канала, куда пересылать анонимки (обычно начинается с -100...)
# Бот должен быть админом канала с правом "публиковать сообщения".
CHANNEL_ID = -1003459870892

logging.basicConfig(level=logging.INFO)

# Создаем бота и диспетчер
bot = Bot(
    token=BOT_TOKEN,
    default=DefaultBotProperties(parse_mode=ParseMode.HTML),
)
dp = Dispatcher()
router = Router()

BLOCKLIST_PATH = Path(__file__).with_name("blocked_users.json")


def load_blocked_users() -> set[int]:
    try:
        if not BLOCKLIST_PATH.exists():
            return set()
        data = json.loads(BLOCKLIST_PATH.read_text(encoding="utf-8") or "[]")
        if isinstance(data, list):
            return {int(x) for x in data}
        return set()
    except Exception:
        logging.exception("Failed to load %s", BLOCKLIST_PATH)
        return set()


def save_blocked_users(blocked: set[int]) -> None:
    try:
        BLOCKLIST_PATH.write_text(
            json.dumps(sorted(blocked), ensure_ascii=False, indent=2),
            encoding="utf-8",
        )
    except Exception:
        logging.exception("Failed to save %s", BLOCKLIST_PATH)


BLOCKED_USERS: set[int] = load_blocked_users()

RATE_LIMIT_MAX = 1          # сколько сообщений
RATE_LIMIT_WINDOW = 60.0    # за сколько секунд
_user_timestamps: dict[int, deque[float]] = {}
_user_last_warn: dict[int, float] = {}

FORBIDDEN_WORDS = {
    "террорист",
    "терроризм",
    "бомба",
    "взрыв",
    "взорвать",
    "взрывать",
    "взрывчатка",
    "тротил",
    "детонатор",
    "теракт",
}


def is_admin(message: Message) -> bool:
    return bool(message.from_user) and message.from_user.id in ADMIN_IDS


def _normalize_text(s: str) -> str:
    return s.lower().replace("ё", "е")


def contains_forbidden_words(text: str) -> bool:
    t = _normalize_text(text)
    return any(w in t for w in FORBIDDEN_WORDS)


def is_rate_limited(user_id: int) -> bool:
    now = time.monotonic()
    q = _user_timestamps.get(user_id)
    if q is None:
        q = deque()
        _user_timestamps[user_id] = q

    # чистим старые
    cutoff = now - RATE_LIMIT_WINDOW
    while q and q[0] < cutoff:
        q.popleft()

    if len(q) >= RATE_LIMIT_MAX:
        return True

    q.append(now)
    return False


@router.message(Command("block"))
async def cmd_block(message: Message):
    if message.chat.type != ChatType.PRIVATE or not is_admin(message):
        return

    parts = (message.text or "").split(maxsplit=1)
    if len(parts) < 2:
        await message.answer("Использование: /block <user_id>")
        return

    try:
        user_id = int(parts[1].strip())
    except ValueError:
        await message.answer("Нужно число. Пример: /block 123456789")
        return

    BLOCKED_USERS.add(user_id)
    save_blocked_users(BLOCKED_USERS)
    await message.answer(f"✅ Заблокирован user_id: <code>{user_id}</code>")


@router.message(Command("unblock"))
async def cmd_unblock(message: Message):
    if message.chat.type != ChatType.PRIVATE or not is_admin(message):
        return

    parts = (message.text or "").split(maxsplit=1)
    if len(parts) < 2:
        await message.answer("Использование: /unblock <user_id>")
        return

    try:
        user_id = int(parts[1].strip())
    except ValueError:
        await message.answer("Нужно число. Пример: /unblock 123456789")
        return

    if user_id in BLOCKED_USERS:
        BLOCKED_USERS.remove(user_id)
        save_blocked_users(BLOCKED_USERS)
        await message.answer(f"✅ Разблокирован user_id: <code>{user_id}</code>")
    else:
        await message.answer("Этого user_id нет в блокировке.")


@router.message(Command("blocked"))
async def cmd_blocked(message: Message):
    if message.chat.type != ChatType.PRIVATE or not is_admin(message):
        return

    if not BLOCKED_USERS:
        await message.answer("Список блокировок пуст.")
        return

    ids = "\n".join(f"- <code>{uid}</code>" for uid in sorted(BLOCKED_USERS))
    await message.answer(f"🚫 Заблокированные user_id:\n{ids}")


@router.message(Command("start"))
async def cmd_start(message: Message):
    await message.answer(
        "Привет!\n\n"
        "Отправь мне сообщение — я анонимно перешлю его в канал.\n"
        "Поддерживается: текст, музыка, видео, голосовые, кружки."
    )


@router.message(F.chat.type == ChatType.PRIVATE)
async def handle_private(message: Message):
    # Игнорируем сообщения от заблокированных пользователей
    if message.from_user and message.from_user.id in BLOCKED_USERS:
        return

    # Не пересылаем команды (/start и т.п.)
    if message.text and message.text.startswith("/"):
        return

    # Антиспам: ограничение частоты сообщений от одного пользователя
    if message.from_user:
        uid = message.from_user.id
        if is_rate_limited(uid):
            now = time.monotonic()
            # не спамим ответами — предупреждаем максимум раз в 5 секунд
            if now - _user_last_warn.get(uid, 0.0) >= 5.0:
                _user_last_warn[uid] = now
                await message.answer("⏳ Слишком часто. Подожди немного и попробуй снова.")
            return

    # Фильтр запрещённых слов (проверяем текст или подпись к медиа)
    content_text = (message.text or message.caption or "").strip()
    if content_text and contains_forbidden_words(content_text):
        await message.answer("🚫 Сообщение не отправлено: содержит запрещённые слова.")
        return

    # 1) Копируем в канал АНОНИМНО (без "forwarded from")
    try:
        await bot.copy_message(
            chat_id=CHANNEL_ID,
            from_chat_id=message.chat.id,
            message_id=message.message_id,
        )
    except Exception:
        logging.exception("Failed to copy message to channel")
        await message.answer(
            "Не получилось отправить этот тип контента в канал. Попробуйте другой файл/формат."
        )
        return

    # 2) Админу(ам): инфо + копия контента
    user = message.from_user
    admin_text = (
        "🆕 <b>Анонимное сообщение</b>\n\n"
        f"<b>От:</b> {user.full_name}\n"
        f"<b>Username:</b> @{user.username if user.username else 'нет'}\n"
        f"<b>User ID:</b> <code>{user.id}</code>\n"
        f"<b>Тип:</b> <code>{message.content_type}</code>"
    )
    for admin_id in ADMIN_IDS:
        await bot.send_message(chat_id=admin_id, text=admin_text)
        try:
            await bot.copy_message(
                chat_id=admin_id,
                from_chat_id=message.chat.id,
                message_id=message.message_id,
            )
        except Exception:
            # Если вдруг не копируется админу — хотя бы не падаем
            logging.exception("Failed to copy message to admin %s", admin_id)

    # 3) Подтверждаем пользователю
    await message.answer("✅ Сообщение отправлено анонимно в канал.")


# Регистрируем роутер
dp.include_router(router)


async def main():
    # Сбросить накопившиеся апдейты (аналог старого skip_updates=True)
    await bot.delete_webhook(drop_pending_updates=True)
    await dp.start_polling(bot)


if __name__ == "__main__":
    asyncio.run(main())
