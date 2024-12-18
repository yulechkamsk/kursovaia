# kursovaia
from PyQt5.QtWidgets import (
    QApplication, QMainWindow, QWidget, QVBoxLayout, QHBoxLayout, QLabel,
    QLineEdit, QPushButton, QMessageBox, QFormLayout, QTableWidget,
    QTableWidgetItem, QListWidget, QComboBox
)

import mysql.connector
import sys
import logging

# Настройки подключения к MySQL
db_config = {
    'host': 'localhost',
    'user': 'root',
    'password': 'root',
    'database': 'shop'
}

logging.basicConfig(filename='app.log', level=logging.DEBUG)

class AuthWindow(QMainWindow):
    def __init__(self):
        super().__init__()
        self.register_window = None
        self.user_system_window = None
        self.manager_system_window = None
        self.setWindowTitle("Интернет-магазин - Вход")
        self.setGeometry(300, 100, 400, 300)

        widget = QWidget()
        self.setCentralWidget(widget)
        layout = QFormLayout()

        self.login_input = QLineEdit()
        self.login_input.setPlaceholderText("Введите логин")
        layout.addRow(QLabel("Логин:"), self.login_input)

        self.password_input = QLineEdit()
        self.password_input.setPlaceholderText("Введите пароль")
        self.password_input.setEchoMode(QLineEdit.Password)
        layout.addRow(QLabel("Пароль:"), self.password_input)

        # Выпадающий список для выбора роли пользователя
        self.role_combo = QComboBox()
        self.role_combo.addItems(["Пользователь", "Менеджер"])  # Добавляем два варианта
        layout.addRow(QLabel("Роль:"), self.role_combo)

        # Кнопка "Войти"
        self.login_button = QPushButton("Войти")
        self.login_button.clicked.connect(self.authenticate)
        layout.addWidget(self.login_button)

        self.register_button = QPushButton("Регистрация пользователя")
        self.register_button.clicked.connect(self.open_register_window)
        layout.addWidget(self.register_button)

        widget.setLayout(layout)

    def authenticate(self):
        login = self.login_input.text()
        password = self.password_input.text()
        role = self.role_combo.currentText()  # Получаем выбранную роль

        if role == "Менеджер":
            self.authenticate_manager(login, password)
        else:
            self.authenticate_user(login, password)

    def authenticate_manager(self, login, password):
        """Аутентификация менеджера"""
        try:
            conn = mysql.connector.connect(**db_config)
            cursor = conn.cursor()

            query = """SELECT manager_id FROM managers WHERE login=%s AND passwords=%s"""
            cursor.execute(query, (login, password))
            user = cursor.fetchone()

            cursor.close()
            conn.close()

            if user:
                self.open_manager_system()
            else:
                QMessageBox.warning(self, "Ошибка", "Неверный логин или пароль менеджера")

        except mysql.connector.Error as err:
            QMessageBox.critical(self, "Ошибка подключения", f"Ошибка: {err}")

    def authenticate_user(self, login, password):
        """Аутентификация пользователя"""
        try:
            conn = mysql.connector.connect(**db_config)
            cursor = conn.cursor()

            query = """SELECT user_id FROM users WHERE username=%s AND password_hash=%s"""
            cursor.execute(query, (login, password))
            user = cursor.fetchone()
            cursor.close()
            conn.close()

            if user:
                self.open_user_system(user[0])  # Переход к пользовательской системе
            else:
                QMessageBox.warning(self, "Ошибка", "Неверный логин или пароль пользователя")

        except mysql.connector.Error as err:
            QMessageBox.critical(self, "Ошибка подключения", f"Ошибка: {err}")

    def open_manager_system(self):
        """Открытие окна менеджера"""
        self.manager_system_window = ManagerSystemWindow()
        self.manager_system_window.show()
        self.close()

    def open_user_system(self, user_id):
        """Открытие окна пользователя, где пользователь может выбрать товары и работать с корзиной"""
        self.user_system_window = UserSystemWindow(user_id)  # Передаем user_id для дальнейшей работы
        self.user_system_window.show()
        self.close()

    def open_register_window(self):
        """Открытие окна регистрации"""
        self.register_window = RegisterWindow()
        self.register_window.show()

