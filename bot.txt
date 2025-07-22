import asyncio
from aiogram import Bot, Dispatcher, types, F
from aiogram.types import FSInputFile, ReplyKeyboardMarkup, KeyboardButton
from aiogram.fsm.context import FSMContext
from aiogram.fsm.storage.memory import MemoryStorage
from aiogram.fsm.state import StatesGroup, State

# === CONFIGURATION ===
BOT_TOKEN = '7911451455:AAF9EBoCOinRkVrvA0qqvgpE1-i2ds0PNo8'
ADMIN_ID = 7657798402
MANAGER_USERNAME = 'DenisHRBP_kadry_s_dushoj'  # без @

# === FSM STATE ===
class Form(StatesGroup):
    waiting_for_name = State()
    waiting_for_phone = State()
    waiting_for_request = State()

class GreetForm(StatesGroup):
    name = State()
    phone = State()

# === KEYBOARDS ===
def main_menu():
    kb = ReplyKeyboardMarkup(keyboard=[
        [KeyboardButton(text="📝 Получить гайд")],
        [KeyboardButton(text="📩 Оставить заявку")],
        [KeyboardButton(text="💬 Написать менеджеру")],
        [KeyboardButton(text="🌐 О компании")],
    ], resize_keyboard=True)
    return kb

back_kb = ReplyKeyboardMarkup(keyboard=[
    [KeyboardButton(text="🔙 Назад в меню")]
], resize_keyboard=True)

# === MAIN LOGIC ===
bot = Bot(BOT_TOKEN)
dp = Dispatcher(storage=MemoryStorage())

@dp.message(F.text == "/start")
@dp.message(F.text == "🔙 Назад в меню")
async def start(message: types.Message, state: FSMContext):
    await state.clear()
    await message.answer("👋 Добро пожаловать! Выберите действие:", reply_markup=main_menu())

# === ГАЙД ===
@dp.message(F.text == "📝 Получить гайд")
async def get_guide_step1(message: types.Message, state: FSMContext):
    await message.answer("Как вас зовут?", reply_markup=back_kb)
    await state.set_state(GreetForm.name)

@dp.message(GreetForm.name)
async def get_guide_step2(message: types.Message, state: FSMContext):
    await state.update_data(name=message.text)
    await message.answer("Укажите ваш номер телефона:")
    await state.set_state(GreetForm.phone)

@dp.message(GreetForm.phone)
async def send_guide(message: types.Message, state: FSMContext):
    data = await state.get_data()
    name = data["name"]
    phone = message.text

    # Отправляем PDF
    pdf = FSInputFile("guide.pdf")
    await message.answer_document(pdf, caption="🎉 Поздравляем! Вот ваш гайд:", reply_markup=back_kb)

    # Отправляем лид админу
    text = f"📥 Новый лид (гайд)\nИмя: {name}\nТелефон: {phone}"
    await bot.send_message(ADMIN_ID, text)
    await state.clear()

# === ЗАЯВКА ===
@dp.message(F.text == "📩 Оставить заявку")
async def request_step1(message: types.Message, state: FSMContext):
    await message.answer("Как вас зовут?", reply_markup=back_kb)
    await state.set_state(Form.waiting_for_name)

@dp.message(Form.waiting_for_name)
async def request_step2(message: types.Message, state: FSMContext):
    await state.update_data(name=message.text)
    await message.answer("Укажите номер телефона:")
    await state.set_state(Form.waiting_for_phone)

@dp.message(Form.waiting_for_phone)
async def request_step3(message: types.Message, state: FSMContext):
    await state.update_data(phone=message.text)
    await message.answer("Кого вы ищете? (например: 'менеджер по продажам')")
    await state.set_state(Form.waiting_for_request)

@dp.message(Form.waiting_for_request)
async def request_finish(message: types.Message, state: FSMContext):
    data = await state.get_data()
    req = message.text

    text = (
        f"📩 Новая заявка:\n"
        f"Имя: {data['name']}\n"
        f"Телефон: {data['phone']}\n"
        f"Запрос: {req}"
    )

    await bot.send_message(ADMIN_ID, text)
    await message.answer(
        f"✅ Ваша заявка отправлена!\n"
        f"Если хотите связаться незамедлительно — пишите сюда @{MANAGER_USERNAME}",
        reply_markup=back_kb
    )
    await state.clear()

# === МЕНЕДЖЕР ===
@dp.message(F.text == "💬 Написать менеджеру")
async def contact_manager(message: types.Message):
    await message.answer(f"📲 Написать менеджеру: https://t.me/{MANAGER_USERNAME}", reply_markup=back_kb)

# === О КОМПАНИИ ===
@dp.message(F.text == "🌐 О компании")
async def about_company(message: types.Message):
    await message.answer(
        "HR-TARGET — команда экспертов по подбору персонала.\n"
        "⚙️ 18 лет опыта\n"
        "🎯 Подбор первых кандидатов — от 3 дней\n"
        "🛡️ Гарантия замены — до 180 дней\n"
        "🔗 Подробнее: https://hr-target.ru",
        reply_markup=back_kb
    )

# === RUN ===
async def main():
    await dp.start_polling(bot)

if __name__ == "__main__":
    asyncio.run(main())
