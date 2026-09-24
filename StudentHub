from flask import Flask, render_template, request, redirect, session
from werkzeug.security import generate_password_hash, check_password_hash
import sqlite3
import requests
import os
from dotenv import load_dotenv

load_dotenv()

app = Flask(__name__)
app.secret_key = "studenthub_super_secret_key"

DIFY_API_URL = os.getenv("DIFY_API_URL", "").strip().rstrip("/")
DIFY_API_KEY = os.getenv("DIFY_API_KEY", "").strip()


# =========================================================
# DATABASE
# =========================================================

def get_db():
    conn = sqlite3.connect("database.db")
    conn.row_factory = sqlite3.Row
    return conn


def init_db():

    conn = get_db()

    conn.execute("""
        CREATE TABLE IF NOT EXISTS users (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            username TEXT UNIQUE NOT NULL,
            password TEXT NOT NULL
        )
    """)

    # Добавляем Premium существующим пользователям
    try:
        conn.execute(
            "ALTER TABLE users ADD COLUMN premium INTEGER DEFAULT 0"
        )
    except sqlite3.OperationalError:
        pass

    conn.execute("""
        CREATE TABLE IF NOT EXISTS tasks (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            user_id INTEGER NOT NULL,
            title TEXT NOT NULL,
            subject TEXT,
            deadline TEXT,
            completed INTEGER DEFAULT 0,
            FOREIGN KEY (user_id) REFERENCES users(id)
        )
    """)

    conn.commit()
    conn.close()


# =========================================================
# ГЛАВНАЯ
# =========================================================

@app.route("/")
def index():

    return render_template(
        "index.html",
        user=session.get("username")
    )


# =========================================================
# РЕГИСТРАЦИЯ
# =========================================================

@app.route("/register", methods=["GET", "POST"])
def register():

    if request.method == "POST":

        username = request.form.get(
            "username",
            ""
        ).strip()

        password = request.form.get(
            "password",
            ""
        )

        if not username or not password:

            return render_template(
                "register.html",
                error="Введите логин и пароль"
            )

        if len(username) < 3:

            return render_template(
                "register.html",
                error="Логин должен содержать минимум 3 символа"
            )

        if len(password) < 6:

            return render_template(
                "register.html",
                error="Пароль должен содержать минимум 6 символов"
            )

        password_hash = generate_password_hash(password)

        conn = get_db()

        try:

            conn.execute(
                """
                INSERT INTO users
                (username, password, premium)
                VALUES (?, ?, 0)
                """,
                (
                    username,
                    password_hash
                )
            )

            conn.commit()
            conn.close()

            return redirect("/login")

        except sqlite3.IntegrityError:

            conn.close()

            return render_template(
                "register.html",
                error="Такой пользователь уже существует"
            )

    return render_template("register.html")


# =========================================================
# ВХОД
# =========================================================

@app.route("/login", methods=["GET", "POST"])
def login():

    if request.method == "POST":

        username = request.form.get(
            "username",
            ""
        ).strip()

        password = request.form.get(
            "password",
            ""
        )

        if not username or not password:

            return render_template(
                "login.html",
                error="Введите логин и пароль"
            )

        conn = get_db()

        user = conn.execute(
            """
            SELECT id, username, password, premium
            FROM users
            WHERE username = ?
            """,
            (username,)
        ).fetchone()

        conn.close()

        if user is None:

            return render_template(
                "login.html",
                error="Пользователь не найден"
            )

        if not check_password_hash(
            user["password"],
            password
        ):

            return render_template(
                "login.html",
                error="Неверный пароль"
            )

        session.clear()

        session["user_id"] = user["id"]
        session["username"] = user["username"]

        return redirect("/dashboard")

    return render_template("login.html")


# =========================================================
# КАБИНЕТ
# =========================================================

@app.route("/dashboard")
def dashboard():

    if "user_id" not in session:
        return redirect("/login")

    user_id = session["user_id"]

    conn = get_db()

    total_tasks = conn.execute(
        """
        SELECT COUNT(*)
        FROM tasks
        WHERE user_id = ?
        """,
        (user_id,)
    ).fetchone()[0]

    completed_tasks = conn.execute(
        """
        SELECT COUNT(*)
        FROM tasks
        WHERE user_id = ?
        AND completed = 1
        """,
        (user_id,)
    ).fetchone()[0]

    remaining_tasks = (
        total_tasks - completed_tasks
    )

    if total_tasks > 0:

        progress = round(
            completed_tasks /
            total_tasks *
            100
        )

    else:

        progress = 0

    recent_tasks = conn.execute(
        """
        SELECT *
        FROM tasks
        WHERE user_id = ?
        ORDER BY completed ASC, id DESC
        LIMIT 5
        """,
        (user_id,)
    ).fetchall()

    conn.close()

    return render_template(
        "dashboard.html",
        user=session["username"],
        total_tasks=total_tasks,
        completed_tasks=completed_tasks,
        remaining_tasks=remaining_tasks,
        progress=progress,
        recent_tasks=recent_tasks
    )


