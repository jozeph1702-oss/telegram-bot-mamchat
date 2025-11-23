# main.py
import os
import json
import re
import random
import datetime
import pytz
import asyncio
from collections import defaultdict
from telegram import Update, InlineKeyboardButton, InlineKeyboardMarkup, InputMediaPhoto
from telegram.ext import (
    Application,
    CommandHandler,
    MessageHandler,
    ContextTypes,
    filters,
    CallbackQueryHandler,
)

# === НАСТРОЙКИ ===
TOKEN = "8085896656:AAHR6quotHw7b8wMWkCcRhhpwdY7pgFcik8"  # ← Замени на свой токен
ALLOWED_USER_IDS = [1280234311, 259176441, 1369946707]  # ← Ты, Серафима, Анастасия
CHAT_ID = -1002778083181  # ← mPid чата

# === ID ВЕТОК (топиков) ===
TOPICS = {
    "болталка": 1,
    "общее": 1,
    "чат": 1,
    "главная": 1,
    "барахолка": 39,
    "женская барахолка": 20888,
    "еда": 77,
    "рецепты": 77,
    "детская еда": 76,
    "ребенкина еда": 76,
    "встречи": 40,
    "прогулки": 40,
    "встречи и прогулки": 40,
    "реклама": 162,
    "я женщина": 470,
    "я рекомендую": 92,
    "интересное": 34,
    "город": 34,
    "интересное в городе": 34,
    "магазины": 98,
    "интернет магазины": 98,
    "интернет-магазины": 98,
    "здоровье": 950,
    "здоровье малыша": 950,
    "полезные": 100,
    "материалы": 100,
    "полезные материалы": 100,
    "фильмы": 17148,
    "мероприятия": 93,
    "события": 93,
    "мероприятия чата": 93,
    "беременность": 471,
    "роды": 471,
    "беременность и роды": 471,
    "мемчики": 27148
}

BABIES_FILE = "babies.json"
MOMS_FILE = "moms.json"

# === БУФЕРЫ ДЛЯ АЛЬБОМОВ ===
ALBUM_TIMEOUT = 1.0  # секунды
ALBUM_BUFFER = defaultdict(list)      # для фото
VIDEO_BUFFER = defaultdict(list)      # для видео

# === ЗАГРУЗКА ДАННЫХ ===
def load_data(filename, default):
    if os.path.exists(filename):
        try:
            with open(filename, "r", encoding="utf-8") as f:
                return json.load(f)
        except Exception as e:
            print(f"Ошибка загрузки {filename}: {e}")
    return default

def save_data(data, filename):
    try:
        with open(filename, "w", encoding="utf-8") as f:
            json.dump(data, f, ensure_ascii=False, indent=2)
        print(f"✅ Сохранено в {filename}")
    except Exception as e:
        print(f"❌ Ошибка сохранения {filename}: {e}")

BABIES = load_data(BABIES_FILE, [])
MOMS = load_data(MOMS_FILE, [])

# === ВСПОМОГАТЕЛЬНЫЕ ФУНКЦИИ ===
def normalize_name(name: str) -> str:
    return ' '.join(word.capitalize() for word in name.lstrip('+-').strip().split())

async def resolve_thread_id(topic_key: str) -> int | None:
    """Преобразует строку (ID или название) → thread_id или None"""
    try:
        return int(topic_key)
    except ValueError:
        return TOPICS.get(topic_key.lower())

async def is_admin_in_chat(bot, chat_id: int, user_id: int) -> bool:
    try:
        member = await bot.get_chat_member(chat_id, user_id)
        return member.status in ['administrator', 'creator']
    except Exception as e:
        print(f"Ошибка проверки прав: {e}")
        return False