class UserSystemWindow(QWidget):
    def __init__(self, user_id):
        super().__init__()
        self.user_id = user_id
        self.setWindowTitle("Интернет-магазин - Пользователь")
        self.setGeometry(300, 100, 800, 600)

        # Здесь будет логика отображения товаров и корзины
        self.layout = QVBoxLayout()

        self.product_list = QTableWidget()  # Таблица с товарами
        self.product_list.setColumnCount(4)  # ID товара, Название, Цена, Количество
        self.product_list.setHorizontalHeaderLabels(["ID товара", "Название", "Цена", "Количество"])
        self.layout.addWidget(self.product_list)

        self.load_products()  # Загрузка списка товаров

        self.setLayout(self.layout)

    def load_products(self):
        """Загрузка списка товаров для пользователя"""
        try:
            conn = mysql.connector.connect(**db_config)
            cursor = conn.cursor()

            query = """SELECT product_id, name, price, stock FROM products"""
            cursor.execute(query)
            products = cursor.fetchall()

            # Очистка таблицы перед добавлением новых данных
            self.product_list.setRowCount(0)

            # Заполнение таблицы товарами
            for row_number, product in enumerate(products):
                self.product_list.insertRow(row_number)
                for col_number, value in enumerate(product):
                    self.product_list.setItem(row_number, col_number, QTableWidgetItem(str(value)))

            cursor.close()
            conn.close()

        except mysql.connector.Error as err:
            QMessageBox.critical(self, "Ошибка загрузки товаров", f"Ошибка: {err}")


class RegisterWindow(QWidget):
    def __init__(self):
        super().__init__()
        self.setWindowTitle("Регистрация пользователя")
        self.setGeometry(400, 150, 400, 300)

        layout = QFormLayout()

        self.username_input = QLineEdit()
        self.username_input.setPlaceholderText("Введите имя пользователя")
        layout.addRow(QLabel("Имя пользователя:"), self.username_input)

        self.password_input = QLineEdit()
        self.password_input.setPlaceholderText("Введите пароль")
        self.password_input.setEchoMode(QLineEdit.Password)
        layout.addRow(QLabel("Пароль:"), self.password_input)

        self.email_input = QLineEdit()
        self.email_input.setPlaceholderText("Введите email")
        layout.addRow(QLabel("Email:"), self.email_input)

        self.register_button = QPushButton("Зарегистрироваться")
        self.register_button.clicked.connect(self.register_user)
        layout.addWidget(self.register_button)

        self.setLayout(layout)

    def register_user(self):
        username = self.username_input.text()
        password = self.password_input.text()
        email = self.email_input.text()

        try:
            conn = mysql.connector.connect(**db_config)
            cursor = conn.cursor()

            check_query = """SELECT user_id FROM users WHERE username = %s"""
            cursor.execute(check_query, (username,))
            existing_user = cursor.fetchone()

            if existing_user:
                QMessageBox.warning(self, "Ошибка", "Пользователь с таким именем уже существует.")
                cursor.close()
                conn.close()
                return

            cursor.close()
            conn.close()
        except mysql.connector.Error as err:
            QMessageBox.critical(self, "Ошибка подключения", f"Ошибка: {err}")
            return

        if not username or not password or not email:
            QMessageBox.warning(self, "Ошибка", "Все поля должны быть заполнены")
            return

        try:
            conn = mysql.connector.connect(**db_config)
            cursor = conn.cursor()

            query = """INSERT INTO users (username, password_hash, email) VALUES (%s, %s, %s)"""
            cursor.execute(query, (username, password, email))
            conn.commit()

            cursor.close()
            conn.close()

            QMessageBox.information(self, "Успех", "Пользователь успешно зарегистрирован!")
            self.close()

        except mysql.connector.Error as err:
            QMessageBox.critical(self, "Ошибка подключения", f"Ошибка: {err}")


