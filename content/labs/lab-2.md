## Тема, Мета, Місце розташування

**Тема:** Створення бази даних у MySQL. Підключення Node.js до MySQL. Використання ORM Sequelize

**Мета:** Метою роботи є набуття навичок створення бази даних у СУБД MySQL, виконання SQL-запитів, підключення Node.js до бази даних за допомогою бібліотеки mysql2, а також використання ORM Sequelize для роботи з базою даних та реалізації зв’язків між таблицями.


**Місце розташування:**
- GitHub: https://github.com/karolinarm08/Interior_RudykhKO
- Live demo: https://karolinarm08.github.io/Interior_RudykhKO/

---

## Завдання лабораторної роботи

1. Створити базу даних, яка буде використовуватись вашим власним вебдодатком (наприклад
web_backend_lab)
2. Створити таблиці у вашый базы даних (наприклад users та posts)
3. Виконати SQL-запити:
o SELECT
o INSERT
o UPDATE
o DELETE
4. Підключити Node.js до MySQL через пакет mysql2
5. Виконати SQL-запити з Node.js
6. Використати ORM Sequelize
7. Створити моделі User та Post
8. Реалізувати зв’язок One-to-Many

---

### 1. Створити базу даних, яка буде використовуватись вашим власним вебдодатком

На першому етапі було виконано встановлення СУБД MySQL Server 8.0 разом із додатковими компонентами, зокрема MySQL Workbench та MySQL Shell. 
![](/assets/labs/lab-2/1.png)

Після запуску MySQL Workbench було створено новий SQL-запит, у якому виконано команду створення бази даних для майбутнього інтернет-магазину. 
![](/assets/labs/lab-2/2.png)

Команда CREATE DATABASE IF NOT EXISTS використовується для створення бази даних interior_shop, якщо вона ще не існує в системі. Це дозволяє уникнути помилки при повторному запуску скрипта. Далі команда USE interior_shop встановлює цю базу даних як активну для подальшої роботи.
![](/assets/labs/lab-2/3.png)

---

### 2. Створити таблиці у вашій базі даних

Після створення бази даних interior_shop було виконано створення таблиць, необхідних для роботи інтернет-магазину товарів для інтер’єру. Структура бази даних була побудована з урахуванням основних сутностей предметної області: категорії товарів, товари, клієнти, замовлення та позиції замовлень.
Для цього було використано такі SQL-команди:

```bash
CREATE TABLE categories (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(100) NOT NULL
);

CREATE TABLE products (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(150) NOT NULL,
    description TEXT,
    price DECIMAL(10,2) NOT NULL,
    stock_status VARCHAR(50),
    category_id INT,
    FOREIGN KEY (category_id) REFERENCES categories(id)
);

CREATE TABLE clients (
    id INT AUTO_INCREMENT PRIMARY KEY,
    full_name VARCHAR(150) NOT NULL,
    email VARCHAR(100) NOT NULL,
    phone VARCHAR(30)
);

CREATE TABLE orders (
    id INT AUTO_INCREMENT PRIMARY KEY,
    client_id INT,
    order_date DATE,
    total_amount DECIMAL(10,2),
    FOREIGN KEY (client_id) REFERENCES clients(id)
);

CREATE TABLE order_items (
    id INT AUTO_INCREMENT PRIMARY KEY,
    order_id INT,
    product_id INT,
    quantity INT NOT NULL,
    price_at_order DECIMAL(10,2) NOT NULL,
    FOREIGN KEY (order_id) REFERENCES orders(id),
    FOREIGN KEY (product_id) REFERENCES products(id)
);
```

![](/assets/labs/lab-2/4.png)

У наведеній структурі таблиця categories призначена для зберігання категорій товарів. Таблиця products містить інформацію про товари магазину, їх опис, ціну, наявність та посилання на категорію. Таблиця clients використовується для збереження даних про клієнтів. Таблиця orders містить інформацію про оформлені замовлення, а order_items зберігає перелік товарів у кожному замовленні.

Між таблицями реалізовано зв’язки за допомогою зовнішніх ключів:
- products.category_id посилається на categories.id;
- orders.client_id посилається на clients.id;
- order_items.order_id посилається на orders.id;
- order_items.product_id посилається на products.id.