# === МЕНЮ ===
async def show_menu(update: Update, context: ContextTypes.DEFAULT_TYPE):
    keyboard = [
        [InlineKeyboardButton("💡 Совет дня", callback_data='tip')],
        [InlineKeyboardButton("📊 Статистика", callback_data='stats')],
        [InlineKeyboardButton("🎂 Ближайшие ДР", callback_data='birthdays')],
        [InlineKeyboardButton("❓ Помощь", callback_data='help')],
    ]

    chat_id = update.effective_chat.id
    user_id = update.effective_user.id

    is_admin_user = await is_admin_in_chat(context.bot, chat_id, user_id)

    if is_admin_user:
        keyboard.insert(0, [InlineKeyboardButton("📄 Список детей", callback_data='list_babies')])

    reply_markup = InlineKeyboardMarkup(keyboard)

    text = (
        "Выберите действие:\n\n"
        "Если тяжело — напишите `/support`, я обниму словом 💬💛"
    )

    if update.message:
        await update.message.reply_text(text, reply_markup=reply_markup)
    elif update.callback_query:
        await update.callback_query.edit_message_text(text=text, reply_markup=reply_markup)

# === ОБРАБОТКА КНОПОК ===
async def button_handler(update: Update, context: ContextTypes.DEFAULT_TYPE):
    query = update.callback_query
    await query.answer()

    try:
        if query.data == 'back':
            await show_menu(update, context)
            return

        if query.data == 'list_babies':
            chat_id = query.message.chat_id
            user_id = query.from_user.id

            if not await is_admin_in_chat(context.bot, chat_id, user_id):
                await query.edit_message_text("❌ Только администратор может просматривать список.")
                return

            if not BABIES:
                await query.edit_message_text("📝 Список детей пуст.")
                return

            tz = pytz.timezone('Asia/Krasnoyarsk')
            now = datetime.datetime.now(tz).date()
            text = "👶 Наши малыши:\n\n"
            for b in sorted(BABIES, key=lambda x: x["name"]):
                name = b["name"]
                birth = datetime.datetime.strptime(b["birth"], "%Y-%m-%d").date()
                m = (now.year - birth.year) * 12 + (now.month - birth.month)
                if now.day < birth.day:
                    m -= 1
                if m < 12:
                    age = f"{m} мес."
                else:
                    y, m_rem = divmod(m, 12)
                    age = f"{y} г." + (f" {m_rem} мес." if m_rem else "")
                text += f"• {name} — {birth.strftime('%d.%m.%Y')} ({age})\n"

            keyboard = [[InlineKeyboardButton("🔙 Назад", callback_data='back')]]
            reply_markup = InlineKeyboardMarkup(keyboard)
            await query.edit_message_text(text=text, reply_markup=reply_markup)

        elif query.data == 'tip':
            tips = [
                "🍼 Пейте больше воды — это важно для лактации.",
                "😴 Ложитесь спать пораньше — ваше здоровье в приоритете.",
                "🧸 Уделяйте себе 15 минут в день — вы этого достойны.",
                "💫 Вы не обязаны быть идеальной. Достаточно — быть здесь.",
                "🧺 Даже если дом в беспорядке — вы уже сделали больше, чем кажется."
            ]
            tip = random.choice(tips)
            keyboard = [[InlineKeyboardButton("🔙 Назад", callback_data='back')]]
            reply_markup = InlineKeyboardMarkup(keyboard)
            await query.edit_message_text(text=tip, reply_markup=reply_markup)

        elif query.data == 'stats':
            if not BABIES:
                text = "📊 Статистика: список пуст."
            else:
                tz = pytz.timezone('Asia/Krasnoyarsk')
                now = datetime.datetime.now(tz).date()
                groups = {"0–3 мес.": 0, "4–6 мес.": 0, "7–9 мес.": 0, "10–12 мес.": 0, "1–2 г.": 0, "2+ г.": 0}
                for b in BABIES:
                    try:
                        bd = datetime.datetime.strptime(b["birth"], "%Y-%m-%d").date()
                        m = (now.year - bd.year) * 12 + (now.month - bd.month)
                        if now.day < bd.day:
                            m -= 1
                        if m < 4: groups["0–3 мес."] += 1
                        elif m < 7: groups["4–6 мес."] += 1
                        elif m < 10: groups["7–9 мес."] += 1
                        elif m < 13: groups["10–12 мес."] += 1
                        elif m < 25: groups["1–2 г."] += 1
                        else: groups["2+ г."] += 1
                    except: pass
                text = "📊 Статистика (дети):\n\n" + "\n".join(f"• {k}: {v}" for k, v in groups.items() if v > 0)
                text += f"\n\nВсего: {len(BABIES)}"
            keyboard = [[InlineKeyboardButton("🔙 Назад", callback_data='back')]]
            reply_markup = InlineKeyboardMarkup(keyboard)
            await query.edit_message_text(text=text, reply_markup=reply_markup)

        elif query.data == 'birthdays':
            tz = pytz.timezone('Asia/Krasnoyarsk')
            now = datetime.datetime.now(tz).date()
            upcoming = []

            for b in BABIES:
                try:
                    name = b["name"]
                    bd = datetime.datetime.strptime(b["birth"], "%Y-%m-%d").date()
                    next_bday = bd.replace(year=now.year)
                    if next_bday < now:
                        next_bday = bd.replace(year=now.year + 1)
                    days = (next_bday - now).days
                    if 0 <= days <= 30:
                        upcoming.append((name, next_bday, days, 'ребёнок'))
                except:
                    continue

            for m in MOMS:
                try:
                    name = m["name"]
                    bd = datetime.datetime.strptime(m["birth"], "%Y-%m-%d").date()
                    next_bday = bd.replace(year=now.year)
                    if next_bday < now:
                        next_bday = bd.replace(year=now.year + 1)
                    days = (next_bday - now).days
                    if 0 <= days <= 30:
                        upcoming.append((name, next_bday, days, 'мама'))
                except:
                    continue

            upcoming.sort(key=lambda x: x[2])
            if upcoming:
                lines = []
                for n, d, da, t in upcoming:
                    when = "сегодня" if da == 0 else "завтра" if da == 1 else f"через {da} дн."
                    lines.append(f"• {n} — {d.strftime('%d.%m')} ({t}, {when})")
                text = "🎂 Ближайшие ДР (30 дней):\n\n" + "\n".join(lines)
            else:
                text = "Никто не празднует в ближайшие 30 дней."

            keyboard = [[InlineKeyboardButton("🔙 Назад", callback_data='back')]]
            reply_markup = InlineKeyboardMarkup(keyboard)
            await query.edit_message_text(text=text, reply_markup=reply_markup)

        elif query.data == 'help':
            help_text = (
                "🤖 Помощь:\n\n"
                "/menu — Меню\n"
                "/support — Тёплое слово в трудный день 💛\n\n"
                "📌 Дети:\n"
                "`+Имя дд.мм.гггг` — добавить (все)\n"
                "`-Имя` — удалить (админы)\n\n"
                "📌 Мамы:\n"
                "`+мама Имя дд.мм.гггг`\n\n"
                "📤 Публикация из ЛС:\n"
                "`публикуй [ветка] Текст` — текст\n"
                "`публикуй фото [ветка] Текст` — фото/альбом\n"
                "`публикуй видео [ветка] Текст` — видео/альбом\n\n"
                "Ветки: болталка, барахолка, встречи, здоровье и др."
            )
            keyboard = [[InlineKeyboardButton("🔙 Назад", callback_data='back')]]
            reply_markup = InlineKeyboardMarkup(keyboard)
            await query.edit_message_text(text=help_text, reply_markup=reply_markup)

    except Exception as e:
        if "Message is not modified" not in str(e):
            print(f"Ошибка в button_handler: {e}")
            await query.message.reply_text("⚠️ Ошибка обработки. Попробуйте снова.")