class UserSystemWindow(QWidget):
    def __init__(self, user_id, manager_window=None):
        super().__init__()
        self.cart_window = None
        self.catalog_window = None
        self.auth_window = None
        self.user_id = user_id
        self.manager_window = manager_window
        self.setWindowTitle("Система пользователя - Интернет-магазин")
        self.setGeometry(300, 100, 800, 600)

        layout = QVBoxLayout()

        self.catalog_button = QPushButton("Просмотр каталога товаров")
        self.catalog_button.clicked.connect(self.open_product_catalog)
        layout.addWidget(self.catalog_button)

        self.view_cart_button = QPushButton("Просмотр корзины")
        self.view_cart_button.clicked.connect(self.view_cart)
        layout.addWidget(self.view_cart_button)

        self.logout_button = QPushButton("Выйти")
        self.logout_button.clicked.connect(self.logout)
        layout.addWidget(self.logout_button)

        self.setLayout(layout)

    def open_product_catalog(self):
        self.catalog_window = ProductCatalogWindow(self.user_id, self.manager_window)
        self.catalog_window.show()

    def view_cart(self):
        self.cart_window = CartWindow(self.user_id)
        self.cart_window.show()

    def logout(self):
        self.auth_window = AuthWindow()
        self.auth_window.show()
        self.close()


class ManagerSystemWindow(QWidget):
    def __init__(self):
        super().__init__()
        self.setWindowTitle("Система учёта заказов - Интернет-магазин")
        self.setGeometry(300, 100, 800, 600)

        layout = QVBoxLayout()

        # Таблица для отображения товаров из корзины
        self.orders_table = QTableWidget()
        self.orders_table.setColumnCount(6)  # Колонки: ID товара, Название товара, Имя пользователя, Цена, Количество, Статус
        self.orders_table.setHorizontalHeaderLabels(
            ["ID товара", "Название товара", "Имя пользователя", "Цена", "Количество", "Статус"]
        )
        layout.addWidget(self.orders_table)

        # Кнопка для обновления данных
        self.refresh_button = QPushButton("Обновить заказы")
        self.refresh_button.clicked.connect(self.load_orders)
        layout.addWidget(self.refresh_button)

        self.setLayout(layout)
        self.load_orders()  # Изначальная загрузка данных

        # Кнопка "Назад"
        self.back_button = QPushButton("Назад")
        self.back_button.clicked.connect(self.go_back_to_auth)
        layout.addWidget(self.back_button)

        self.setLayout(layout)
        self.load_orders()  # Изначальная загрузка данных

    def go_back_to_auth(self):
        """Возвращает в окно входа (AuthWindow)"""
        self.close()  # Закрыть текущее окно

        # Создаем и показываем окно входа
        self.auth_window = AuthWindow()
        self.auth_window.show()

    def load_orders(self):
        """Загрузка товаров из корзины пользователей"""
        try:
            conn = mysql.connector.connect(**db_config)
            cursor = conn.cursor()

            #  SQL-запрос для получения данных
            query = """
            SELECT 
                p.product_id AS product_id,
                p.name AS product_name, 
                u.username AS user_name, 
                p.price, 
                c.quantity, 
                c.status 
            FROM 
                cart c
            JOIN 
                users u ON c.user_id = u.user_id
            JOIN 
                products p ON c.product_id = p.product_id
            """
            cursor.execute(query)
            cart_items = cursor.fetchall()

            # Очистка таблицы перед обновлением
            self.orders_table.setRowCount(0)

            # Заполнение таблицы
            for row_number, item in enumerate(cart_items):
                self.orders_table.insertRow(row_number)

                # Заполняем таблицу по правильному порядку колонок
                self.orders_table.setItem(row_number, 0, QTableWidgetItem(str(item[0])))  # ID товара
                self.orders_table.setItem(row_number, 1, QTableWidgetItem(str(item[1])))  # Название товара
                self.orders_table.setItem(row_number, 2, QTableWidgetItem(str(item[2])))  # Имя пользователя
                self.orders_table.setItem(row_number, 3, QTableWidgetItem(str(item[3])))  # Цена
                self.orders_table.setItem(row_number, 4, QTableWidgetItem(str(item[4])))  # Количество

                # Добавляем QComboBox для колонки "Статус"
                status_combobox = QComboBox()
                status_combobox.addItems(["pending", "shipped", "delivered", "cancelled"])  # Возможные статусы
                current_status = item[5] or "pending"  # Текущий статус
                status_combobox.setCurrentText(current_status)

                # Сохраняем изменения статуса
                status_combobox.currentTextChanged.connect(
                    lambda status, product_id=item[0]: self.update_cart_status(product_id, status)
                )

                self.orders_table.setCellWidget(row_number, 5, status_combobox)  # Колонка "Статус"

            cursor.close()
            conn.close()

        except mysql.connector.Error as err:
            QMessageBox.critical(self, "Ошибка подключения", f"Ошибка: {err}")

    def update_cart_status(self, product_id, new_status):
        """Обновляет статус товара в корзине"""
        try:
            conn = mysql.connector.connect(**db_config)
            cursor = conn.cursor()

            query = """
            UPDATE cart
            SET status = %s
            WHERE product_id = %s
            """
            cursor.execute(query, (new_status, product_id))
            conn.commit()

            QMessageBox.information(self, "Статус обновлён", f"Статус товара с ID {product_id} изменён на {new_status}.")
            cursor.close()
            conn.close()

        except mysql.connector.Error as err:
            QMessageBox.critical(self, "Ошибка обновления", f"Ошибка: {err}")

