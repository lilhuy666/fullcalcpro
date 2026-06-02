import sys
import tkinter as tk
from tkinter import messagebox, ttk
import mysql.connector
from mysql.connector import Error


# ─────────────────────────────────────────────
#  НАСТРОЙКИ ПОДКЛЮЧЕНИЯ — измени под себя
# ─────────────────────────────────────────────
DB_CONFIG = {
    "host": "localhost",
    "user": "root",
    "password": "1234",   # <-- вставь свой пароль MySQL
    "database": "shop_db",
    "autocommit": True,
}


def get_connection():
    """Создаёт новое соединение с БД."""
    return mysql.connector.connect(**DB_CONFIG)


# ─────────────────────────────────────────────
#  ОКНО ТОВАРОВ
# ─────────────────────────────────────────────
def show_products(role="guest"):
    products_window = tk.Toplevel(root)
    products_window.title("Список товаров")
    products_window.geometry("600x400")
    products_window.resizable(True, True)

    # Заголовок с ролью
    header = "Вы вошли как: гость" if role == "guest" else f"Вы вошли как: {role}"
    tk.Label(products_window, text=header, font=("Arial", 11, "italic"),
             fg="gray").pack(anchor="w", padx=10, pady=(10, 0))

    # Таблица товаров
    frame = tk.Frame(products_window)
    frame.pack(fill="both", expand=True, padx=10, pady=10)

    columns = ("Название", "Цена (руб)", "Остаток")
    tree = ttk.Treeview(products_window, columns=columns, show="headings")
    for col in columns:
        tree.heading(col, text=col)
        tree.column(col, width=180)

    scrollbar = ttk.Scrollbar(products_window, orient="vertical", command=tree.yview)
    tree.configure(yscrollcommand=scrollbar.set)
    tree.pack(side="left", fill="both", expand=True, padx=(10, 0), pady=10)
    scrollbar.pack(side="right", fill="y", pady=10, padx=(0, 10))

    try:
        conn = get_connection()
        cursor = conn.cursor(dictionary=True)
        cursor.execute("SELECT name, price, count FROM Products")
        products = cursor.fetchall()
        cursor.close()
        conn.close()

        if not products:
            tk.Label(products_window, text="Товары не найдены",
                     font=("Arial", 12), fg="gray").pack(pady=20)
            return

        for product in products:
            tree.insert("", "end", values=(
                product["name"],
                f"{product['price']:.2f}",
                product["count"]
            ))

    except Error as err:
        messagebox.showerror("Ошибка БД", str(err), parent=products_window)


# ─────────────────────────────────────────────
#  АВТОРИЗАЦИЯ
# ─────────────────────────────────────────────
def login():
    user = entry_user.get().strip()
    pwd = entry_pass.get().strip()

    if not user or not pwd:
        messagebox.showwarning("Внимание", "Введите логин и пароль")
        return

    try:
        conn = get_connection()
        cursor = conn.cursor(dictionary=True)
        cursor.execute(
            "SELECT role FROM Users WHERE login = %s AND password = %s",
            (user, pwd)
        )
        result = cursor.fetchone()
        cursor.close()
        conn.close()

        if result:
            role = result["role"]
            messagebox.showinfo("Успех", f"Вход выполнен. Роль: {role}")
            show_products(role=role)
        else:
            messagebox.showerror("Ошибка", "Неверный логин или пароль")

    except Error as err:
        messagebox.showerror("Ошибка БД", str(err))


def guest_login():
    show_products(role="guest")


# ─────────────────────────────────────────────
#  ПРОВЕРКА СОЕДИНЕНИЯ ПРИ СТАРТЕ
# ─────────────────────────────────────────────
def check_connection():
    try:
        conn = get_connection()
        conn.close()
        return True
    except Error as err:
        messagebox.showerror(
            "Ошибка подключения",
            f"Не удалось подключиться к БД:\n{err}\n\nПроверь настройки DB_CONFIG в начале файла."
        )
        return False


# ─────────────────────────────────────────────
#  ГЛАВНОЕ ОКНО
# ─────────────────────────────────────────────
root = tk.Tk()
root.title("Авторизация")
root.geometry("320x220")
root.resizable(False, False)

# Проверяем соединение до запуска UI
if not check_connection():
    sys.exit(1)

tk.Label(root, text="Логин", font=("Arial", 11)).pack(pady=(15, 2))
entry_user = tk.Entry(root, font=("Arial", 11), width=24)
entry_user.pack()

tk.Label(root, text="Пароль", font=("Arial", 11)).pack(pady=(10, 2))
entry_pass = tk.Entry(root, show="*", font=("Arial", 11), width=24)
entry_pass.pack()

# Нажатие Enter = войти
root.bind("<Return>", lambda event: login())

tk.Button(root, text="Войти", command=login,
          font=("Arial", 11), width=20).pack(pady=(15, 4))
tk.Button(root, text="Войти как гость", command=guest_login,
          font=("Arial", 10), width=20).pack()

root.mainloop()
