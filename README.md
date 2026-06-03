import tkinter as tk
from tkinter import ttk, messagebox
import mysql.connector
import os
from PIL import Image, ImageTk

# ================== НАСТРОЙКА ПОДКЛЮЧЕНИЯ К БД ==================
try:
    conn = mysql.connector.connect(
        host="localhost",
        user="root",
        password="1234",        # <-- ВАШ ПАРОЛЬ MYSQL
        database="shop_db"
    )
    cursor = conn.cursor(dictionary=True)
except mysql.connector.Error as err:
    print("Ошибка подключения к БД:", err)
    exit()

# Путь к картинке-заглушке
PLACEHOLDER_PATH = "picture.png"

# ================== ВСПОМОГАТЕЛЬНЫЕ ФУНКЦИИ ==================
def get_placeholder_image(size=(120, 120)):
    """Возвращает PhotoImage заглушки. Если файла нет — серая картинка."""
    if os.path.exists(PLACEHOLDER_PATH):
        img = Image.open(PLACEHOLDER_PATH).resize(size, Image.LANCZOS)
        return ImageTk.PhotoImage(img)
    else:
        # Создаём серую заглушку
        img = Image.new('RGB', size, color='#CCCCCC')
        return ImageTk.PhotoImage(img)

def load_product_image(image_path, size=(120, 120)):
    """Загружает фото товара или возвращает заглушку."""
    if image_path and os.path.exists(image_path):
        img = Image.open(image_path).resize(size, Image.LANCZOS)
        return ImageTk.PhotoImage(img)
    return get_placeholder_image(size)

# ================== ГЛАВНЫЙ ЭКРАН ВХОДА ==================
class LoginWindow:
    def __init__(self, master):
        self.master = master
        self.master.title("Авторизация")
        self.master.geometry("350x250")
        self.master.resizable(False, False)

        tk.Label(master, text="Логин", font=("Arial", 12)).pack(pady=5)
        self.entry_user = tk.Entry(master, font=("Arial", 12))
        self.entry_user.pack(pady=2)

        tk.Label(master, text="Пароль", font=("Arial", 12)).pack(pady=5)
        self.entry_pass = tk.Entry(master, show="*", font=("Arial", 12))
        self.entry_pass.pack(pady=2)

        tk.Button(master, text="Войти", font=("Arial", 12), width=15,
                  command=self.login).pack(pady=8)

        tk.Button(master, text="Войти как гость", font=("Arial", 10),
                  command=self.guest_login).pack()

    def login(self):
        user = self.entry_user.get()
        pwd = self.entry_pass.get()

        try:
            query = "SELECT * FROM Users WHERE login=%s AND password=%s"
            cursor.execute(query, (user, pwd))
            result = cursor.fetchone()

            if result:
                self.master.withdraw()  # Скрываем окно входа
                role = result['role']
                if role == 'admin':
                    AdminWindow(self.master, result)
                elif role == 'manager':
                    ManagerWindow(self.master, result)
                elif role == 'client':
                    ClientWindow(self.master, result)
                else:
                    GuestWindow(self.master, result)
            else:
                messagebox.showerror("Ошибка", "Неверный логин или пароль")
        except mysql.connector.Error as err:
            messagebox.showerror("Ошибка БД", str(err))

    def guest_login(self):
        self.master.withdraw()
        GuestWindow(self.master, None)