# =========================================================
# ЗАДАНИЯ
# =========================================================

@app.route("/tasks")
def tasks():

    if "user_id" not in session:
        return redirect("/login")

    conn = get_db()

    tasks_list = conn.execute(
        """
        SELECT *
        FROM tasks
        WHERE user_id = ?
        ORDER BY completed ASC, id DESC
        """,
        (session["user_id"],)
    ).fetchall()

    conn.close()

    return render_template(
        "tasks.html",
        user=session["username"],
        tasks=tasks_list
    )


# =========================================================
# ДОБАВЛЕНИЕ ЗАДАНИЯ
# =========================================================

@app.route("/tasks/add", methods=["POST"])
def add_task():

    if "user_id" not in session:
        return redirect("/login")

    title = request.form.get(
        "title",
        ""
    ).strip()

    subject = request.form.get(
        "subject",
        ""
    ).strip()

    deadline = request.form.get(
        "deadline",
        ""
    ).strip()

    if title:

        conn = get_db()

        conn.execute(
            """
            INSERT INTO tasks
            (user_id, title, subject, deadline, completed)
            VALUES (?, ?, ?, ?, 0)
            """,
            (
                session["user_id"],
                title,
                subject,
                deadline
            )
        )

        conn.commit()
        conn.close()

    return redirect("/tasks")


# =========================================================
# ВЫПОЛНЕНИЕ ЗАДАНИЯ
# =========================================================

@app.route(
    "/tasks/complete/<int:task_id>",
    methods=["POST"]
)
def complete_task(task_id):

    if "user_id" not in session:
        return redirect("/login")

    conn = get_db()

    task = conn.execute(
        """
        SELECT *
        FROM tasks
        WHERE id = ?
        AND user_id = ?
        """,
        (
            task_id,
            session["user_id"]
        )
    ).fetchone()

    if task:

        new_status = (
            0
            if task["completed"]
            else 1
        )

        conn.execute(
            """
            UPDATE tasks
            SET completed = ?
            WHERE id = ?
            AND user_id = ?
            """,
            (
                new_status,
                task_id,
                session["user_id"]
            )
        )

        conn.commit()

    conn.close()

    return redirect("/tasks")


# =========================================================
# УДАЛЕНИЕ ЗАДАНИЯ
# =========================================================

@app.route(
    "/tasks/delete/<int:task_id>",
    methods=["POST"]
)
def delete_task(task_id):

    if "user_id" not in session:
        return redirect("/login")

    conn = get_db()

    conn.execute(
        """
        DELETE FROM tasks
        WHERE id = ?
        AND user_id = ?
        """,
        (
            task_id,
            session["user_id"]
        )
    )

    conn.commit()
    conn.close()

    return redirect("/tasks")


# =========================================================
# ПРОФИЛЬ
# =========================================================

@app.route("/profile")
def profile():

    if "user_id" not in session:
        return redirect("/login")

    user_id = session["user_id"]

    conn = get_db()

    user = conn.execute(
        """
        SELECT id, username, premium
        FROM users
        WHERE id = ?
        """,
        (user_id,)
    ).fetchone()

    total_tasks = conn.execute(
        """
        SELECT COUNT(*)
        FROM tasks
        WHERE user_id = ?
        """,
        (user_id,)
    ).fetchone()[0]

    completed_tasks = conn.execute(
        """
        SELECT COUNT(*)
        FROM tasks
        WHERE user_id = ?
        AND completed = 1
        """,
        (user_id,)
    ).fetchone()[0]

    remaining_tasks = (
        total_tasks - completed_tasks
    )

    if total_tasks > 0:

        progress = round(
            completed_tasks /
            total_tasks *
            100
        )

    else:

        progress = 0

    conn.close()

    return render_template(
        "profile.html",
        user=user,
        total_tasks=total_tasks,
        completed_tasks=completed_tasks,
        remaining_tasks=remaining_tasks,
        progress=progress
    )


# =========================================================
# РЕДАКТИРОВАНИЕ ПРОФИЛЯ
# =========================================================