class ProductCatalogWindow(QWidget):
    def __init__(self, user_id, manager_window=None):
        super().__init__()
        self.user_id = user_id
        self.manager_system_window = manager_window  # Сохраняем переданный параметр
        self.setWindowTitle("Каталог товаров")
        self.setGeometry(400, 150, 600, 400)

        layout = QVBoxLayout()

        self.product_list = QListWidget()
        layout.addWidget(self.product_list)

        quantity_layout = QHBoxLayout()
        self.quantity_input = QLineEdit()
        self.quantity_input.setPlaceholderText("Количество")
        quantity_layout.addWidget(QLabel("Количество:"))
        quantity_layout.addWidget(self.quantity_input)
        layout.addLayout(quantity_layout)

        self.add_to_cart_button = QPushButton("Добавить в корзину")
        self.add_to_cart_button.clicked.connect(self.add_to_cart)
        layout.addWidget(self.add_to_cart_button)

        self.setLayout(layout)
        self.load_products()

    def load_products(self):
        try:
            conn = mysql.connector.connect(**db_config)
            cursor = conn.cursor()

            query = """SELECT product_id, name, price FROM products"""
            cursor.execute(query)
            products = cursor.fetchall()

            cursor.close()
            conn.close()

            for product in products:
                self.product_list.addItem(f"{product[0]}: {product[1]} - {product[2]} руб.")

        except mysql.connector.Error as err:
            QMessageBox.critical(self, "Ошибка подключения", f"Ошибка: {err}")

    def add_to_cart(self):
        selected_item = self.product_list.currentItem()
        quantity = self.quantity_input.text()

        if not selected_item or not quantity.isdigit() or int(quantity) <= 0:
            QMessageBox.warning(self, "Ошибка", "Выберите товар и введите корректное количество")
            return

        product_id = int(selected_item.text().split(":")[0])
        quantity = int(quantity)

        try:
            conn = mysql.connector.connect(**db_config)
            cursor = conn.cursor()

            # Проверяем, есть ли товар уже в корзине
            check_query = """SELECT quantity FROM cart WHERE user_id = %s AND product_id = %s"""
            cursor.execute(check_query, (self.user_id, product_id))
            existing_item = cursor.fetchone()

            if existing_item:
                update_query = """UPDATE cart SET quantity = quantity + %s WHERE user_id = %s AND product_id = %s"""
                cursor.execute(update_query, (quantity, self.user_id, product_id))
            else:
                insert_query = """INSERT INTO cart (user_id, product_id, quantity) VALUES (%s, %s, %s)"""
                cursor.execute(insert_query, (self.user_id, product_id, quantity))

            conn.commit()
            cursor.close()
            conn.close()

            QMessageBox.information(self, "Успех", "Товар добавлен в корзину!")

            # Обновляем систему менеджера, если она открыта
            if self.manager_system_window:
                self.manager_system_window.load_orders()  # Обновляем таблицу заказов в менеджерской системе

        except mysql.connector.Error as err:
            QMessageBox.critical(self, "Ошибка подключения", f"Ошибка: {err}")
        except Exception as e:
            QMessageBox.critical(self, "Неизвестная ошибка", f"Ошибка: {e}")