# ================== БАЗОВЫЙ КЛАСС ОКНА С ТОВАРАМИ ==================
class ProductListMixin:
    """Примесь для отображения списка товаров с подсветкой и форматированием."""
    def create_product_list(self, parent_frame):
        """Создаёт прокручиваемую область с товарами. parent_frame — куда разместить."""
        # Canvas + Scrollbar
        self.canvas = tk.Canvas(parent_frame, highlightthickness=0)
        scrollbar = ttk.Scrollbar(parent_frame, orient="vertical", command=self.canvas.yview)
        self.scrollable_frame = ttk.Frame(self.canvas)

        self.scrollable_frame.bind("<Configure>",
            lambda e: self.canvas.configure(scrollregion=self.canvas.bbox("all")))

        self.canvas.create_window((0, 0), window=self.scrollable_frame, anchor="nw")
        self.canvas.configure(yscrollcommand=scrollbar.set)

        self.canvas.pack(side="left", fill="both", expand=True)
        scrollbar.pack(side="right", fill="y")

        # Загружаем товары
        self.load_products()

    def load_products(self):
        """Запрашивает товары из БД и рисует карточки."""
        # Очистка старых виджетов
        for widget in self.scrollable_frame.winfo_children():
            widget.destroy()

        try:
            cursor.execute("""
                SELECT p.*, c.name as category_name
                FROM Products p
                LEFT JOIN Categories c ON p.category_id = c.id
            """)
            products = cursor.fetchall()

            for product in products:
                self.create_product_card(product)

        except mysql.connector.Error as err:
            messagebox.showerror("Ошибка", str(err))

    def create_product_card(self, product):
        """Создаёт карточку одного товара с учётом скидки и наличия."""
        discount = product['discount']
        count = product['count']
        price = product['price']
        final_price = price * (1 - discount / 100) if discount else price

        # Определяем цвет фона
        bg_color = "#FFFFFF"
        if count == 0:
            bg_color = "#D3D3D3"  # Серый, если нет на складе
        elif discount > 15:
            bg_color = "#483D8B"  # Тёмно-синий, если скидка >15%

        fg_color = "white" if bg_color == "#483D8B" else "black"

        # Рамка карточки
        card_frame = tk.Frame(self.scrollable_frame, bg=bg_color, relief="groove", bd=2)
        card_frame.pack(fill="x", padx=10, pady=5, ipady=5)

        # Фото товара
        img = load_product_image(product['image_path'])
        img_label = tk.Label(card_frame, image=img, bg=bg_color)
        img_label.image = img  # сохраняем ссылку
        img_label.grid(row=0, column=0, rowspan=5, padx=10, pady=10, sticky="n")

        # Информация о товаре
        info_frame = tk.Frame(card_frame, bg=bg_color)
        info_frame.grid(row=0, column=1, sticky="w", padx=10)

        tk.Label(info_frame, text=f"{product['category_name']} | {product['name']}",
                 font=("Arial", 13, "bold"), bg=bg_color, fg=fg_color).pack(anchor="w")

        details = [
            f"Описание: {product['description'] or 'Нет описания'}",
            f"Производитель: {product['manufacturer'] or 'Не указан'}",
            f"Поставщик: {product['supplier'] or 'Не указан'}",
        ]

        for d in details:
            tk.Label(info_frame, text=d, font=("Arial", 10), bg=bg_color, fg=fg_color,
                     justify="left").pack(anchor="w")

        # Цена
        price_frame = tk.Frame(info_frame, bg=bg_color)
        price_frame.pack(anchor="w", pady=3)

        if discount > 0:
            # Зачёркнутая старая цена
            tk.Label(price_frame, text=f"{price:.2f} руб", font=("Arial", 11, "overstrike"),
                     fg="red", bg=bg_color).pack(side="left", padx=(0, 10))
            # Новая цена
            tk.Label(price_frame, text=f"{final_price:.2f} руб", font=("Arial", 11, "bold"),
                     fg="black" if bg_color != "#483D8B" else "white", bg=bg_color).pack(side="left")
        else:
            tk.Label(price_frame, text=f"Цена: {price:.2f} руб", font=("Arial", 11),
                     bg=bg_color, fg=fg_color).pack(anchor="w")

        # Остальные поля
        more_info = [
            f"Единица измерения: {product['unit']}",
            f"Количество на складе: {count}",
            f"Действующая скидка: {discount}%"
        ]
        for m in more_info:
            tk.Label(info_frame, text=m, font=("Arial", 10), bg=bg_color, fg=fg_color).pack(anchor="w")

# ================== ОКНА ДЛЯ РАЗНЫХ РОЛЕЙ ==================
class BaseRoleWindow:
    """Базовый класс для окон ролей с заголовком ФИО и кнопкой выхода."""
    def __init__(self, master, user):
        self.master = master
        self.user = user
        self.window = tk.Toplevel(master)
        self.window.state('zoomed')  # На весь экран
        self.window.protocol("WM_DELETE_WINDOW", self.logout)

        # Верхняя панель с ФИО и кнопкой выхода
        top_bar = tk.Frame(self.window, bg="#E0E0E0", height=40)
        top_bar.pack(fill="x", side="top")

        if user:
            full_name = user.get('full_name', 'Неизвестный пользователь')
            tk.Label(top_bar, text=f"Пользователь: {full_name}",
                     font=("Arial", 12, "bold"), bg="#E0E0E0").pack(side="left", padx=10, pady=5)

        tk.Button(top_bar, text="Выход", command=self.logout,
                 font=("Arial", 10), bg="#F44336", fg="white").pack(side="right", padx=10, pady=3)

        # Заголовок окна
        role_name = self.user['role'] if self.user else 'guest'
        tk.Label(self.window, text=f"Интерфейс {role_name}",
                 font=("Arial", 16, "bold")).pack(pady=10)

    def logout(self):
        """Возврат на окно входа."""
        self.window.destroy()
        self.master.deiconify()  # Показать окно входа

