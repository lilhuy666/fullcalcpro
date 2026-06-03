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
        try:
            img = Image.open(PLACEHOLDER_PATH).resize(size, Image.LANCZOS)
            return ImageTk.PhotoImage(img)
        except Exception:
            pass
    # Создаём серую заглушку если файл не найден или повреждён
    img = Image.new('RGB', size, color='#CCCCCC')
    return ImageTk.PhotoImage(img)

def load_product_image(image_path, size=(120, 120)):
    """Загружает фото товара или возвращает заглушку."""
    if image_path and os.path.exists(image_path):
        try:
            img = Image.open(image_path).resize(size, Image.LANCZOS)
            return ImageTk.PhotoImage(img)
        except Exception:
            pass
    return get_placeholder_image(size)

# ================== ГЛАВНЫЙ ЭКРАН ВХОДА ==================
class LoginWindow:
    def __init__(self, master):
        self.master = master
        self.master.title("Авторизация")
        self.master.geometry("350x250")
        self.master.resizable(False, False)
        self.current_window = None  # Отслеживаем открытое окно

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

        if not user or not pwd:
            messagebox.showwarning("Предупреждение", "Введите логин и пароль")
            return

        try:
            query = "SELECT * FROM Users WHERE login=%s AND password=%s"
            cursor.execute(query, (user, pwd))
            result = cursor.fetchone()

            if result:
                self.master.withdraw()
                role = result['role']
                
                if self.current_window is not None:
                    self.current_window.destroy()
                
                if role == 'admin':
                    self.current_window = AdminWindow(self.master, result)
                elif role == 'manager':
                    self.current_window = ManagerWindow(self.master, result)
                elif role == 'client':
                    self.current_window = ClientWindow(self.master, result)
                else:
                    self.current_window = GuestWindow(self.master, None)
            else:
                messagebox.showerror("Ошибка", "Неверный логин или пароль")
        except mysql.connector.Error as err:
            messagebox.showerror("Ошибка БД", str(err))

    def guest_login(self):
        if self.current_window is not None:
            self.current_window.destroy()
        
        self.master.withdraw()
        self.current_window = GuestWindow(self.master, None)