# === ДОБАВЛЕНИЕ/УДАЛЕНИЕ ===
async def handle_baby_operations(update: Update, context: ContextTypes.DEFAULT_TYPE):
    msg = update.message
    if not msg or not msg.text:
        return
    text = msg.text.strip()

    user_id = msg.from_user.id
    chat_id = msg.chat_id

    if text.startswith('-'):
        name = normalize_name(text[1:].strip())
        if not name:
            return
        if not await is_admin_in_chat(context.bot, chat_id, user_id):
            await msg.reply_text("❌ Только админы могут удалять детей.")
            return

        for i, b in enumerate(BABIES):
            if b["name"] == name:
                del BABIES[i]
                save_data(BABIES, BABIES_FILE)
                await msg.reply_text(f"✅ {name} удалён из списка детей.")
                return
        await msg.reply_text(f"❌ Ребёнок '{name}' не найден.")
        return

    if text.lower().startswith('+мама '):
        match = re.search(r'\b(\d{2})\.(\d{2})\.(\d{4})\b$', text)
        if not match:
            await msg.reply_text("❌ Не найдена дата в формате ДД.ММ.ГГГГ")
            return
        date_str = match.group(0)
        name = normalize_name(text[6:match.start()].strip())
        if not name:
            await msg.reply_text("❌ Не указано имя")
            return
        try:
            birth = datetime.datetime.strptime(date_str, "%d.%m.%Y")
            new_mom = {"name": name, "birth": birth.strftime("%Y-%m-%d")}
            if not any(m["name"] == name for m in MOMS):
                MOMS.append(new_mom)
                save_data(MOMS, MOMS_FILE)
                await msg.reply_text(f"✅ Мама {name} добавлена! Дата рождения: {date_str}")
            else:
                await msg.reply_text(f"ℹ️ Мама {name} уже есть в списке.")
        except Exception as e:
            print(f"❌ Ошибка добавления мамы: {e}")
            await msg.reply_text("❌ Ошибка при добавлении мамы. Проверьте дату.")

    elif text.startswith('+'):
        match = re.search(r'\b(\d{2})\.(\d{2})\.(\d{4})\b$', text)
        if not match:
            await msg.reply_text("❌ Не найдена дата в формате ДД.ММ.ГГГГ")
            return
        date_str = match.group(0)
        name = normalize_name(text[1:match.start()].strip())
        if not name:
            await msg.reply_text("❌ Не указано имя")
            return
        try:
            birth = datetime.datetime.strptime(date_str, "%d.%m.%Y")
            new_baby = {"name": name, "birth": birth.strftime("%Y-%m-%d")}
            if not any(b["name"] == name for b in BABIES):
                BABIES.append(new_baby)
                save_data(BABIES, BABIES_FILE)
                await msg.reply_text(f"✅ {name} добавлен! Дата рождения: {date_str}")
            else:
                await msg.reply_text(f"ℹ️ Ребёнок {name} уже есть в списке.")
        except Exception as e:
            print(f"❌ Ошибка добавления ребёнка: {e}")
            await msg.reply_text("❌ Ошибка при добавлении ребёнка. Проверьте дату.")

