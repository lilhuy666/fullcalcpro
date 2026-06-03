import tkinter as tk
from tkinter import ttk, messagebox
import mysql.connector

# ================== НАСТРОЙКА ПОДКЛЮЧЕНИЯ К БД ==================
try:
    conn = mysql.connector.connect(
        host="localhost",
        user="root",
        password="1234",        # ВАШ ПАРОЛЬ MYSQL
        database="shop_db",
        auth_plugin='mysql_native_password'
    )
    cursor = conn.cursor(dictionary=True)
    print("✅ Подключение к БД успешно установлено")
except mysql.connector.Error as err:
    print(f"❌ Ошибка подключения к БД: {err}")
    print("\n📌 Если ошибка связана с плагином аутентификации, выполните в MySQL:")
    print("ALTER USER 'root'@'localhost' IDENTIFIED WITH mysql_native_password BY '1234';")
    print("FLUSH PRIVILEGES;")
    exit()

# ================== ГЛАВНОЕ ОКНО ВХОДА ==================
class LoginWindow:
    def __init__(self, master):
        self.master = master
        self.master.title("Авторизация - Магазин")
        self.master.geometry("400x350")
        self.master.resizable(False, False)
        self.master.configure(bg="#F0F0F0")
        self.current_window = None
        
        # Центрирование окна
        self.master.update_idletasks()
        width = self.master.winfo_width()
        height = self.master.winfo_height()
        x = (self.master.winfo_screenwidth() // 2) - (width // 2)
        y = (self.master.winfo_screenheight() // 2) - (height // 2)
        self.master.geometry(f'{width}x{height}+{x}+{y}')

        # Заголовок
        tk.Label(master, text="🔐 АВТОРИЗАЦИЯ", font=("Arial", 18, "bold"), 
                bg="#F0F0F0", fg="#2C3E50").pack(pady=20)
        
        # Рамка для полей ввода
        frame = tk.Frame(master, bg="white", relief="groove", bd=2)
        frame.pack(pady=20, padx=30, fill="both")
        
        tk.Label(frame, text="Логин:", font=("Arial", 12), 
                bg="white", fg="#2C3E50").pack(pady=(15, 5))
        self.entry_user = tk.Entry(frame, font=("Arial", 14), width=25, 
                                   justify="center")
        self.entry_user.pack(pady=5, padx=20)
        self.entry_user.insert(0, "admin")  # Подсказка
        self.entry_user.focus()

        tk.Label(frame, text="Пароль:", font=("Arial", 12), 
                bg="white", fg="#2C3E50").pack(pady=(10, 5))
        self.entry_pass = tk.Entry(frame, show="*", font=("Arial", 14), 
                                   width=25, justify="center")
        self.entry_pass.pack(pady=5, padx=20)
        self.entry_pass.insert(0, "123")  # Подсказка
        
        # Привязка Enter к входу
        self.entry_pass.bind('<Return>', lambda event: self.login())

        # Кнопка входа
        tk.Button(master, text="🚪 ВОЙТИ", font=("Arial", 14, "bold"), 
                  width=20, height=2, bg="#2ECC71", fg="white",
                  command=self.login, cursor="hand2", relief="flat").pack(pady=15)

        # Кнопка гостя
        tk.Button(master, text="👤 ВОЙТИ КАК ГОСТЬ", font=("Arial", 12),
                  width=20, height=2, bg="#3498DB", fg="white",
                  command=self.guest_login, cursor="hand2", relief="flat").pack(pady=5)
        
        # Информация
        tk.Label(master, text="Тестовые логины: admin, manager, client\nПароль: 123", 
                font=("Arial", 9), bg="#F0F0F0", fg="#7F8C8D").pack(pady=15)

    def login(self):
        user = self.entry_user.get().strip()
        pwd = self.entry_pass.get().strip()

        if not user or not pwd:
            messagebox.showwarning("Предупреждение", "Введите логин и пароль")
            return

        try:
            query = "SELECT * FROM Users WHERE login=%s AND password=%s"
            cursor.execute(query, (user, pwd))
            result = cursor.fetchone()

            if result:
                self.open_role_window(result)
            else:
                messagebox.showerror("Ошибка", "Неверный логин или пароль")
        except mysql.connector.Error as err:
            messagebox.showerror("Ошибка БД", f"Ошибка при входе: {err}")

    def guest_login(self):
        self.open_role_window(None)

    def open_role_window(self, user):
        self.master.withdraw()
        
        if self.current_window is not None:
            try:
                self.current_window.destroy()
            except:
                pass
        
        try:
            if user is None:
                self.current_window = GuestWindow(self.master)
            else:
                role = user.get('role', 'guest')
                
                if role == 'admin':
                    self.current_window = AdminWindow(self.master, user)
                elif role == 'manager':
                    self.current_window = ManagerWindow(self.master, user)
                elif role == 'client':
                    self.current_window = ClientWindow(self.master, user)
                else:
                    self.current_window = GuestWindow(self.master)
        except Exception as e:
            messagebox.showerror("Ошибка", f"Не удалось открыть окно: {e}")
            self.master.deiconify()

# ================== БАЗОВЫЙ КЛАСС ДЛЯ ВСЕХ РОЛЕЙ ==================
class BaseWindow:
    def __init__(self, master, user=None):
        self.master = master
        self.user = user
        self.window = tk.Toplevel(master)
        self.window.title("Магазин")
        self.window.state('zoomed')  # На весь экран
        self.window.configure(bg="#ECF0F1")
        self.window.protocol("WM_DELETE_WINDOW", self.logout)
        
        self.create_top_bar()
        self.create_main_content()

    def create_top_bar(self):
        """Создание верхней панели с ФИО и кнопкой выхода"""
        top_bar = tk.Frame(self.window, bg="#2C3E50", height=60)
        top_bar.pack(fill="x", side="top")
        top_bar.pack_propagate(False)

        # Информация о пользователе
        if self.user:
            full_name = self.user.get('full_name', 'Пользователь')
            role = self.user.get('role', '')
            role_names = {
                'admin': 'Администратор',
                'manager': 'Менеджер',
                'client': 'Клиент'
            }
            role_display = role_names.get(role, role)
            user_info = f"👤 {full_name} | {role_display}"
        else:
            user_info = "👤 Гость | Просмотр товаров"

        tk.Label(top_bar, text=user_info,
                 font=("Arial", 14, "bold"), fg="white", bg="#2C3E50").pack(
                     side="left", padx=30, pady=15)

        # Кнопка выхода
        tk.Button(top_bar, text="🚪 ВЫЙТИ", font=("Arial", 12, "bold"),
                 bg="#E74C3C", fg="white", relief="flat", cursor="hand2",
                 command=self.logout, width=15, height=2).pack(
                     side="right", padx=20, pady=10)

    def create_main_content(self):
        """Создание основного содержимого"""
        # Заголовок
        if self.user:
            role = self.user.get('role', '')
            titles = {
                'admin': '📋 ПАНЕЛЬ АДМИНИСТРАТОРА',
                'manager': '📋 ПАНЕЛЬ МЕНЕДЖЕРА',
                'client': '📋 ЛИЧНЫЙ КАБИНЕТ КЛИЕНТА'
            }
            title_text = titles.get(role, '📋 ПАНЕЛЬ ПОЛЬЗОВАТЕЛЯ')
        else:
            title_text = '📋 ПРОСМОТР ТОВАРОВ (ГОСТЬ)'
        
        tk.Label(self.window, text=title_text, font=("Arial", 18, "bold"),
                bg="#ECF0F1", fg="#2C3E50").pack(pady=20)

        # Разделительная линия
        ttk.Separator(self.window, orient='horizontal').pack(fill='x', padx=20)

        # Контейнер для списка товаров
        self.products_frame = tk.Frame(self.window, bg="#ECF0F1")
        self.products_frame.pack(fill="both", expand=True, padx=20, pady=10)
        
        # Загрузка товаров
        self.load_products()

    def load_products(self):
        """Загрузка и отображение списка товаров"""
        # Очистка предыдущих товаров
        for widget in self.products_frame.winfo_children():
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
                tk.Label(self.products_frame, text="📦 Товары не найдены",
                        font=("Arial", 16), bg="#ECF0F1", fg="#95A5A6").pack(pady=50)
                return

            # Создание прокручиваемой области
            canvas = tk.Canvas(self.products_frame, bg="#ECF0F1", highlightthickness=0)
            scrollbar = ttk.Scrollbar(self.products_frame, orient="vertical", command=canvas.yview)
            scroll_frame = tk.Frame(canvas, bg="#ECF0F1")

            scroll_frame.bind("<Configure>", 
                lambda e: canvas.configure(scrollregion=canvas.bbox("all")))

            canvas.create_window((0, 0), window=scroll_frame, anchor="nw")
            canvas.configure(yscrollcommand=scrollbar.set)

            # Привязка колеса мыши
            def on_mousewheel(event):
                canvas.yview_scroll(int(-1*(event.delta/120)), "units")
            canvas.bind_all("<MouseWheel>", on_mousewheel)

            canvas.pack(side="left", fill="both", expand=True)
            scrollbar.pack(side="right", fill="y")

            # Отображение товаров
            for product in products:
                self.create_product_card(scroll_frame, product)

        except mysql.connector.Error as err:
            messagebox.showerror("Ошибка", f"Не удалось загрузить товары: {err}")

    def create_product_card(self, parent, product):
        """Создание карточки товара"""
        discount = int(product.get('discount') or 0)
        count = int(product.get('count') or 0)
        price = float(product['price'])
        final_price = price * (1 - discount / 100) if discount else price

        # Определение цвета фона
        if count == 0:
            bg_color = "#E0E0E0"  # Серый - нет в наличии
            border_color = "#BDBDBD"
        elif discount > 15:
            bg_color = "#E8EAF6"  # Синий - большая скидка
            border_color = "#3F51B5"
        elif discount > 0:
            bg_color = "#FFF3E0"  # Оранжевый - есть скидка
            border_color = "#FF9800"
        else:
            bg_color = "#FFFFFF"  # Белый - обычный
            border_color = "#4CAF50"

        # Рамка карточки
        card_frame = tk.Frame(parent, bg=bg_color, relief="solid", 
                              bd=2, highlightbackground=border_color, 
                              highlightthickness=1)
        card_frame.pack(fill="x", padx=10, pady=5, ipady=10)

        # Левая часть - основная информация
        left_frame = tk.Frame(card_frame, bg=bg_color)
        left_frame.pack(side="left", fill="both", expand=True, padx=15, pady=10)

        # Категория и название
        category = product.get('category_name', 'Без категории')
        name = product.get('name', 'Без названия')
        tk.Label(left_frame, text=f"{category} | {name}",
                font=("Arial", 13, "bold"), bg=bg_color, 
                fg="#2C3E50", anchor="w").pack(fill="x")

        # Описание
        description = product.get('description', 'Нет описания')
        if len(description) > 100:
            description = description[:100] + "..."
        tk.Label(left_frame, text=f"📝 {description}",
                font=("Arial", 10), bg=bg_color, 
                fg="#546E7A", anchor="w", wraplength=500).pack(fill="x", pady=(5, 0))

        # Дополнительная информация
        info_text = f"🏭 {product.get('manufacturer', 'Не указан')} | "
        info_text += f"🚚 {product.get('supplier', 'Не указан')} | "
        info_text += f"📏 {product.get('unit', 'шт')}"
        tk.Label(left_frame, text=info_text,
                font=("Arial", 9), bg=bg_color, 
                fg="#78909C", anchor="w").pack(fill="x", pady=(5, 0))

        # Правая часть - цена и количество
        right_frame = tk.Frame(card_frame, bg=bg_color)
        right_frame.pack(side="right", padx=20, pady=10)

        # Цена со скидкой
        if discount > 0:
            tk.Label(right_frame, text=f"{price:.2f} руб.",
                    font=("Arial", 11, "overstrike"), 
                    bg=bg_color, fg="#E74C3C").pack(anchor="e")
            tk.Label(right_frame, text=f"{final_price:.2f} руб.",
                    font=("Arial", 16, "bold"), 
                    bg=bg_color, fg="#2ECC71").pack(anchor="e")
        else:
            tk.Label(right_frame, text=f"{price:.2f} руб.",
                    font=("Arial", 16, "bold"), 
                    bg=bg_color, fg="#2ECC71").pack(anchor="e")

        # Скидка
        if discount > 0:
            tk.Label(right_frame, text=f"Скидка {discount}%",
                    font=("Arial", 10, "bold"), 
                    bg="#FF5252", fg="white").pack(anchor="e", pady=(5, 5))
        else:
            tk.Label(right_frame, text="Без скидки",
                    font=("Arial", 10), 
                    bg=bg_color, fg="#90A4AE").pack(anchor="e", pady=(5, 5))

        # Наличие
        if count > 0:
            tk.Label(right_frame, text=f"✅ В наличии: {count} {product.get('unit', 'шт')}",
                    font=("Arial", 10, "bold"), 
                    bg="#C8E6C9", fg="#2E7D32", 
                    relief="solid", padx=10, pady=3).pack(anchor="e")
        else:
            tk.Label(right_frame, text="❌ Нет в наличии",
                    font=("Arial", 10, "bold"), 
                    bg="#FFCDD2", fg="#C62828", 
                    relief="solid", padx=10, pady=3).pack(anchor="e")

    def logout(self):
        """Выход из системы"""
        self.window.destroy()
        self.master.deiconify()

# ================== ОКНА ДЛЯ РАЗНЫХ РОЛЕЙ ==================
class AdminWindow(BaseWindow):
    def __init__(self, master, user):
        super().__init__(master, user)
        self.window.title("Панель администратора")
        self.create_admin_controls()

    def create_admin_controls(self):
        """Панель управления администратора"""
        controls_frame = tk.Frame(self.window, bg="#2C3E50")
        controls_frame.pack(fill="x", padx=20, pady=(10, 0))
        
        tk.Label(controls_frame, text="🔧 ПАНЕЛЬ УПРАВЛЕНИЯ", 
                font=("Arial", 12, "bold"), fg="white", bg="#2C3E50").pack(
                    side="left", padx=15, pady=10)
        
        buttons = [
            ("👥 Пользователи", "#3498DB"),
            ("📦 Товары", "#2ECC71"),
            ("📊 Отчёты", "#F39C12"),
            ("⚙️ Настройки", "#9B59B6")
        ]
        
        for text, color in buttons:
            tk.Button(controls_frame, text=text, font=("Arial", 10),
                     bg=color, fg="white", relief="flat", 
                     padx=15, pady=5, cursor="hand2").pack(
                         side="left", padx=5, pady=10)

class ManagerWindow(BaseWindow):
    def __init__(self, master, user):
        super().__init__(master, user)
        self.window.title("Панель менеджера")
        self.create_manager_controls()

    def create_manager_controls(self):
        """Панель управления менеджера"""
        controls_frame = tk.Frame(self.window, bg="#2C3E50")
        controls_frame.pack(fill="x", padx=20, pady=(10, 0))
        
        tk.Label(controls_frame, text="📋 УПРАВЛЕНИЕ ЗАКАЗАМИ", 
                font=("Arial", 12, "bold"), fg="white", bg="#2C3E50").pack(
                    side="left", padx=15, pady=10)
        
        buttons = [
            ("📝 Заказы", "#3498DB"),
            ("📦 Поставки", "#2ECC71"),
            ("👥 Клиенты", "#F39C12"),
            ("📊 Статистика", "#9B59B6")
        ]
        
        for text, color in buttons:
            tk.Button(controls_frame, text=text, font=("Arial", 10),
                     bg=color, fg="white", relief="flat", 
                     padx=15, pady=5, cursor="hand2").pack(
                         side="left", padx=5, pady=10)

class ClientWindow(BaseWindow):
    def __init__(self, master, user):
        super().__init__(master, user)
        self.window.title("Личный кабинет клиента")
        self.create_client_controls()

    def create_client_controls(self):
        """Панель клиента"""
        controls_frame = tk.Frame(self.window, bg="#2C3E50")
        controls_frame.pack(fill="x", padx=20, pady=(10, 0))
        
        tk.Label(controls_frame, text="🛒 МОИ ПОКУПКИ", 
                font=("Arial", 12, "bold"), fg="white", bg="#2C3E50").pack(
                    side="left", padx=15, pady=10)
        
        buttons = [
            ("📋 История заказов", "#3498DB"),
            ("🛒 Корзина", "#2ECC71"),
            ("❤️ Избранное", "#E91E63"),
            ("👤 Профиль", "#9B59B6")
        ]
        
        for text, color in buttons:
            tk.Button(controls_frame, text=text, font=("Arial", 10),
                     bg=color, fg="white", relief="flat", 
                     padx=15, pady=5, cursor="hand2").pack(
                         side="left", padx=5, pady=10)

class GuestWindow(BaseWindow):
    def __init__(self, master):
        super().__init__(master, None)
        self.window.title("Просмотр товаров (Гость)")
        
        # Дополнительное сообщение для гостей
        guest_frame = tk.Frame(self.window, bg="#FFF9C4")
        guest_frame.pack(fill="x", padx=20, pady=(10, 0))
        tk.Label(guest_frame, 
                text="⚠️ Вы просматриваете товары как гость. Для оформления заказа войдите в систему.",
                font=("Arial", 10), bg="#FFF9C4", fg="#F57F17", 
                wraplength=600).pack(pady=10, padx=15)

# ================== ЗАПУСК ПРИЛОЖЕНИЯ ==================
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

-- Добавление тестовых пользователей
INSERT INTO Users (login, password, role, full_name) VALUES
('admin', '123', 'admin', 'Иванов Иван Иванович'),
('manager', '123', 'manager', 'Петров Пётр Петрович'),
('manager2', '123', 'manager', 'Смирнова Елена Викторовна'),
('client', '123', 'client', 'Сидорова Анна Сергеевна'),
('client2', '123', 'client', 'Козлов Дмитрий Александрович');

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
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    FOREIGN KEY (category_id) REFERENCES Categories(id) ON DELETE SET NULL
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- ============================================
-- ЗАПОЛНЕНИЕ ТОВАРОВ
-- ============================================

-- Товары со скидкой >15% (синий фон в приложении)
INSERT INTO Products (name, category_id, description, manufacturer, supplier, price, unit, count, discount) VALUES
('Кроссовки Nike Air Max', 1, 'Спортивные кроссовки с амортизацией Air Max. Подходят для бега и повседневной носки.', 
 'Nike', 'ООО "СпортТорг"', 12999.00, 'пара', 25, 20),
 
('Зимняя куртка North Face', 2, 'Тёплая зимняя куртка с наполнителем из пуха. Водонепроницаемая ткань.', 
 'The North Face', 'ИП "Экипировка"', 25999.00, 'шт', 12, 25),
 
('Беспроводные наушники Sony', 5, 'Bluetooth наушники с шумоподавлением. 30 часов работы без подзарядки.', 
 'Sony', 'ООО "ТехноМир"', 15999.00, 'шт', 8, 18);

-- Товары с обычной скидкой или без скидки
INSERT INTO Products (name, category_id, description, manufacturer, supplier, price, unit, count, discount) VALUES
('Ботинки Timberland', 1, 'Классические кожаные ботинки. Износостойкая подошва, влагозащита.', 
 'Timberland', 'ИП "Петров А.В."', 18999.00, 'пара', 15, 5),
 
('Футболка Adidas', 2, 'Хлопковая футболка с логотипом. Дышащий материал.', 
 'Adidas', 'ООО "СпортТорг"', 2999.00, 'шт', 50, 0),
 
('Сандалии Crocs', 1, 'Летние сандалии из лёгкого материала. Анатомическая стелька.', 
 'Crocs', 'ООО "ЛетоСтиль"', 3999.00, 'пара', 30, 10),
 
('Рюкзак Herschel', 3, 'Городской рюкзак с отделением для ноутбука. Объём 25 литров.', 
 'Herschel Supply Co.', 'ООО "СтильМаркет"', 7999.00, 'шт', 20, 0);

-- Товары, которых нет на складе (серый фон)
INSERT INTO Products (name, category_id, description, manufacturer, supplier, price, unit, count, discount) VALUES
('Умные часы Apple Watch', 5, 'Смарт-часы с функцией ЭКГ и отслеживанием тренировок.', 
 'Apple', 'ООО "ТехноМир"', 45999.00, 'шт', 0, 5),
 
('Кожаный ремень Calvin Klein', 3, 'Классический кожаный ремень. Ширина 3.5 см, пряжка из нержавеющей стали.', 
 'Calvin Klein', 'ООО "МодаСтиль"', 4999.00, 'шт', 0, 0),
 
('Спортивный костюм Puma', 4, 'Костюм для тренировок из влагоотводящей ткани.', 
 'Puma', 'ООО "СпортТорг"', 8999.00, 'шт', 0, 15);

-- Товары без скидки с большим количеством
INSERT INTO Products (name, category_id, description, manufacturer, supplier, price, unit, count, discount) VALUES
('Носки спортивные (набор 3 пары)', 4, 'Носки из хлопка с усиленной пяткой и мысом.', 
 'Nike', 'ООО "СпортТорг"', 999.00, 'набор', 100, 0),
 
('Кепка бейсболка New Era', 3, 'Классическая бейсболка с регулируемым размером.', 
 'New Era', 'ООО "МодаСтиль"', 2499.00, 'шт', 45, 0),
 
('Шорты для бега Under Armour', 4, 'Лёгкие шорты с внутренними трусами. Быстросохнущий материал.', 
 'Under Armour', 'ООО "СпортТорг"', 3499.00, 'шт', 35, 0);

-- ============================================
-- ТАБЛИЦА ЗАКАЗОВ (для будущего расширения)
-- ============================================
CREATE TABLE Orders (
    id INT AUTO_INCREMENT PRIMARY KEY,
    user_id INT NOT NULL,
    order_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    status ENUM('new', 'processing', 'completed', 'cancelled') DEFAULT 'new',
    total_amount DECIMAL(10, 2) DEFAULT 0,
    FOREIGN KEY (user_id) REFERENCES Users(id) ON DELETE CASCADE
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- ============================================
-- ТАБЛИЦА ПОЗИЦИЙ ЗАКАЗА (для будущего расширения)
-- ============================================
CREATE TABLE OrderItems (
    id INT AUTO_INCREMENT PRIMARY KEY,
    order_id INT NOT NULL,
    product_id INT NOT NULL,
    quantity INT NOT NULL CHECK (quantity > 0),
    price DECIMAL(10, 2) NOT NULL,
    FOREIGN KEY (order_id) REFERENCES Orders(id) ON DELETE CASCADE,
    FOREIGN KEY (product_id) REFERENCES Products(id) ON DELETE CASCADE
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- ============================================
-- ПРОВЕРОЧНЫЕ ЗАПРОСЫ
-- ============================================

-- Показать всех пользователей
SELECT * FROM Users;

-- Показать товары с категориями и статусом
SELECT 
    p.name AS 'Название товара',
    c.name AS 'Категория',
    p.price AS 'Цена',
    p.count AS 'Остаток',
    p.discount AS 'Скидка %',
    CASE 
        WHEN p.count = 0 THEN '❌ Нет в наличии'
        WHEN p.discount > 15 THEN '🔥 Большая скидка'
        WHEN p.discount > 0 THEN '💰 Есть скидка'
        ELSE '✓ В наличии'
    END AS 'Статус'
FROM Products p
LEFT JOIN Categories c ON p.category_id = c.id
ORDER BY p.discount DESC, p.name;

-- Статистика по товарам
SELECT 
    COUNT(*) AS 'Всего товаров',
    SUM(count) AS 'Общий остаток',
    AVG(price) AS 'Средняя цена',
    COUNT(CASE WHEN discount > 15 THEN 1 END) AS 'Товаров с большой скидкой',
    COUNT(CASE WHEN count = 0 THEN 1 END) AS 'Товаров нет в наличии'
FROM Products;
    
