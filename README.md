# Telegram_bot
Python Developer Test Pass
import telebot

import nest_asyncio
nest_asyncio.apply()

import aiosqlite
import asyncio
import logging
from aiogram import Bot, Dispatcher, types
from aiogram.filters.command import Command
from aiogram.filters.command import Command
from aiogram import types
from aiogram.utils.keyboard import InlineKeyboardBuilder, ReplyKeyboardBuilder
from aiogram import F
import pandas as pd

logging.basicConfig(level=logging.INFO)

API_TOKEN = '7197930884:AAEXP98W6cnORBPmBf6_NHCvn0EzQ_6n-Sc'

bot = Bot(token=API_TOKEN)
dp = Dispatcher()
DB_NAME = 'quiz_bot.db'

quiz_data = [
    {
        'question': 'Что такое Python?',
        'options': ['Язык программирования', 'Тип данных', 'Музыкальный инструмент', 'Змея на английском'],
        'correct_option': 0
    },
    {
        'question': 'Какой тип данных используется для хранения целых чисел?',
        'options': ['int', 'float', 'str', 'natural'],
        'correct_option': 0

    },
    {   'question': 'Где правильно создана переменная?',
        'options': ['int num = 2', 'Нет подходящего варианта', 'var num = 2', 'num = float(2)'],
        'correct_option': 3
    },
    {   'question': 'Какие ошибки допущены в коде ниже?'
                    '''def factorial(n):
                           if n == 0:
                             return 1
                           else:
                             return n * factorial(n - 1)
                       print(factorial(5))''',
        'options': ['Функция не может вызывать сама себя',
                    'Функция всегда будет возвращать 1',
                    'Необходимо указать тип возвращаемого значения',
                    'В коде нет никаких ошибок'],
        'correct_option': 3
    },
    {   'question': 'Сколько библиотек можно импортировать в один проект?',
        'options': ['Не более 3', 'Не более 10', 'Неограниченное количество', 'Не более 5'],
        'correct_option': 2
    },
    {   'question': 'Какая функция выводит что-либо в консоль?',
        'options': ['write()', 'print()', 'out()', 'log()'],
        'correct_option': 1
    },
    {   'question': '''Что будет результатом этого кода?
                       x = 23
                       num = 0 if x > 10 else 11
                       print(num)''',
        'options': ['23', '0', '10', '11'],
        'correct_option': 1
    },
    {   'question': '''Что покажет этот код?
                       for i in range(5):
                           if i % 2 == 0:
                               continue
                       print(i)''',
        'options': ['Ошибку, так как i не присвоена',
                    'Числа: 0, 2 и 4', 'Ошибку из-за неверного вывода',
                    'Числа: 1 и 3'],
        'correct_option': 3
    },
    {   'question': 'Как получить данные от пользователя?',
        'options': ['Использовать метод get()', 'Использовать метод input()',
                    'Использовать метод read()', 'Использовать метод cin()'],
        'correct_option': 1
    },
    {   'question': 'Какая библиотека отвечает за время?',
        'options': ['localtime', 'Time', 'time', 'clock'],
        'correct_option': 2
    },
    {   'question': '''Что будет показано в результате?
                       name = "John"
                       print('Hi, %s' % name)''',
        'options': ['"Hi, name"', '"Hi, John"', 'Ошибка', '"Hi, "'],
        'correct_option': 1
    },
    {   'question': 'Для чего в Python используется встроенная функция enumerate()',
        'options': ['Для определения количества элементов последовательности.',
                    'Для сортировки элементов по значениям id.',
                    'Для одновременного итерирования по самим элементам и их индексам.'
                    'natural'],
        'correct_option': 2
    },
    {
        'question': ' Необходимо собрать и вывести все уникальные слова из строки рекламного текста. Какой из перечисленных типов данных Python подходит лучше всего?',
        'options': ['кортеж (tuple)', 'список (list)', 'словарь (dict)', 'множество (set)'],
        'correct_option': 3
    }]


# Хэндлер на команду /start
@dp.message(Command("hello"))
async def cmd_start(message: types.Message):
    await message.answer("Привет! Я твой новый бот.")


async def update_quiz_index(user_id, index):
    # Создаем соединение с базой данных (если она не существует, она будет создана)
    async with aiosqlite.connect(DB_NAME) as db:
        # Вставляем новую запись или заменяем ее, если с данным user_id уже существует
        await db.execute('INSERT OR REPLACE INTO quiz_state (user_id, question_index) VALUES (?, ?)', (user_id, index))
        # Сохраняем изменения
        await db.commit()


async def get_quiz_index(user_id):
    # Подключаемся к базе данных
    async with aiosqlite.connect(DB_NAME) as db:
        # Получаем запись для заданного пользователя
        async with db.execute('SELECT question_index FROM quiz_state WHERE user_id = (?)', (user_id,)) as cursor:
            # Возвращаем результат
            results = await cursor.fetchone()
            if results is not None:
                return results[0]
            else:
                return 0

