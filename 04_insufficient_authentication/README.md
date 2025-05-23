# Уязвимость 4: Недостаточная аутентификация (Insufficient Authentication)

Это задание демонстрирует уязвимость, при которой критически важные действия или доступ к данным возможны *до* того, как пользователь подтвердил свою личность (прошел аутентификацию).

## Файлы

*   `vulnerable_app.py`: Уязвимое приложение, позволяющее просматривать профиль любого пользователя по ID без входа в систему.
*   `fixed_app.py`: Исправленное приложение, которое требует входа для просмотра профиля и позволяет пользователю видеть только свой профиль. Использует простой файл `.user_session.tmp` для имитации сессии.
*   `.user_session.tmp`: Файл для хранения ID вошедшего пользователя (создается и управляется `fixed_app.py`).

## Описание уязвимости

В `vulnerable_app.py` функция `get_user_profile(user_id)` запрашивает и отображает информацию о пользователе, включая его "секретные данные". Проблема в том, что для вызова этой функции не требуется предварительная аутентификация. Любой, кто знает (или может угадать) ID пользователя, может выполнить команду `profile <user_id>` и увидеть его данные.

```python
# vulnerable_app.py
def get_user_profile(user_id):
    # УЯЗВИМОСТЬ: Нет проверки, вошел ли кто-то в систему!
    user = USERS.get(user_id)
    if user:
        print(f"  Секретная информация: {user['secret']}") # Показ данных без логина
    # ...

# ... в основном блоке ...
elif action == "profile":
    user_id = sys.argv[2]
    # Уязвимый вызов - профиль показывается без проверки логина
    get_user_profile(user_id)
```

## Эксплуатация

1.  **Запустите уязвимое приложение, чтобы посмотреть профиль пользователя 'admin' (ID=1) без входа:**
    ```bash
    python vulnerable_app.py profile 1
    ```
    Вы увидите секретные данные администратора, хотя вы не вводили его пароль.

2.  **Посмотрите профиль пользователя 'guest' (ID=2):**
    ```bash
    python vulnerable_app.py profile 2
    ```
    Вы увидите секретные данные гостя.

## Исправление

Уязвимость устранена в `fixed_app.py` несколькими мерами:

1.  **Введено состояние "вошедшего пользователя":** Глобальная переменная `current_logged_in_user_id` хранит ID пользователя, который успешно вошел. Эта переменная сохраняется в файле `.user_session.tmp` и загружается при запуске, имитируя сессию.
2.  **Функция `get_my_profile()` проверяет аутентификацию:** Перед показом профиля она проверяет, установлено ли значение `current_logged_in_user_id`. Если нет, выводится ошибка.
3.  **Пользователь видит только свой профиль:** Функция `get_my_profile()` не принимает `user_id` как аргумент, а использует `current_logged_in_user_id` для получения данных текущего пользователя.

```python
# fixed_app.py
current_logged_in_user_id = None
session_file = ".user_session.tmp"

# ... load_session() и save_session() ...

def get_my_profile():
    load_session() # Загружаем ID из файла сессии
    # ИСПРАВЛЕНО: Требуется аутентификация
    if not current_logged_in_user_id:
        print("Ошибка: Вы должны войти в систему...")
        return

    # ИСПРАВЛЕНО: Пользователь видит только свой профиль
    user = USERS.get(current_logged_in_user_id)
    # ... показать профиль ...
```

## Проверка исправления

1.  **(Необязательно) Удалите старый файл сессии, если он есть:**
    ```bash
    # Linux/macOS
    rm -f .user_session.tmp
    # Windows
    del .user_session.tmp
    ```
2.  **Попытайтесь посмотреть профиль без входа:**
    ```bash
    python fixed_app.py profile
    ```
    Вы увидите сообщение об ошибке, требующее входа в систему.

3.  **Войдите в систему как admin:**
    ```bash
    python fixed_app.py login 1 admin_pass
    ```
    Вы увидите сообщение об успешном входе. Файл `.user_session.tmp` будет создан/обновлен.

4.  **Теперь посмотрите профиль:**
    ```bash
    python fixed_app.py profile
    ```
    Вы увидите профиль администратора.

5.  **Выйдите из системы:**
    ```bash
    python fixed_app.py logout
    ```
    Файл `.user_session.tmp` будет очищен.

6.  **Попробуйте снова посмотреть профиль:**
    ```bash
    python fixed_app.py profile
    ```
    Вы снова увидите сообщение об ошибке, так как вы вышли из системы. Исправление работает. 