Після створення таблиць база даних була заповнена початковими тестовими даними. Для цього були використані команди вставки записів:
```bash
INSERT INTO categories (name) VALUES
('Декор'),
('Освітлення'),
('Текстиль');

INSERT INTO products (name, description, price, stock_status, category_id) VALUES
('Настільна лампа', 'Сучасна лампа для спальні', 1200.00, 'В наявності', 2),
('Декоративна ваза', 'Керамічна ваза для інтер’єру', 850.00, 'В наявності', 1),
('Плед', 'М’який плед у скандинавському стилі', 990.00, 'Немає в наявності', 3);

INSERT INTO clients (full_name, email, phone) VALUES
('Рудих Кароліна', 'karolina@example.com', '+380991112233');

INSERT INTO orders (client_id, order_date, total_amount) VALUES
(1, '2026-03-24', 2050.00);

INSERT INTO order_items (order_id, product_id, quantity, price_at_order) VALUES
(1, 1, 1, 1200.00),
(1, 2, 1, 850.00);
```

---

### 3. Виконати SQL-запити: SELECT, INSERT, UPDATE, DELETE

Після створення таблиць та заповнення їх початковими даними було виконано перевірку коректності внесеної інформації використовуючи SELECT
![](/assets/labs/lab-2/5.png)

Додамо нову категорію за допомогою INSERT
```bash
INSERT INTO categories (name) VALUES ('Оздоблення стін');
```

Перевіряємо
![](/assets/labs/lab-2/6.png)

Оновимо ціну продукту використовуючи UPDATE
```bash
UPDATE products
SET price = 1350.00
WHERE id = 1;
```

Перевіряємо. Так було до:
![](/assets/labs/lab-2/7.png)
А так стало після оновлення:
![](/assets/labs/lab-2/8.png)

Видалимо продукт використовуючи DELETE
```bash
DELETE FROM categories
WHERE name = 'Оздоблення стін';
```

Перевіряємо. Так було до:
![](/assets/labs/lab-2/9.png)
А так стало після оновлення:
![](/assets/labs/lab-2/10.png)

---

### 4. Підключити Node.js до MySQL через пакет mysql2

На наступному етапі роботи було виконано підготовку середовища Node.js для взаємодії з базою даних MySQL. Спочатку в терміналі було перевірено коректність встановлення Node.js та npm
![](/assets/labs/lab-2/11.png)

Після цього в папці проєкту було ініціалізовано Node.js-застосунок
![](/assets/labs/lab-2/12.png)

У результаті виконання цієї команди автоматично створився файл package.json, який містить основну інформацію про проєкт і використовується для керування залежностями.

Далі було встановлено необхідні пакети
![](/assets/labs/lab-2/13.png)

Після встановлення залежностей було створено файл db.js, у якому налаштовано параметри підключення до бази даних
![](/assets/labs/lab-2/15.png)

Для перевірки правильності підключення було створено файл test-db.js, у якому виконується підключення до бази даних та простий SQL-запит вибірки
![](/assets/labs/lab-2/14.png)

У результаті запуску команди
![](/assets/labs/lab-2/16.png)

було отримано успішне підключення до бази даних та виведено записи з таблиці products. Це підтвердило, що Node.js-застосунок коректно взаємодіє з MySQL через пакет mysql2

---

### 5. Виконати SQL-запити з Node.js

Після успішного підключення Node.js до MySQL було реалізовано виконання основних CRUD-операцій безпосередньо з програмного коду. Для цього було створено файл crud.js, у якому використано підключення до бази даних через раніше створений модуль db.js

```bash
const connectDB = require('./db');

async function runCRUD() {
  try {
    const db = await connectDB();

    const [products] = await db.execute('SELECT * FROM products');
    console.log('Товари:');
    console.log(products);

    await db.execute(
      'INSERT INTO categories (name) VALUES (?)',
      ['Меблі']
    );
    console.log('Категорію додано');

    await db.execute(
      'UPDATE products SET price = ? WHERE id = ?',
      [1500, 1]
    );
    console.log('Ціну товару оновлено');

    await db.execute(
      'DELETE FROM categories WHERE name = ?',
      ['Меблі']
    );
    console.log('Категорію видалено');

    await db.end();
    console.log('CRUD виконано успішно');
  } catch (error) {
    console.error('Помилка:', error.message);
  }
}

runCRUD();
```

Під час виконання цього скрипта в консолі було виведено поточний список товарів, після чого послідовно підтверджено додавання категорії, оновлення ціни товару, видалення категорії та успішне завершення всіх CRUD-операцій
![](/assets/labs/lab-2/17.png)

---

### 6. Використати ORM Sequelize. Створити моделі User та Post. Реалізувати зв’язок One-to-Many

На наступному етапі роботи для взаємодії з базою даних було використано ORM Sequelize. Використання ORM дозволяє працювати з таблицями бази даних як з об’єктами JavaScript, що значно спрощує створення, зміну, вибірку та видалення даних без постійного написання SQL-запитів вручну.