@app.route(
    "/profile/edit",
    methods=["GET", "POST"]
)
def edit_profile():

    if "user_id" not in session:
        return redirect("/login")

    user_id = session["user_id"]

    conn = get_db()

    user = conn.execute(
        """
        SELECT id, username, password, premium
        FROM users
        WHERE id = ?
        """,
        (user_id,)
    ).fetchone()

    if user is None:

        conn.close()

        session.clear()

        return redirect("/login")

    if request.method == "POST":

        username = request.form.get(
            "username",
            ""
        ).strip()

        current_password = request.form.get(
            "current_password",
            ""
        )

        new_password = request.form.get(
            "new_password",
            ""
        )

        repeat_password = request.form.get(
            "repeat_password",
            ""
        )

        if not username:

            conn.close()

            return render_template(
                "edit_profile.html",
                user=user,
                error="Введите логин"
            )

        if len(username) < 3:

            conn.close()

            return render_template(
                "edit_profile.html",
                user=user,
                error="Логин должен содержать минимум 3 символа"
            )

        if not check_password_hash(
            user["password"],
            current_password
        ):

            conn.close()

            return render_template(
                "edit_profile.html",
                user=user,
                error="Неверный текущий пароль"
            )

        existing_user = conn.execute(
            """
            SELECT id
            FROM users
            WHERE username = ?
            AND id != ?
            """,
            (
                username,
                user_id
            )
        ).fetchone()

        if existing_user:

            conn.close()

            return render_template(
                "edit_profile.html",
                user=user,
                error="Этот логин уже занят"
            )

        password_hash = user["password"]

        if new_password:

            if len(new_password) < 6:

                conn.close()

                return render_template(
                    "edit_profile.html",
                    user=user,
                    error="Новый пароль должен содержать минимум 6 символов"
                )

            if new_password != repeat_password:

                conn.close()

                return render_template(
                    "edit_profile.html",
                    user=user,
                    error="Новые пароли не совпадают"
                )

            password_hash = generate_password_hash(
                new_password
            )

        try:

            conn.execute(
                """
                UPDATE users
                SET username = ?, password = ?
                WHERE id = ?
                """,
                (
                    username,
                    password_hash,
                    user_id
                )
            )

            conn.commit()
            conn.close()

            session["username"] = username

            return redirect("/profile")

        except sqlite3.IntegrityError:

            conn.close()

            return render_template(
                "edit_profile.html",
                user=user,
                error="Не удалось сохранить изменения"
            )

    conn.close()

    return render_template(
        "edit_profile.html",
        user=user
    )


# =========================================================
# ЗАРАБОТОК
# =========================================================

@app.route("/earn")
def earn():

    if "user_id" not in session:
        return redirect("/login")

    opportunities = [
        {
            "title": "Создание сайтов",
            "category": "IT",
            "icon": "🌐",
            "description": "Создавай простые сайты и лендинги для людей и небольших проектов.",
            "level": "Начальный",
            "skills": "HTML • CSS • Flask",
            "income": "Зависит от сложности проекта"
        },
        {
            "title": "Python-разработка",
            "category": "IT",
            "icon": "🐍",
            "description": "Создавай небольшие программы, автоматизацию и полезные приложения.",
            "level": "Начальный",
            "skills": "Python • Flask",
            "income": "Зависит от сложности проекта"
        },
        {
            "title": "Дизайн",
            "category": "Дизайн",
            "icon": "🎨",
            "description": "Создавай презентации, баннеры, обложки и интерфейсы.",
            "level": "Начальный",
            "skills": "Figma • Photoshop",
            "income": "Зависит от типа работы"
        },
        {
            "title": "AI-контент",
            "category": "AI",
            "icon": "🤖",
            "description": "Используй AI для подготовки текстов, идей и цифрового контента.",
            "level": "Начальный",
            "skills": "AI • Prompt Engineering",
            "income": "Зависит от объёма работы"
        },
        {
            "title": "Репетиторство",
            "category": "Образование",
            "icon": "📚",
            "description": "Помогай другим ученикам разобраться в предметах, которые хорошо знаешь.",
            "level": "Средний",
            "skills": "Знания • Объяснение",
            "income": "Зависит от формата занятий"
        },
        {
            "title": "Монтаж видео",
            "category": "Контент",
            "icon": "🎬",
            "description": "Создавай короткие ролики и другой видеоконтент.",
            "level": "Начальный",
            "skills": "CapCut • Premiere Pro",
            "income": "Зависит от сложности"
        },
        {
            "title": "Копирайтинг",
            "category": "Контент",
            "icon": "✍️",
            "description": "Пиши тексты для сайтов, социальных сетей и проектов.",
            "level": "Начальный",
            "skills": "Текст • AI • Редактура",
            "income": "Зависит от объёма"
        },
        {
            "title": "Создание презентаций",
            "category": "Дизайн",
            "icon": "📊",
            "description": "Создавай красивые учебные и рабочие презентации.",
            "level": "Начальный",
            "skills": "PowerPoint • Canva • Figma",
            "income": "Зависит от количества слайдов"
        },
        {
            "title": "Тестирование сайтов",
            "category": "IT",
            "icon": "🔎",
            "description": "Ищи ошибки в интерфейсах и проверяй сайты.",
            "level": "Начальный",
            "skills": "QA • Внимательность",
            "income": "Зависит от задания"
        },
        {
            "title": "Свой цифровой проект",
            "category": "Проекты",
            "icon": "🚀",
            "description": "Создай собственный сайт, приложение или полезный сервис.",
            "level": "Средний",
            "skills": "Идея • Разработка • Маркетинг",
            "income": "Зависит от модели проекта"
        }
    ]

    return render_template(
        "earn.html",
        user=session["username"],
        opportunities=opportunities
    )