# ================== БАЗОВЫЙ КЛАСС ОКНА С ТОВАРАМИ ==================
class ProductListMixin:
    """Примесь для отображения списка товаров с подсветкой и форматированием."""
    def create_product_list(self, parent_frame):
        """Создаёт прокручиваемую область с товарами."""
        # Canvas + Scrollbar
        self.canvas = tk.Canvas(parent_frame, highlightthickness=0, bg="#F5F5F5")
        scrollbar = ttk.Scrollbar(parent_frame, orient="vertical", command=self.canvas.yview)
        self.scrollable_frame = ttk.Frame(self.canvas)

        self.scrollable_frame.bind("<Configure>",
            lambda e: self.canvas.configure(scrollregion=self.canvas.bbox("all")))

        self.canvas_window = self.canvas.create_window((0, 0), window=self.scrollable_frame, anchor="nw")

        # Привязка изменения ширины canvas к ширине scrollable_frame
        self.canvas.bind('<Configure>', self._configure_canvas)

        self.canvas.configure(yscrollcommand=scrollbar.set)

        self.canvas.pack(side="left", fill="both", expand=True)
        scrollbar.pack(side="right", fill="y")
        
        # Привязка колеса мыши
        self.canvas.bind('<MouseWheel>', self._on_mousewheel)

        # Загружаем товары
        self.load_products()

    def _configure_canvas(self, event):
        """Подгоняет ширину внутреннего фрейма под canvas"""
        self.canvas.itemconfig(self.canvas_window, width=event.width)

    def _on_mousewheel(self, event):
        """Прокрутка колёсиком мыши"""
        self.canvas.yview_scroll(int(-1*(event.delta/120)), "units")

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
                ORDER BY p.name
            """)
            products = cursor.fetchall()

            if not products:
                tk.Label(self.scrollable_frame, text="Товары не найдены", 
                        font=("Arial", 14), fg="gray").pack(pady=50)
                return

            for product in products:
                self.create_product_card(product)

        except mysql.connector.Error as err:
            messagebox.showerror("Ошибка", f"Не удалось загрузить товары: {err}")

    def create_product_card(self, product):
        """Создаёт карточку одного товара с учётом скидки и наличия."""
        discount = int(product['discount'] or 0)
        count = int(product['count'] or 0)
        price = float(product['price'])
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
        card_frame.pack(fill="x", padx=15, pady=8, ipady=10)

        # Фото товара
        img = load_product_image(product.get('image_path'))
        img_label = tk.Label(card_frame, image=img, bg=bg_color, bd=0)
        img_label.image = img  # сохраняем ссылку
        img_label.grid(row=0, column=0, rowspan=6, padx=15, pady=10, sticky="n")

        # Информация о товаре
        info_frame = tk.Frame(card_frame, bg=bg_color)
        info_frame.grid(row=0, column=1, sticky="w", padx=15)

        # Название и категория
        category = product.get('category_name', 'Без категории')
        name = product.get('name', 'Без названия')
        tk.Label(info_frame, text=f"{category} | {name}",
                 font=("Arial", 14, "bold"), bg=bg_color, fg=fg_color).pack(anchor="w", pady=(0, 5))

        # Детали
        details = [
            f"📝 Описание: {product.get('description') or 'Нет описания'}",
            f"🏭 Производитель: {product.get('manufacturer') or 'Не указан'}",
            f"🚚 Поставщик: {product.get('supplier') or 'Не указан'}",
        ]

        for d in details:
            tk.Label(info_frame, text=d, font=("Arial", 10), bg=bg_color, fg=fg_color,
                     justify="left").pack(anchor="w", pady=1)

        # Цена
        price_frame = tk.Frame(info_frame, bg=bg_color)
        price_frame.pack(anchor="w", pady=5)

        if discount > 0:
            # Зачёркнутая старая цена
            tk.Label(price_frame, text=f"{price:.2f} руб.", font=("Arial", 11, "overstrike"),
                     fg="red", bg=bg_color).pack(side="left", padx=(0, 15))
            # Новая цена
            tk.Label(price_frame, text=f"{final_price:.2f} руб.", font=("Arial", 12, "bold"),
                     fg="black" if bg_color != "#483D8B" else "white", bg=bg_color).pack(side="left")
        else:
            tk.Label(price_frame, text=f"💰 Цена: {price:.2f} руб.", font=("Arial", 11, "bold"),
                     bg=bg_color, fg=fg_color).pack(anchor="w")

        # Остальные поля
        more_info = [
            f"📏 Единица измерения: {product.get('unit', 'шт')}",
            f"📊 Количество на складе: {count} {'❌ Нет в наличии' if count == 0 else '✅ В наличии'}",
            f"🏷️ Действующая скидка: {discount}%"
        ]
        for m in more_info:
            tk.Label(info_frame, text=m, font=("Arial", 10), bg=bg_color, fg=fg_color).pack(anchor="w", pady=1)

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
        top_bar = tk.Frame(self.window, bg="#2C3E50", height=50)
        top_bar.pack(fill="x", side="top")
        top_bar.pack_propagate(False)

        # ФИО пользователя
        if user:
            full_name = user.get('full_name', 'Пользователь')
            role = user.get('role', '')
        else:
            full_name = 'Гость'
            role = 'guest'

        tk.Label(top_bar, text=f"👤 {full_name} ({role})",
                 font=("Arial", 12, "bold"), fg="white", bg="#2C3E50").pack(side="left", padx=20, pady=10)

        # Кнопка выхода
        tk.Button(top_bar, text="🚪 Выход", command=self.logout,
                 font=("Arial", 10, "bold"), bg="#E74C3C", fg="white",
                 relief="flat", cursor="hand2").pack(side="right", padx=20, pady=8)

        # Заголовок окна
        role_names = {
            'admin': 'Администратор',
            'manager': 'Менеджер',
            'client': 'Клиент',
            'guest': 'Гость'
        }
        role_name = role_names.get(role, 'Пользователь')
        tk.Label(self.window, text=f"📋 Панель {role_name}",
                 font=("Arial", 18, "bold"), fg="#2C3E50").pack(pady=15)

        # Разделительная линия
        ttk.Separator(self.window, orient='horizontal').pack(fill='x', padx=20)

    def logout(self):
        """Возврат на окно входа."""
        self.window.destroy()
        self.master.deiconify()  # Показать окно входа

# Миксины для расширения функционала
class AdminMixin:
    def setup_admin_controls(self):
        controls_frame = tk.Frame(self.window, bg="#ECF0F1")
        controls_frame.pack(fill="x", padx=20, pady=10)
        
        tk.Label(controls_frame, text="🔧 Панель управления:", 
                font=("Arial", 12, "bold"), bg="#ECF0F1").pack(side="left", padx=10)
        
        tk.Button(controls_frame, text="👥 Пользователи (скоро)", 
                 font=("Arial", 10), bg="#3498DB", fg="white",
                 relief="flat", cursor="hand2").pack(side="left", padx=5)
        
        tk.Button(controls_frame, text="📦 Товары (скоро)", 
                 font=("Arial", 10), bg="#2ECC71", fg="white",
                 relief="flat", cursor="hand2").pack(side="left", padx=5)
        
        tk.Button(controls_frame, text="📊 Отчёты (скоро)", 
                 font=("Arial", 10), bg="#F39C12", fg="white",
                 relief="flat", cursor="hand2").pack(side="left", padx=5)

class ManagerMixin:
    def setup_manager_controls(self):
        controls_frame = tk.Frame(self.window, bg="#ECF0F1")
        controls_frame.pack(fill="x", padx=20, pady=10)
        
        tk.Label(controls_frame, text="📋 Управление заказами:", 
                font=("Arial", 12, "bold"), bg="#ECF0F1").pack(side="left", padx=10)
        
        tk.Button(controls_frame, text="📝 Заказы (скоро)", 
                 font=("Arial", 10), bg="#3498DB", fg="white",
                 relief="flat", cursor="hand2").pack(side="left", padx=5)

class ClientMixin:
    def setup_client_controls(self):
        controls_frame = tk.Frame(self.window, bg="#ECF0F1")
        controls_frame.pack(fill="x", padx=20, pady=10)
        
        tk.Label(controls_frame, text="🛒 Мои покупки:", 
                font=("Arial", 12, "bold"), bg="#ECF0F1").pack(side="left", padx=10)
        
        tk.Button(controls_frame, text="📋 История заказов (скоро)", 
                 font=("Arial", 10), bg="#3498DB", fg="white",
                 relief="flat", cursor="hand2").pack(side="left", padx=5)

# ================== КОНКРЕТНЫЕ ОКНА ==================
class AdminWindow(BaseRoleWindow, ProductListMixin, AdminMixin):
    def __init__(self, master, user):
        super().__init__(master, user)
        self.window.title("Панель администратора")
        self.setup_admin_controls()

        products_frame = tk.Frame(self.window)
        products_frame.pack(fill="both", expand=True, padx=20, pady=10)
        self.create_product_list(products_frame)

class ManagerWindow(BaseRoleWindow, ProductListMixin, ManagerMixin):
    def __init__(self, master, user):
        super().__init__(master, user)
        self.window.title("Панель менеджера")
        self.setup_manager_controls()

        products_frame = tk.Frame(self.window)
        products_frame.pack(fill="both", expand=True, padx=20, pady=10)
        self.create_product_list(products_frame)

class ClientWindow(BaseRoleWindow, ProductListMixin, ClientMixin):
    def __init__(self, master, user):
        super().__init__(master, user)
        self.window.title("Личный кабинет клиента")
        self.setup_client_controls()

        products_frame = tk.Frame(self.window)
        products_frame.pack(fill="both", expand=True, padx=20, pady=10)
        self.create_product_list(products_frame)

class GuestWindow(BaseRoleWindow, ProductListMixin):
    def __init__(self, master, user):
        super().__init__(master, user)
        self.window.title("Просмотр товаров (Гость)")

        products_frame = tk.Frame(self.window)
        products_frame.pack(fill="both", expand=True, padx=20, pady=10)
        self.create_product_list(products_frame)

# ================== ЗАПУСК ==================
if __name__ == "__main__":
    root = tk.Tk()
    app = LoginWindow(root)
    root.mainloop()


    -- ============================================
-- СОЗДАНИЕ БАЗЫ ДАННЫХ
-- ============================================
DROP DATABASE IF EXISTS shop_db;
CREATE DATABASE shop_db CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
USE shop_db;

-- ============================================
-- ТАБЛИЦА ПОЛЬЗОВАТЕЛЕЙ
-- ============================================
CREATE TABLE Users (
    id INT AUTO_INCREMENT PRIMARY KEY,
    login VARCHAR(50) UNIQUE NOT NULL,
    password VARCHAR(255) NOT NULL,
    role ENUM('admin', 'manager', 'client') NOT NULL,
    full_name VARCHAR(150) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- Пароли: 123 (в реальном проекте нужно хешировать!)
INSERT INTO Users (login, password, role, full_name) VALUES
('admin', '123', 'admin', 'Иванов Иван Иванович'),
('manager', '123', 'manager', 'Петров Пётр Петрович'),
('client', '123', 'client', 'Сидорова Анна Сергеевна'),
('client2', '123', 'client', 'Козлов Дмитрий Александрович'),
('manager2', '123', 'manager', 'Смирнова Елена Викторовна');

-- ============================================
-- ТАБЛИЦА КАТЕГОРИЙ
-- ============================================
CREATE TABLE Categories (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(100) NOT NULL UNIQUE,
    description TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

INSERT INTO Categories (name, description) VALUES
('Обувь', 'Все виды обуви: кроссовки, ботинки, сандалии и др.'),
('Одежда', 'Верхняя и повседневная одежда'),
('Аксессуары', 'Сумки, ремни, головные уборы'),
('Спортивные товары', 'Товары для спорта и активного отдыха'),
('Электроника', 'Гаджеты и электронные устройства');

-- ============================================
-- ТАБЛИЦА ТОВАРОВ
-- ============================================
CREATE TABLE Products (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(200) NOT NULL,
    category_id INT,
    description TEXT,
    manufacturer VARCHAR(150),
    supplier VARCHAR(150),
    price DECIMAL(10, 2) NOT NULL CHECK (price >= 0),
    unit VARCHAR(20) DEFAULT 'шт',
    count INT DEFAULT 0 CHECK (count >= 0),
    discount INT DEFAULT 0 CHECK (discount >= 0 AND discount <= 100),
    image_path VARCHAR(500) DEFAULT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    FOREIGN KEY (category_id) REFERENCES Categories(id) ON DELETE SET NULL
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- ============================================
-- ЗАПОЛНЕНИЕ ТОВАРОВ (разные сценарии для демонстрации)
-- ============================================

-- Товары со скидкой >15% (будет фиолетовый фон #483D8B)
INSERT INTO Products (name, category_id, description, manufacturer, supplier, price, unit, count, discount, image_path) VALUES
('Кроссовки Nike Air Max', 1, 'Спортивные кроссовки с амортизацией Air Max. Подходят для бега и повседневной носки.', 
 'Nike', 'ООО "СпортТорг"', 12999.00, 'пара', 25, 20, NULL),
 
('Зимняя куртка North Face', 2, 'Тёплая зимняя куртка с наполнителем из пуха. Водонепроницаемая ткань.', 
 'The North Face', 'ИП "Экипировка"', 25999.00, 'шт', 12, 25, NULL),
 
('Беспроводные наушники Sony', 5, 'Bluetooth наушники с шумоподавлением. 30 часов работы без подзарядки.', 
 'Sony', 'ООО "ТехноМир"', 15999.00, 'шт', 8, 18, NULL);

-- Товары с обычной скидкой или без скидки
INSERT INTO Products (name, category_id, description, manufacturer, supplier, price, unit, count, discount, image_path) VALUES
('Ботинки Timberland', 1, 'Классические кожаные ботинки. Износостойкая подошва, влагозащита.', 
 'Timberland', 'ИП "Петров А.В."', 18999.00, 'пара', 15, 5, NULL),
 
('Футболка Adidas', 2, 'Хлопковая футболка с логотипом. Дышащий материал.', 
 'Adidas', 'ООО "СпортТорг"', 2999.00, 'шт', 50, 0, NULL),
 
('Сандалии Crocs', 1, 'Летние сандалии из лёгкого материала. Анатомическая стелька.', 
 'Crocs', 'ООО "ЛетоСтиль"', 3999.00, 'пара', 30, 10, NULL),
 
('Рюкзак Herschel', 3, 'Городской рюкзак с отделением для ноутбука. Объём 25 литров.', 
 'Herschel Supply Co.', 'ООО "СтильМаркет"', 7999.00, 'шт', 20, 0, 'backpack.jpg');

-- Товары, которых нет на складе (будет серый фон)
INSERT INTO Products (name, category_id, description, manufacturer, supplier, price, unit, count, discount, image_path) VALUES
('Умные часы Apple Watch', 5, 'Смарт-часы с функцией ЭКГ и отслеживанием тренировок.', 
 'Apple', 'ООО "ТехноМир"', 45999.00, 'шт', 0, 5, NULL),
 
('Кожаный ремень', 3, 'Классический кожаный ремень. Ширина 3.5 см, пряжка из нержавеющей стали.', 
 'Calvin Klein', 'ООО "МодаСтиль"', 4999.00, 'шт', 0, 0, NULL),
 
('Спортивный костюм Puma', 4, 'Костюм для тренировок из влагоотводящей ткани.', 
 'Puma', 'ООО "СпортТорг"', 8999.00, 'шт', 0, 15, NULL);

-- Товары без скидки с большим количеством
INSERT INTO Products (name, category_id, description, manufacturer, supplier, price, unit, count, discount, image_path) VALUES
('Носки спортивные (набор 3 пары)', 4, 'Носки из хлопка с усиленной пяткой и мысом.', 
 'Nike', 'ООО "СпортТорг"', 999.00, 'набор', 100, 0, NULL),
 
('Кепка бейсболка', 3, 'Классическая бейсболка с регулируемым размером.', 
 'New Era', 'ООО "МодаСтиль"', 2499.00, 'шт', 45, 0, NULL),
 
('Шорты для бега', 4, 'Лёгкие шорты с внутренними трусами. Быстросохнущий материал.', 
 'Under Armour', 'ООО "СпортТорг"', 3499.00, 'шт', 35, 0, NULL);

-- ============================================
-- ДОПОЛНИТЕЛЬНЫЕ ТАБЛИЦЫ (на будущее)
-- ============================================

-- Таблица заказов
CREATE TABLE Orders (
    id INT AUTO_INCREMENT PRIMARY KEY,
    user_id INT NOT NULL,
    order_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    status ENUM('new', 'processing', 'completed', 'cancelled') DEFAULT 'new',
    total_amount DECIMAL(10, 2) DEFAULT 0,
    FOREIGN KEY (user_id) REFERENCES Users(id) ON DELETE CASCADE
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- Таблица позиций в заказе
CREATE TABLE OrderItems (
    id INT AUTO_INCREMENT PRIMARY KEY,
    order_id INT NOT NULL,
    product_id INT NOT NULL,
    quantity INT NOT NULL CHECK (quantity > 0),
    price DECIMAL(10, 2) NOT NULL,
    FOREIGN KEY (order_id) REFERENCES Orders(id) ON DELETE CASCADE,
    FOREIGN KEY (product_id) REFERENCES Products(id) ON DELETE CASCADE
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- Таблица корзины
CREATE TABLE Cart (
    id INT AUTO_INCREMENT PRIMARY KEY,
    user_id INT NOT NULL,
    product_id INT NOT NULL,
    quantity INT NOT NULL DEFAULT 1 CHECK (quantity > 0),
    added_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (user_id) REFERENCES Users(id) ON DELETE CASCADE,
    FOREIGN KEY (product_id) REFERENCES Products(id) ON DELETE CASCADE,
    UNIQUE KEY unique_cart (user_id, product_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- ============================================
-- ПРОВЕРОЧНЫЕ ЗАПРОСЫ
-- ============================================

-- Показать всех пользователей
SELECT * FROM Users;

-- Показать товары с категориями
SELECT 
    p.id,
    p.name AS 'Название',
    c.name AS 'Категория',
    p.price AS 'Цена',
    p.count AS 'Остаток',
    p.discount AS 'Скидка %',
    CASE 
        WHEN p.count = 0 THEN 'Нет в наличии'
        WHEN p.discount > 15 THEN 'Большая скидка'
        WHEN p.discount > 0 THEN 'Есть скидка'
        ELSE 'Обычная цена'
    END AS 'Статус'
FROM Products p
LEFT JOIN Categories c ON p.category_id = c.id
ORDER BY p.discount DESC, p.name;

-- Статистика по товарам
SELECT 
    COUNT(*) AS 'Всего товаров',
    SUM(count) AS 'Общий остаток',
    AVG(price) AS 'Средняя цена',
    COUNT(CASE WHEN discount > 15 THEN 1 END) AS 'Товаров со скидкой >15%',
    COUNT(CASE WHEN count = 0 THEN 1 END) AS 'Товаров нет в наличии'
FROM Products;