@dp.message(Command("start"))
async def cmd_start(message: types.Message):
    # Создаем сборщика клавиатур типа Reply
    builder = ReplyKeyboardBuilder()
    # Добавляем в сборщик одну кнопку
    builder.add(types.KeyboardButton(text="Начать игру"))
    # Прикрепляем кнопки к сообщению
    await message.answer("Добро пожаловать в квиз!", reply_markup=builder.as_markup(resize_keyboard=True))


@dp.message(F.text=="Начать игру")
@dp.message(Command("quiz"))
async def cmd_quiz(message: types.Message):
    # Отправляем новое сообщение без кнопок
    await message.answer(f"Давайте начнем квиз!")
    # Запускаем новый квиз
    await new_quiz(message)

async def new_quiz(message):
    # получаем id пользователя, отправившего сообщение
    user_id = message.from_user.id
    # сбрасываем значение текущего индекса вопроса квиза в 0
    current_question_index = 0
    await update_quiz_index(user_id, current_question_index)

    # запрашиваем новый вопрос для квиза
    await get_question(message, user_id)

async def get_question(message, user_id):

    # Запрашиваем из базы текущий индекс для вопроса
    current_question_index = await get_quiz_index(user_id)
    # Получаем индекс правильного ответа для текущего вопроса
    correct_index = quiz_data[current_question_index]['correct_option']
    # Получаем список вариантов ответа для текущего вопроса
    opts = quiz_data[current_question_index]['options']

    # Функция генерации кнопок для текущего вопроса квиза
    # В качестве аргументов передаем варианты ответов и значение правильного ответа (не индекс!)
    kb = generate_options_keyboard(opts, opts[correct_index])
    # Отправляем в чат сообщение с вопросом, прикрепляем сгенерированные кнопки
    await message.answer(f"{quiz_data[current_question_index]['question']}", reply_markup=kb)


def generate_options_keyboard(answer_options, right_answer):

  # Создаем сборщика клавиатур типа Inline
    builder = InlineKeyboardBuilder()

    # В цикле создаем 4 Inline кнопки, а точнее Callback-кнопки
    for option in answer_options:
        builder.add(types.InlineKeyboardButton(
            # Текст на кнопках соответствует вариантам ответов
            text=option,
            # Присваиваем данные для колбэк запроса.
            # Если ответ верный сформируется колбэк-запрос с данными 'right_answer'
            # Если ответ неверный сформируется колбэк-запрос с данными 'wrong_answer'
            callback_data="right_answer" if option == right_answer else "wrong_answer")
        )

    # Выводим по одной кнопке в столбик
    builder.adjust(1)
    return builder.as_markup()


count = 0
@dp.callback_query(F.data == "right_answer")
async def right_answer(callback: types.CallbackQuery):

    await callback.bot.edit_message_reply_markup(
        chat_id=callback.from_user.id,
        message_id=callback.message.message_id,
        reply_markup=None
    )

    await callback.message.answer("Верно!")
    current_question_index = await get_quiz_index(callback.from_user.id)

    # Обновление номера текущего вопроса в базе данных
    current_question_index += 1
    global count
    count += 1

    await update_quiz_index(callback.from_user.id, current_question_index)


    if current_question_index < len(quiz_data):
        await get_question(callback.message, callback.from_user.id)
    else:
        await callback.message.answer(f"Это был последний вопрос. Квиз завершен!\n"
                                      f"Правильных ответов {count} из 12")


@dp.callback_query(F.data == "wrong_answer")
async def wrong_answer(callback: types.CallbackQuery):
    await callback.bot.edit_message_reply_markup(
        chat_id=callback.from_user.id,
        message_id=callback.message.message_id,
        reply_markup=None
    )

    # Получение текущего вопроса из словаря состояний пользователя
    current_question_index = await get_quiz_index(callback.from_user.id)
    correct_option = quiz_data[current_question_index]['correct_option']

    await callback.message.answer(f"Неправильно. Правильный ответ: {quiz_data[current_question_index]['options'][correct_option]}")

    # Обновление номера текущего вопроса в базе данных
    current_question_index += 1
    await update_quiz_index(callback.from_user.id, current_question_index)


    if current_question_index < len(quiz_data):
        await get_question(callback.message, callback.from_user.id)
    else:
        await callback.message.answer(f"Это был последний вопрос. Квиз завершен!\n"
                                      f"Правильных ответов {count} из 12")

# Запуск процесса поллинга новых апдейтов
async def main():
    async def create_table():
        # Создаем соединение с базой данных (если она не существует, то она будет создана)
        async with aiosqlite.connect('quiz_bot.db') as db:
            # Выполняем SQL-запрос к базе данных
            await db.execute(
                '''CREATE TABLE IF NOT EXISTS quiz_state (user_id INTEGER PRIMARY KEY, question_index INTEGER)''')
            # Сохраняем изменения
            await db.commit()

    # Запускаем создание таблицы базы данных
    await create_table()

    await dp.start_polling(bot)

if __name__ == "__main__":
    asyncio.run(main())