# =========================================================
# PREMIUM
# =========================================================

@app.route("/premium")
def premium():

    if "user_id" not in session:
        return redirect("/login")

    conn = get_db()

    user = conn.execute(
        """
        SELECT id, username, premium
        FROM users
        WHERE id = ?
        """,
        (session["user_id"],)
    ).fetchone()

    conn.close()

    return render_template(
        "premium.html",
        user=user
    )


# =========================================================
# ТЕСТОВАЯ АКТИВАЦИЯ PREMIUM
# =========================================================

@app.route(
    "/premium/activate",
    methods=["POST"]
)
def activate_premium():

    if "user_id" not in session:
        return redirect("/login")

    conn = get_db()

    conn.execute(
        """
        UPDATE users
        SET premium = 1
        WHERE id = ?
        """,
        (session["user_id"],)
    )

    conn.commit()
    conn.close()

    return redirect("/premium")


# =========================================================
# AI
# =========================================================

@app.route(
    "/ai",
    methods=["GET", "POST"]
)
def ai_assistant():

    if "user_id" not in session:
        return redirect("/login")

    question = ""
    answer = ""
    error = ""

    if request.method == "POST":

        question = request.form.get(
            "question",
            ""
        ).strip()

        if not question:

            error = "Напиши вопрос для AI."

        elif not DIFY_API_URL:

            error = (
                "Не указан DIFY_API_URL "
                "в файле .env"
            )

        elif not DIFY_API_KEY:

            error = (
                "Не указан DIFY_API_KEY "
                "в файле .env"
            )

        else:

            try:

                response = requests.post(
                    f"{DIFY_API_URL}/workflows/run",

                    headers={
                        "Authorization":
                            f"Bearer {DIFY_API_KEY}",

                        "Content-Type":
                            "application/json"
                    },

                    json={
                        "inputs": {
                            "question": question
                        },

                        "response_mode":
                            "blocking",

                        "user":
                            str(session["user_id"])
                    },

                    timeout=120
                )

                if response.status_code != 200:

                    error = (
                        f"Dify вернул ошибку "
                        f"{response.status_code}: "
                        f"{response.text[:1000]}"
                    )

                else:

                    data = response.json()

                    outputs = data.get(
                        "data",
                        {}
                    ).get(
                        "outputs",
                        {}
                    )

                    answer = outputs.get(
                        "answer",
                        ""
                    )

                    if not answer:

                        answer = outputs.get(
                            "text",
                            ""
                        )

                    if not answer and outputs:

                        for value in outputs.values():

                            if (
                                isinstance(value, str)
                                and value.strip()
                            ):

                                answer = value

                                break

                    if not answer:

                        error = (
                            "Dify получил вопрос, "
                            "но не вернул текст. "
                            "Проверь Output в End node."
                        )

            except requests.exceptions.Timeout:

                error = (
                    "AI слишком долго отвечает. "
                    "Попробуй ещё раз."
                )

            except requests.exceptions.RequestException as e:

                error = (
                    f"Ошибка подключения к Dify: {e}"
                )

            except Exception as e:

                error = f"Ошибка: {e}"

    return render_template(
        "ai_planner.html",
        user=session["username"],
        question=question,
        answer=answer,
        error=error
    )


# =========================================================
# ВЫХОД
# =========================================================

@app.route("/logout")
def logout():

    session.clear()

    return redirect("/")


# =========================================================
# ЗАПУСК
# =========================================================

if __name__ == "__main__":

    init_db()

    print()
    print("================================")
    print("      STUDENTHUB STARTED")
    print("================================")
    print("Главная:        http://127.0.0.1:5000/")
    print("Регистрация:   http://127.0.0.1:5000/register")
    print("Вход:           http://127.0.0.1:5000/login")
    print("Кабинет:        http://127.0.0.1:5000/dashboard")
    print("Задания:        http://127.0.0.1:5000/tasks")
    print("Заработок:      http://127.0.0.1:5000/earn")
    print("Premium:        http://127.0.0.1:5000/premium")
    print("AI:             http://127.0.0.1:5000/ai")
    print("Профиль:        http://127.0.0.1:5000/profile")
    print("Редактирование: http://127.0.0.1:5000/profile/edit")
    print("================================")
    print()

    app.run(
        debug=True,
        host="127.0.0.1",
        port=5000
    )