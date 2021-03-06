##########################
Описание структуры плагина
##########################

========================
Как создать свой плагин:
========================

1. Импортировать файл, который отвечает за работу плагинов:

::

    from plugin_system import Plugin


2. Инициировать сам плагин с именем "plugin".

   В описании очень удобно писать 1 элемент - одна команда с описанием без ведущего префикса. Каждая строчка при выводе пользователю с помощью "!команды" начинается с префикса.

::

    plugin = Plugin("имя плагина",
                    usage=["описание", "описание", "и т.д."],
                    plugin_id="unique_id_i_invented")


============================
Как использовать базу данных
============================

Если вы намерены использовать базу данных, во-первых, вас стоит почитать что это такое и что такое SQL. В боте работа с базами реализована через peewee и peewee_async.

1. Подключить к плагину базу данных:

::

    from database import \*


2. Создать нужные вам модели (см. peewee), которые должны наследовать BaseModel:

::

    class Memo(BaseModel):
        user_id = peewee.BigIntegerField(unique=True)
        text = peewee.TextField(null=True)


3. Создать таблицы для всех ваших моделей:

::

    Memo.create_table(True)


4. Работать с бд, как написано в документации peewee_async. Например:

::

    mem.text = msg.text + divider + attachment
    await db.update(mem)


::

    mem.text = msg.text
    await db.update(mem)


=====================
Плагины с состояниями
=====================

Если вашему плагину нужно, чтобы сообщение пользователя было обработано только им - можно использовать методы ``plugin.lock``, ``plugin.unlock`` и ``plugin.is_locked``.

4.1. Сопрограмма plugin.lock(user, message), где user - это модель из БД, а message - сообщение, которое должно будет выводиться, когда другой плагин попробует занять пользователя. Вернёт (True, ""), если пользователь успешно занят плагином и (False, сообщение плагина, занявшего пользователя), если не удалось (например, потому что пользователь уже занят другим плагином). Если не удалось занять пользователя - необходимо по желанию вывести сообщение пользователю и прекратить программу.
4.2. Сопрограмма plugin.unlock(user), где user - это модель из БД, освобождает пользователя для других плагинов.
4.2. Сопрограмма plugin.is_mine(user), где user - это модель из БД, True, если пользователь занят текущим плагином.
4.3. Сопрограмма plugin.clear(user), где user - это модель из БД, выполняет unlock(user) и set_user_status(user, 0).


.. py:class:: Plugin

    .. py:decorator:: on_init()

       (Декоратор для сопрограммы)
       Сопрограмма должна принимать аргумент VkPlus. Эта сопрограмма будет выполняться при активации плаигнов в самом начале.

    .. py:decorator:: on_command(\*commands, status=None)

       (Декоратор для сопрограммы)
       Сопрограмма должна принимать аргументы Message и VkPlus. Эта сопрограмма будет выполняться каждый раз, когда пользователь будет присылать сообщение с префиксом, которое начинается с одной из строк, указанных в аргументах on_command.

       Пример (command будет выполняться, если написать боту (у которого префикс "!") "!тест" или "!test ", но не "!тесты"):

       ::

        Если аргумент "status" присутствует - сопрограмма будет выполняться только тогда, когда у пользователя в плагине есть определённый статус.
        Для каждого плагина этот статус свой за исключением случаев, когда у плагинов одинаковое название или одинаковые, также статус будет общим у плагинов с совпадающим plugin_id, который можно передать в конструкторе плагина, и если название будет совпадать с plugin_id. Поэтому названия следует выбирать уникальные.
        Менять статусы пользователей можно сопрограммами plugin.set_user_status(user, status) и plugin.get_user_status(user), где user - экземпляр User из базы данных.


       ::

        @plugin.on_command('тест', 'test')
        async def command(msg, args):
            pass

    .. py:decorator:: on_message(status=None)

       (Декоратор для сопрограммы)
       Сопрограмма должна принимать аргументы Message и VkPlus. Эта сопрограмма будет выполняться в ответ на любое необработанное сообщение пользователя.

       ::

        Если аргумент "status" присутствует - сопрограмма будет выполняться только тогда, когда у пользователя в плагине есть определённый статус.
        Для каждого плагина этот статус свой за исключением случаев, когда у плагинов одинаковое название или одинаковые, также статус будет общим у плагинов с совпадающим plugin_id, который можно передать в конструкторе плагина, и если название будет совпадать с plugin_id. Поэтому названия следует выбирать уникальные.
        Менять статусы пользователей можно сопрограммами plugin.set_user_status(user, status) и plugin.get_user_status(user), где user - экземпляр User из базы данных.


       ::

        @plugin.on_message()
        async def command(msg, args):
            pass


    .. py:decorator:: before_command(priority=0)

       (Декоратор для сопрограммы)
       Сопрограмма должна принимать аргументы Message и VkPlus. Сопрограмма будет выполняться до вызова обработчика любой команды.

       Если сопрограмма вернёт False - сообщение не будет передано другим обработчикам.

       Порядок вызова всех сопрограмм с декоратором определяется порядком инициализации плагина / сопрограммы и параметра priority. Больше значение приоритета - первее будет выполнена сопрограмма

       ::

        @plugin.before_command()
        async def before_command_processed(msg, args):
            pass


    .. py:decorator:: after_command(priority=0)

       (Декоратор для сопрограммы)
       Сопрограмма должна принимать аргументы: результат обработки функции, Message и VkPlus. Сопрограмма будет выполняться после обработки любой команды.

       Порядок вызова всех сопрограмм с декоратором определяется порядком инициализации плагина / сопрограммы и параметра priority. Больше значение приоритета - первее будет выполнена сопрограмма

       ::

        @plugin.after_command()
        async def after_command_processed(result, msg, args):
            pass


    .. py:function:: lock(user: User, message: str)

       (Сопрограмма)
       user - это модель из БД, а message - сообщение, которое должно будет выводиться, когда другой плагин попробует занять пользователя.

       Вернёт (True, ""), если пользователь успешно занят плагином и (False, сообщение плагина, занявшего пользователя), если не удалось (например, потому что пользователь уже занят другим плагином). Если не удалось занять пользователя - необходимо по желанию вывести сообщение пользователю и прекратить программу.

    .. py:function:: unlock(user: User)

       (Сопрограмма)
       user - это модель из БД,

       Освобождает пользователя для других плагинов.

    .. py:function:: is_free(user: User)

       (Сопрограмма)

       user - это модель пользователя из БД.

       Возвращает True, если пользователь свободен

    .. py:function:: is_mine(user: User)

       (Сопрограмма)
       user - это модель пользователя из БД.

       Возвращает True, если пользователь занят текущим плагином

    .. py:function:: clear(user: User)

       (Сопрограмма)
       user - это модель из БД пользователя.

       Выполняет unlock(user) и set_user_status(user, 0).