class CartWindow(QWidget):
    def __init__(self, user_id):
        super().__init__()
        self.user_id = user_id
        self.setWindowTitle("Корзина")
        self.setGeometry(400, 150, 600, 400)

        layout = QVBoxLayout()

        self.cart_table = QTableWidget()
        self.cart_table.setColumnCount(4)
        self.cart_table.setHorizontalHeaderLabels(["ID Товара", "Название", "Цена", "Количество"])
        layout.addWidget(self.cart_table)

        self.remove_button = QPushButton("Удалить выбранный товар")
        self.remove_button.clicked.connect(self.remove_item)
        layout.addWidget(self.remove_button)

        self.setLayout(layout)
        self.load_cart()

    def load_cart(self):
        try:
            conn = mysql.connector.connect(**db_config)
            cursor = conn.cursor()

            query = """
            SELECT c.product_id, p.name, p.price, c.quantity
            FROM cart c
            JOIN products p ON c.product_id = p.product_id
            WHERE c.user_id = %s
            """
            cursor.execute(query, (self.user_id,))
            cart_items = cursor.fetchall()

            if not cart_items:
                QMessageBox.information(self, "Корзина пуста", "Ваша корзина пуста.")
                return

            self.cart_table.setRowCount(0)
            for row_number, item in enumerate(cart_items):
                self.cart_table.insertRow(row_number)
                for column_number, data in enumerate(item):
                    self.cart_table.setItem(row_number, column_number, QTableWidgetItem(str(data)))

            cursor.close()
            conn.close()

        except mysql.connector.Error as err:
            logging.error(f"Ошибка при загрузке корзины: {err}")
            QMessageBox.critical(self, "Ошибка подключения", f"Ошибка: {err}")

        except Exception as e:
            logging.error(f"Необработанная ошибка при загрузке корзины: {e}")
            QMessageBox.critical(self, "Неизвестная ошибка", f"Ошибка: {e}")

    def remove_item(self):
        selected_row = self.cart_table.currentRow()
        if selected_row < 0:
            QMessageBox.warning(self, "Ошибка", "Выберите товар для удаления")
            return

        product_id = self.cart_table.item(selected_row, 0).text()

        try:
            conn = mysql.connector.connect(**db_config)
            cursor = conn.cursor()

            delete_query = """DELETE FROM cart WHERE user_id = %s AND product_id = %s"""
            cursor.execute(delete_query, (self.user_id, product_id))
            conn.commit()

            cursor.close()
            conn.close()

            QMessageBox.information(self, "Успех", "Товар удалён из корзины!")
            self.load_cart()

        except mysql.connector.Error as err:
            QMessageBox.critical(self, "Ошибка подключения", f"Ошибка: {err}")

    def load_cart(self):
        try:
            conn = mysql.connector.connect(**db_config)
            cursor = conn.cursor()

            query = """
            SELECT c.product_id, p.name, p.price, c.quantity
            FROM cart c
            JOIN products p ON c.product_id = p.product_id
            WHERE c.user_id = %s
            """
            cursor.execute(query, (self.user_id,))
            cart_items = cursor.fetchall()

            if not cart_items:
                QMessageBox.information(self, "Корзина пуста", "Ваша корзина пуста.")
                return

            self.cart_table.setRowCount(0)
            for row_number, item in enumerate(cart_items):
                self.cart_table.insertRow(row_number)
                for column_number, data in enumerate(item):
                    self.cart_table.setItem(row_number, column_number, QTableWidgetItem(str(data)))

            cursor.close()
            conn.close()

        except mysql.connector.Error as err:
            logging.error(f"Ошибка при загрузке корзины: {err}")
            QMessageBox.critical(self, "Ошибка подключения", f"Ошибка: {err}")

        except Exception as e:
            logging.error(f"Необработанная ошибка при загрузке корзины: {e}")
            QMessageBox.critical(self, "Неизвестная ошибка", f"Ошибка: {e}")

    def create_order(self):
        """Оформление заказа"""
        try:
            conn = mysql.connector.connect(**db_config)
            cursor = conn.cursor()

            # Получаем адрес пользователя
            cursor.execute("""
                SELECT address_id 
                FROM addresses 
                WHERE user_id = %s 
                LIMIT 1
            """, (self.user_id,))
            address = cursor.fetchone()

            if not address:
                QMessageBox.warning(self, "Ошибка", "У вас нет адреса для доставки. Пожалуйста, добавьте адрес.")
                return

            address_id = address[0]  # Получаем address_id из результата запроса

            # Получение всех товаров из корзины
            cursor.execute("SELECT product_id, quantity FROM cart WHERE user_id = %s", (self.user_id,))
            cart_items = cursor.fetchall()

            if not cart_items:
                QMessageBox.warning(self, "Корзина пуста", "Ваша корзина пуста!")
                return

            # Суммируем стоимость товаров
            total_amount = 0
            for item in cart_items:
                product_id, quantity = item
                cursor.execute("SELECT price FROM products WHERE product_id = %s", (product_id,))
                product = cursor.fetchone()
                if product:
                    total_amount += product[0] * quantity

            # Создаем новый заказ
            cursor.execute("""
                INSERT INTO orders (user_id, address_id, order_date, total_amount, status)
                VALUES (%s, %s, NOW(), %s, 'pending')
            """, (self.user_id, address_id, total_amount))
            order_id = cursor.lastrowid  # Получаем ID нового заказа

            # Перемещаем товары из корзины в заказ
            for item in cart_items:
                product_id, quantity = item
                cursor.execute("""
                    INSERT INTO order_items (order_id, product_id, quantity)
                    VALUES (%s, %s, %s)
                """, (order_id, product_id, quantity))

            # Удаляем товары из корзины
            cursor.execute("DELETE FROM cart WHERE user_id = %s", (self.user_id,))

            conn.commit()

            # Отладочный вывод, чтобы убедиться, что заказ добавлен
            print(f"Order ID {order_id} added successfully.")

            # Обновляем окно менеджера
            if self.manager_window:
                self.manager_window.load_orders()  # Обновляем список заказов в менеджерской системе

            QMessageBox.information(self, "Успех", "Ваш заказ оформлен!")
            self.close()

        except mysql.connector.Error as err:
            QMessageBox.critical(self, "Ошибка", f"Ошибка при оформлении заказа: {err}")
        finally:
            cursor.close()
            conn.close()


def main():
    app = QApplication(sys.argv)
    window = AuthWindow()
    window.show()
    sys.exit(app.exec_())


if __name__ == "__main__":
    main()