# Миксины для расширения функционала (пока заглушки)
class AdminMixin:
    def setup_admin_controls(self):
        tk.Button(self.window, text="Управление пользователями (заглушка)").pack(pady=10)
        tk.Button(self.window, text="Управление товарами (заглушка)").pack(pady=10)

class ManagerMixin:
    def setup_manager_controls(self):
        tk.Button(self.window, text="Управление заказами (заглушка)").pack(pady=10)

class ClientMixin:
    def setup_client_controls(self):
        tk.Button(self.window, text="Мои заказы (заглушка)").pack(pady=10)

# ================== КОНКРЕТНЫЕ ОКНА ==================
class AdminWindow(BaseRoleWindow, ProductListMixin, AdminMixin):
    def __init__(self, master, user):
        super().__init__(master, user)
        self.window.title("Панель администратора")

        # Элементы администратора
        self.setup_admin_controls()

        # Список товаров
        products_frame = tk.Frame(self.window)
        products_frame.pack(fill="both", expand=True, padx=10, pady=10)
        self.create_product_list(products_frame)

class ManagerWindow(BaseRoleWindow, ProductListMixin, ManagerMixin):
    def __init__(self, master, user):
        super().__init__(master, user)
        self.window.title("Панель менеджера")

        self.setup_manager_controls()

        products_frame = tk.Frame(self.window)
        products_frame.pack(fill="both", expand=True, padx=10, pady=10)
        self.create_product_list(products_frame)

class ClientWindow(BaseRoleWindow, ProductListMixin, ClientMixin):
    def __init__(self, master, user):
        super().__init__(master, user)
        self.window.title("Личный кабинет клиента")

        self.setup_client_controls()

        products_frame = tk.Frame(self.window)
        products_frame.pack(fill="both", expand=True, padx=10, pady=10)
        self.create_product_list(products_frame)

class GuestWindow(BaseRoleWindow, ProductListMixin):
    def __init__(self, master, user):
        super().__init__(master, user)
        self.window.title("Просмотр товаров (Гость)")

        products_frame = tk.Frame(self.window)
        products_frame.pack(fill="both", expand=True, padx=10, pady=10)
        self.create_product_list(products_frame)

# ================== ЗАПУСК ==================
if __name__ == "__main__":
    root = tk.Tk()
    LoginWindow(root)
    root.mainloop()


    CREATE DATABASE shop_db;
USE shop_db;

-- Пользователи
CREATE TABLE Users (
    id INT AUTO_INCREMENT PRIMARY KEY,
    login VARCHAR(50) UNIQUE NOT NULL,
    password VARCHAR(255) NOT NULL,       -- В реальном проекте хранить хеш!
    role VARCHAR(20) NOT NULL,            -- admin, manager, client, guest
    full_name VARCHAR(100) NOT NULL       -- ФИО
);

INSERT INTO Users(login, password, role, full_name) VALUES
('admin', '123', 'admin', 'Иванов Иван Иванович'),
('manager', '123', 'manager', 'Петров Пётр Петрович'),
('client', '123', 'client', 'Сидорова Анна Сергеевна');

-- Категории
CREATE TABLE Categories (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(100) NOT NULL
);

INSERT INTO Categories(name) VALUES
('Обувь'), ('Одежда'), ('Аксессуары');

-- Продукты
CREATE TABLE Products (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    category_id INT,
    description TEXT,
    manufacturer VARCHAR(100),
    supplier VARCHAR(100),
    price DECIMAL(10,2) NOT NULL,
    unit VARCHAR(20) DEFAULT 'шт',
    count INT DEFAULT 0,
    discount INT DEFAULT 0,              -- Скидка в процентах (0-100)
    image_path VARCHAR(255) DEFAULT NULL, -- Путь к файлу фото (или NULL)
    FOREIGN KEY (category_id) REFERENCES Categories(id)
);

INSERT INTO Products(name, category_id, description, manufacturer, supplier, price, unit, count, discount, image_path) VALUES
('Кроссовки', 1, 'Спортивные кроссовки для бега', 'Nike', 'ООО СпортТорг', 5000.00, 'пара', 10, 20, NULL),
('Ботинки', 1, 'Кожаные ботинки', 'Timberland', 'ИП Петров', 8000.00, 'пара', 0, 5, NULL),
('Сандалии', 1, 'Летние сандалии', 'Crocs', 'ООО ЛетоСтиль', 2000.00, 'пара', 25, 0, NULL),
('Футболка', 2, 'Хлопковая футболка', 'Adidas', 'ООО СпортТорг', 1500.00, 'шт', 40, 17, NULL);