Для підключення Sequelize було створено файл config/database.js, у якому налаштовано параметри доступу до бази даних interior_shop
```bash
const { Sequelize } = require('sequelize');

const sequelize = new Sequelize(
  'interior_shop',
  'root',
  '********',
  {
    host: 'localhost',
    port: 3307,
    dialect: 'mysql'
  }
);

module.exports = sequelize;
```

Після цього були створені моделі, що відповідають таблицям бази даних.

Оскільки в даній лабораторній роботі розроблявся не блог, а інтернет-магазин товарів для інтер’єру, замість моделей User та Post були створені аналогічні за логікою моделі Category та Product

Модель Category описує таблицю categories

```bash
const { DataTypes } = require('sequelize');
const sequelize = require('../config/database');

const Category = sequelize.define('Category', {
  name: {
    type: DataTypes.STRING,
    allowNull: false
  }
}, {
  tableName: 'categories',
  timestamps: false
});

module.exports = Category;
```

Модель Product описує таблицю products

```bash
const { DataTypes } = require('sequelize');
const sequelize = require('../config/database');

const Product = sequelize.define('Product', {
  name: {
    type: DataTypes.STRING,
    allowNull: false
  },
  description: {
    type: DataTypes.TEXT
  },
  price: {
    type: DataTypes.DECIMAL(10, 2),
    allowNull: false
  },
  stock_status: {
    type: DataTypes.STRING
  }
}, {
  tableName: 'products',
  timestamps: false
});

module.exports = Product;
```

Для реалізації зв’язку один-до-багатьох між категоріями та товарами було використано файл models/index.js

```bash
const Category = require('./Category');
const Product = require('./Product');

Category.hasMany(Product, {
  foreignKey: 'category_id',
  as: 'products'
});

Product.belongsTo(Category, {
  foreignKey: 'category_id',
  as: 'category'
});

module.exports = { Category, Product };
```
У цьому випадку одна категорія може містити багато товарів, а кожен товар належить одній категорії

Для перевірки роботи ORM було створено демонстраційний файл sequelize-demo.js, у якому послідовно виконувались основні операції

```bash
const sequelize = require('./config/database');
const { Category, Product } = require('./models');

async function runSequelizeDemo() {
  try {
    await sequelize.authenticate();
    console.log('Підключення через Sequelize успішне');

    await sequelize.sync();
    console.log('Синхронізація виконана');

    const category = await Category.create({
      name: 'Тестова категорія Sequelize'
    });
    console.log('Категорію створено:', category.toJSON());

    const product = await Product.create({
      name: 'Тестовий товар Sequelize',
      description: 'Створено через ORM Sequelize',
      price: 1999.99,
      stock_status: 'В наявності',
      category_id: category.id
    });
    console.log('Товар створено:', product.toJSON());

    const products = await Product.findAll({
      include: {
        model: Category,
        as: 'category'
      }
    });

    console.log('Усі товари разом з категоріями:');
    console.log(JSON.stringify(products, null, 2));

    await Product.update(
      { price: 2199.99 },
      { where: { id: product.id } }
    );
    console.log('Ціну товару оновлено');

    const updatedProduct = await Product.findByPk(product.id);
    console.log('Оновлений товар:', updatedProduct.toJSON());

    await Product.destroy({
      where: { id: product.id }
    });
    console.log('Товар видалено');

    await Category.destroy({
      where: { id: category.id }
    });
    console.log('Категорію видалено');

    await sequelize.close();
    console.log('З’єднання закрито');
  } catch (error) {
    console.error('Помилка Sequelize:', error.message);
  }
}

runSequelizeDemo();
```

Запускаємо
![](/assets/labs/lab-2/18.png)
![](/assets/labs/lab-2/19.png)
![](/assets/labs/lab-2/20.png)


У результаті виконання скрипта було отримано повідомлення про успішне підключення, створення записів, оновлення даних, видалення записів та завершення роботи. Також у консолі було виведено список товарів разом із пов’язаними категоріями, що підтвердило правильність налаштування зв’язку між моделями.

---

## Висновки

У ході виконання лабораторної роботи було створено базу даних для інтернет-магазину, реалізовано таблиці та зв’язки між ними. Було виконано основні SQL-операції, а також реалізовано підключення Node.js до MySQL за допомогою бібліотеки mysql2. Додатково було використано ORM Sequelize для роботи з базою даних через моделі та реалізації зв’язків типу один-до-багатьох. Отримані знання дозволяють створювати серверну частину веб-застосунків із використанням сучасних технологій.