# === ПРИВЕТСТВИЕ НОВЫХ УЧАСТНИКОВ ===
async def greet_new_member(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if update.message is None:
        return
    for member in update.message.new_chat_members:
        if member.is_bot:
            continue
        try:
            name = member.first_name or "Добро пожаловать"
            welcome_text = (
                f"{name}, Добро пожаловать в МАМчат 🤗\n"
                "Обязательно прочитайте правила и поставьте ❤️\n\n"
                "И давайте знакомиться 🫰\n"
                "Расскажите о себе: сколько вашим деткам? Кем были до декрета?\n\n"
                "Реклама себя и обмен вещами — поощряются, но ТОЛЬКО в соответствующих ветках 🙌"
            )
            await context.bot.send_message(
                chat_id=update.effective_chat.id,
                text=welcome_text
            )
        except Exception as e:
            print(f"❌ Ошибка приветствия: {e}")

# === ПУБЛИКАЦИЯ ТЕКСТА ИЗ ЛС ===
async def receive_and_post(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if update.message is None or update.message.chat.type != 'private':
        return

    user_id = update.message.from_user.id
    if user_id not in ALLOWED_USER_IDS:
        await update.message.reply_text("❌ У вас нет прав для публикации.")
        return

    text = update.message.text.strip()

    if text.lower().startswith("публикуй "):
        parts = text[9:].strip().split(maxsplit=1)
        if len(parts) < 2:
            await update.message.reply_text(
                "❌ Укажите ID ветки или название и текст.\n"
                "Пример: `публикуй встречи Привет, девочки!`"
            )
            return

        topic_key, message_to_send = parts[0].strip(), parts[1].strip()

        if not message_to_send:
            await update.message.reply_text("❌ Нет текста для публикации.")
            return

        thread_id = await resolve_thread_id(topic_key)
        if not thread_id:
            available = ", ".join(TOPICS.keys())
            await update.message.reply_text(f"❌ Неизвестная ветка. Доступные: {available}")
            return

        try:
            if thread_id == 1:
                await context.bot.send_message(chat_id=CHAT_ID, text=message_to_send, disable_web_page_preview=True)
                await update.message.reply_text("✅ Опубликовано в *Болталку*", parse_mode='Markdown')
            else:
                await context.bot.send_message(
                    chat_id=CHAT_ID,
                    message_thread_id=thread_id,
                    text=message_to_send,
                    disable_web_page_preview=True
                )
                await update.message.reply_text(f"✅ Опубликовано в ветку `{thread_id}`", parse_mode='Markdown')
        except Exception as e:
            error_msg = "mPid ветки неверный или ветка удалена." if "message thread not found" in str(e) else str(e)
            await update.message.reply_text(f"❌ Ошибка: {error_msg}")

# === ПУБЛИКАЦИЯ ФОТО ИЗ ЛС ===
async def process_photo_album(context: ContextTypes.DEFAULT_TYPE, media_group_id: str, thread_id: int, caption: str):
    await asyncio.sleep(ALBUM_TIMEOUT)
    media_items = ALBUM_BUFFER.get(media_group_id, [])
    if not media_items:
        return

    media_items.sort(key=lambda x: x.get('file_size', 0), reverse=True)
    media = [InputMediaPhoto(media=media_items[0]['file_id'], caption=caption)]
    media.extend(InputMediaPhoto(media=item['file_id']) for item in media_items[1:])

    try:
        if thread_id == 1:
            await context.bot.send_media_group(chat_id=CHAT_ID, media=media)
        else:
            await context.bot.send_media_group(
                chat_id=CHAT_ID,
                message_thread_id=thread_id,
                media=media
            )
        await context.bot.send_message(
            chat_id=media_items[0]['user_id'],
            text=f"✅ Альбом из {len(media_items)} фото опубликован в ветку `{thread_id}`",
            parse_mode='Markdown'
        )
    except Exception as e:
        print(f"❌ Ошибка отправки фотоальбома: {e}")
        await context.bot.send_message(
            chat_id=media_items[0]['user_id'],
            text=f"❌ Ошибка при отправке альбома: {str(e)[:100]}"
        )
    finally:
        ALBUM_BUFFER.pop(media_group_id, None)

async def receive_photo(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if update.message is None or update.message.chat.type != 'private':
        return

    user_id = update.message.from_user.id
    if user_id not in ALLOWED_USER_IDS:
        return

    photos = update.message.photo
    caption = update.message.caption or ""
    media_group_id = getattr(update.message, 'media_group_id', None)

    if not photos:
        return

    if not caption.lower().startswith("публикуй фото "):
        await update.message.reply_text(
            "📌 Чтобы опубликовать фото, напишите в подпись:\n"
            "`публикуй фото [ветка] Текст`"
        )
        return

    parts = caption[13:].strip().split(maxsplit=1)
    if len(parts) < 2:
        await update.message.reply_text("❌ Укажите ID ветки и текст. Пример: `публикуй фото 39 Продам!`")
        return

    topic_key, text_to_send = parts[0].strip(), parts[1]
    thread_id = await resolve_thread_id(topic_key)
    if not thread_id:
        available = ", ".join(TOPICS.keys())
        await update.message.reply_text(f"❌ Неизвестная ветка. Доступные: {available}")
        return

    file_id = photos[-1].file_id
    file_size = photos[-1].file_size

    if media_group_id:
        ALBUM_BUFFER[media_group_id].append({
            'file_id': file_id,
            'file_size': file_size,
            'user_id': user_id
        })
        if len(ALBUM_BUFFER[media_group_id]) == 1:
            context.application.create_task(
                process_photo_album(context, media_group_id, thread_id, text_to_send)
            )
    else:
        try:
            if thread_id == 1:
                await context.bot.send_photo(chat_id=CHAT_ID, photo=file_id, caption=text_to_send)
                await update.message.reply_text("✅ Фото опубликовано в *Болталку*", parse_mode='Markdown')
            else:
                await context.bot.send_photo(
                    chat_id=CHAT_ID,
                    message_thread_id=thread_id,
                    photo=file_id,
                    caption=text_to_send
                )
                await update.message.reply_text(f"✅ Фото опубликовано в ветку `{thread_id}`", parse_mode='Markdown')
        except Exception as e:
            error_msg = "mPid ветки неверный или ветка удалена." if "message thread not found" in str(e) else str(e)
            await update.message.reply_text(f"❌ Ошибка: {error_msg}")

# === ПУБЛИКАЦИЯ ВИДЕО ИЗ ЛС ===
async def process_video_album(context: ContextTypes.DEFAULT_TYPE, media_group_id: str, thread_id: int, caption: str):
    await asyncio.sleep(ALBUM_TIMEOUT)
    video_items = VIDEO_BUFFER.get(media_group_id, [])
    if not video_items:
        return

    try:
        for i, item in enumerate(video_items):
            msg_caption = caption if i == 0 else None
            if thread_id == 1:
                await context.bot.send_video(chat_id=CHAT_ID, video=item['file_id'], caption=msg_caption)
            else:
                await context.bot.send_video(
                    chat_id=CHAT_ID,
                    message_thread_id=thread_id,
                    video=item['file_id'],
                    caption=msg_caption
                )
        await context.bot.send_message(
            chat_id=video_items[0]['user_id'],
            text=f"✅ Альбом из {len(video_items)} видео опубликован в ветку `{thread_id}`",
            parse_mode='Markdown'
        )
    except Exception as e:
        print(f"❌ Ошибка отправки видео-альбома: {e}")
        await context.bot.send_message(
            chat_id=video_items[0]['user_id'],
            text=f"❌ Ошибка при отправке видео: {str(e)[:100]}"
        )
    finally:
        VIDEO_BUFFER.pop(media_group_id, None)

async def receive_video(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if update.message is None or update.message.chat.type != 'private':
        return

    user_id = update.message.from_user.id
    if user_id not in ALLOWED_USER_IDS:
        return

    video = update.message.video
    caption = update.message.caption or ""
    media_group_id = getattr(update.message, 'media_group_id', None)

    if not video:
        return

    if not caption.lower().startswith("публикуй видео "):
        await update.message.reply_text(
            "📌 Чтобы опубликовать видео, напишите в подпись:\n"
            "`публикуй видео [ветка] Текст`"
        )
        return

    parts = caption[14:].strip().split(maxsplit=1)
    if len(parts) < 2:
        await update.message.reply_text(
            "❌ Укажите ID/название ветки и текст.\n"
            "Пример: `публикуй видео встречи Смотрите, как мы гуляли!`"
        )
        return

    topic_key, text_to_send = parts[0].strip(), parts[1]
    thread_id = await resolve_thread_id(topic_key)
    if not thread_id:
        available = ", ".join(TOPICS.keys())
        await update.message.reply_text(f"❌ Неизвестная ветка `{topic_key}`. Доступные: {available}")
        return

    file_id = video.file_id

    if media_group_id:
        VIDEO_BUFFER[media_group_id].append({
            'file_id': file_id,
            'user_id': user_id
        })
        if len(VIDEO_BUFFER[media_group_id]) == 1:
            context.application.create_task(
                process_video_album(context, media_group_id, thread_id, text_to_send)
            )
            await update.message.reply_text(
                f"⏳ Собираю видеоальбом... (Ветка `{thread_id}`)",
                parse_mode='Markdown'
            )
    else:
        try:
            if thread_id == 1:
                await context.bot.send_video(chat_id=CHAT_ID, video=file_id, caption=text_to_send)
                await update.message.reply_text("✅ Видео опубликовано в *Болталку*", parse_mode='Markdown')
            else:
                await context.bot.send_video(
                    chat_id=CHAT_ID,
                    message_thread_id=thread_id,
                    video=file_id,
                    caption=text_to_send
                )
                await update.message.reply_text(
                    f"✅ Видео опубликовано в ветку `{thread_id}`",
                    parse_mode='Markdown'
                )
        except Exception as e:
            error_msg = "mPid ветки неверный или ветка удалена." if "message thread not found" in str(e) else str(e)
            await update.message.reply_text(f"❌ Ошибка: {error_msg}")

# === КОМАНДА /support ===
async def support_command(update: Update, context: ContextTypes.DEFAULT_TYPE):
    responses = [
        "Я тебя слышу. Это тяжело — и ты всё равно продолжаешь. Это и есть сила. 💛",
        "Ты не одна. Иногда достаточно просто сказать: «Мне тяжело». Я рядом. 🤗",
        "Пусть сегодня будет хотя бы одна минута, когда ты почувствуешь: «Я — уже достаточно хороша». 💕",
        "Ты не обязана быть сильной. Достаточно — быть здесь. А ты здесь. 💖",
        "Признать усталость — не слабость. Это смелость. И ты — смелая. 🌟",
        "Дыши. Просто дыши. И знай — ты уже справляешься. Просто по-другому, чем ожидали другие.",
        "Ты — не функция. Ты — человек. И у человека бывают дни, когда просто *быть* — это уже победа."
    ]
    response = random.choice(responses)
    await update.message.reply_text(response)

# === ЕЖЕДНЕВНЫЕ ПОЗДРАВЛЕНИЯ ===
async def daily_check(context: ContextTypes.DEFAULT_TYPE):
    try:
        chat_id = context.application.bot_data.get('main_chat_id')
        if not chat_id:
            return
        tz = pytz.timezone('Asia/Krasnoyarsk')
        now = datetime.datetime.now(tz).date()
        today_str = now.strftime("%Y-%m-%d")

        # Сбрасываем greeted каждый новый день
        last_date = context.application.bot_data.get("last_check_date")
        if last_date != today_str:
            context.application.bot_data["greeted"] = set()
            context.application.bot_data["last_check_date"] = today_str

        greeted = context.application.bot_data.setdefault("greeted", set())

        # Дети: месяцы до года + 1 год
        for baby in BABIES:
            name = baby["name"]
            birth = datetime.datetime.strptime(baby["birth"], "%Y-%m-%d").date()

            months = (now.year - birth.year) * 12 + (now.month - birth.month)
            if now.day < birth.day:
                months -= 1
            years = now.year - birth.year

            key = f"baby_{name}_{months}_{years}"
            if key in greeted:
                continue

            if years == 0 and 1 <= months <= 11:
                suffix = "месяц" if months == 1 else "месяца" if 2 <= months <= 4 else "месяцев"
                msg = f"🎉 Ура! Сегодня {name} празднует {months} {suffix} жизни!"
                await context.bot.send_message(chat_id, msg)
                greeted.add(key)

            elif years == 1 and months >= 12:
                msg = f"🎂 Поздравляем! Сегодня {name} празднует 1 год!"
                await context.bot.send_message(chat_id, msg)
                greeted.add(key)

        # Мамы: только ДР
        for mom in MOMS:
            name = mom["name"]
            birth = datetime.datetime.strptime(mom["birth"], "%Y-%m-%d").date()
            if now.month == birth.month and now.day == birth.day:
                years = now.year - birth.year
                key = f"mom_{name}_{years}"
                if key not in greeted:
                    msg = f"💐 Дорогая {name}! С Днём Рождения! Пусть каждый день будет счастливым!"
                    await context.bot.send_message(chat_id, msg)
                    greeted.add(key)

    except Exception as e:
        print(f"Ошибка в daily_check: {e}")

# === ЗАПУСК ===
async def remember_chat(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if update.message:
        context.application.bot_data['main_chat_id'] = update.effective_chat.id
    await show_menu(update, context)

def main():
    print("🤖 Бот запускается...")
    app = Application.builder().token(TOKEN).build()
    app.bot_data['main_chat_id'] = None
    app.bot_data['greeted'] = set()
    app.bot_data['last_check_date'] = None

    tz = pytz.timezone('Asia/Krasnoyarsk')
    app.job_queue.run_daily(daily_check, time=datetime.time(9, 0, tzinfo=tz))

    # Хендлеры
    app.add_handler(CommandHandler("start", remember_chat))
    app.add_handler(CommandHandler("menu", remember_chat))
    app.add_handler(CommandHandler("support", support_command))
    app.add_handler(MessageHandler(filters.StatusUpdate.NEW_CHAT_MEMBERS, greet_new_member), group=3)
    app.add_handler(MessageHandler(filters.TEXT & ~filters.COMMAND, handle_baby_operations), group=1)
    app.add_handler(MessageHandler(filters.TEXT, receive_and_post), group=4)
    app.add_handler(MessageHandler(filters.PHOTO, receive_photo), group=5)
    app.add_handler(MessageHandler(filters.VIDEO, receive_video), group=6)
    app.add_handler(CallbackQueryHandler(button_handler))

    print("✅ Бот запущен. Слушаю чат...")
    app.run_polling(drop_pending_updates=True)

if __name__ == '__main__':
    main()
