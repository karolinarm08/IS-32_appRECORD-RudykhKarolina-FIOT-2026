## Тема, Мета, Місце розташування

**Тема:** «Документування API за допомогою Swagger Деплой Node.js-додатку. Підсумковий проєкт: REST API з MySQL»


**Мета:** 
- розуміти принципи документування REST API;
- навчитися використовувати Swagger/OpenAPI для автоматичної документації;
- навчитися інтегрувати Swagger у Node.js + Express застосунок;
- створювати опис endpoint-ів, параметрів та моделей даних;
- виконувати деплой Node.js-додатку на хмарний сервіс;
- тестувати API через Swagger UI.

**Місце розташування:**
- GitHub: https://github.com/karolinarm08/Interior_RudykhKO
- Live demo: https://karolinarm08.github.io/Interior_RudykhKO/

---

## Завдання лабораторної роботи

Завдання 1. Створити REST API на Node.js + Express.

Завдання 2. Підключити MySQL. Реалізувати CRUD-операції.

Завдання 3. Інтегрувати Swagger UI у проєкт.

Завдання 4. Задокументувати всі endpoint-и

Завдання 5. Виконати тестування API через Swagger UI

Примітка: Можна застосувати REST API ,підключену БД та CRUD-операції з попередніх лабораторних робіт

Завдання 6. Продемонструвати підсумковий проєкт: REST API з MySQL та Swagger документацією.

---
### 3. Інтегрувати Swagger UI у проєкт.

Оновлено конфігурацію Swagger (додано підтримку JWT-токенів)
```bash
const swaggerOptions = {
  definition: {
    openapi: '3.0.0',
    info: {
      title: 'Interior API',
      version: '1.0.0',
      description: 'Документація REST API для інтернет-магазину інтер’єру'
    },
    servers: [
      {
        url: `http://localhost:${PORT}`,
        description: 'Локальний сервер'
      }
    ],
    // Додаємо конфігурацію для Bearer JWT авторизації
    components: {
      securitySchemes: {
        bearerAuth: {
          type: 'http',
          scheme: 'bearer',
          bearerFormat: 'JWT',
          description: 'Введіть JWT-токен доступу (AccessToken), отриманий при логіні'
        }
      }
    }
  },
  apis: ['./app.js']
};

const swaggerDocs = swaggerJsdoc(swaggerOptions);

app.use(
  '/api-docs',
  swaggerUi.serve,
  swaggerUi.setup(swaggerDocs)
);
```

### 4. Задокументувати всі endpoint-и

Оновлено коментарі JSDoc до основних маршрутів

1. Авторизація та вхід
```bash
/**
 * @swagger
 * /api/auth/register:
 *   post:
 *     summary: Реєстрація нового користувача
 *     tags: [Auth]
 *     requestBody:
 *       required: true
 *       content:
 *         application/json:
 *           schema:
 *             type: object
 *             required: [name, email, password, confirmPassword]
 *             properties:
 *               name:
 *                 type: string
 *                 example: "Name"
 *               email:
 *                 type: string
 *                 example: "UserName@example.com"
 *               password:
 *                 type: string
 *                 example: "StrongPass123!"
 *               confirmPassword:
 *                 type: string
 *                 example: "StrongPass123!"
 *     responses:
 *       201:
 *         description: Користувача успішно зареєстровано
 *       400:
 *         description: Помилка валідації або email вже існує
 */

 /**
 * @swagger
 * /api/auth/login:
 *   post:
 *     summary: Вхід користувача в систему
 *     tags: [Auth]
 *     requestBody:
 *       required: true
 *       content:
 *         application/json:
 *           schema:
 *             type: object
 *             required: [email, password]
 *             properties:
 *               email:
 *                 type: string
 *                 example: "UserName@example.com"
 *               password:
 *                 type: string
 *                 example: "StrongPass123!"
 *     responses:
 *       200:
 *         description: Успішний вхід, повертає токени
 *       400:
 *         description: Невірний email або пароль
 *       403:
 *         description: Email не підтверджено
 */
```

2. Список всіх користувачів
```bash
/**
 * @swagger
 * /api/admin/users:
 *   get:
 *     summary: Отримати список користувачів (для адміністраторів)
 *     tags: [Admin]
 *     security:
 *       - bearerAuth: []
 *     responses:
 *       200:
 *         description: Список користувачів
 *         content:
 *           application/json:
 *             schema:
 *               type: array
 *               items:
 *                 type: object
 *                 properties:
 *                   id:
 *                     type: integer
 *                   name:
 *                     type: string
 *                   email:
 *                     type: string
 *                   role:
 *                     type: string
 *                   is_email_confirmed:
 *                     type: boolean
 *                   created_at:
 *                     type: string
 *                   updated_at:
 *                     type: string
 *       401:
 *         description: Неавторизований - потрібен токен
 *         content:
 *           application/json:
 *             schema:
 *               type: object
 *               properties:
 *                 message:
 *                   type: string
 *                   example: "Немає токена доступу"
 *       403:
 *         description: Недостатньо прав - потрібна роль admin
 *         content:
 *           application/json:
 *             schema:
 *               type: object
 *               properties:
 *                 message:
 *                   type: string
 *                   example: "Недостатньо прав"
 */
 ```

3. CRUD для Категорій
```bash
/**
 * @swagger
 * /api/categories:
 *   post:
 *     summary: Створити нову категорію (Лише для Адміна)
 *     tags: [Categories]
 *     security:
 *       - bearerAuth: []
 *     requestBody:
 *       required: true
 *       content:
 *         application/json:
 *           schema:
 *             type: object
 *             required: [name]
 *             properties:
 *               name:
 *                 type: string
 *                 example: "Освітлення"
 *     responses:
 *       201:
 *         description: Категорію успішно створено
 *       400:
 *         description: Категорія з такою назвою вже існує
 *       401:
 *         description: Неавторизований запит
 *       403:
 *         description: Недостатньо прав (не адмін)
 */

 /**
 * @swagger
 * /api/categories/{id}:
 *   get:
 *     summary: Отримати категорію за ID
 *     tags: [Categories]
 *     parameters:
 *       - in: path
 *         name: id
 *         required: true
 *         schema:
 *           type: integer
 *         description: ID категорії
 *     responses:
 *       200:
 *         description: Дані категорії
 *       404:
 *         description: Категорію не знайдено
 */

 /**
 * @swagger
 * /api/categories/{id}:
 *   put:
 *     summary: Оновити категорію (Лише для Адміна)
 *     tags: [Categories]
 *     security:
 *       - bearerAuth: []
 *     parameters:
 *       - in: path
 *         name: id
 *         required: true
 *         schema:
 *           type: integer
 *         description: ID категорії
 *     requestBody:
 *       required: true
 *       content:
 *         application/json:
 *           schema:
 *             type: object
 *             required: [name]
 *             properties:
 *               name:
 *                 type: string
 *                 example: "Нова назва"
 *     responses:
 *       200:
 *         description: Категорію оновлено
 *       404:
 *         description: Категорію не знайдено
 */

 /**
 * @swagger
 * /api/categories/{id}:
 *   delete:
 *     summary: Видалити категорію за ID (Лише для Адміна)
 *     tags: [Categories]
 *     security:
 *       - bearerAuth: []
 *     parameters:
 *       - in: path
 *         name: id
 *         required: true
 *         schema:
 *           type: integer
 *         description: ID категорії
 *     responses:
 *       200:
 *         description: Категорію видалено
 *       400:
 *         description: Категорія містить товари
 *       404:
 *         description: Категорію не знайдено
 */
```
4. CRUD для Товарів
```bash
/**
 * @swagger
 * /api/products:
 *   get:
 *     summary: Отримати список товарів (з пагінацією та фільтрацією)
 *     tags: [Products]
 *     parameters:
 *       - in: query
 *         name: category
 *         schema:
 *           type: integer
 *         description: Фільтр за ID категорії
 *       - in: query
 *         name: page
 *         schema:
 *           type: integer
 *           default: 1
 *         description: Номер сторінки
 *       - in: query
 *         name: limit
 *         schema:
 *           type: integer
 *           default: 10
 *         description: Кількість товарів на сторінці
 *     responses:
 *       200:
 *         description: Список товарів
 */

 /**
 * @swagger
 * /api/products/{id}:
 *   get:
 *     summary: Отримати товар за ID
 *     tags: [Products]
 *     parameters:
 *       - in: path
 *         name: id
 *         required: true
 *         schema:
 *           type: integer
 *         description: ID товару
 *     responses:
 *       200:
 *         description: Дані товару
 *       404:
 *         description: Товар не знайдено
 */

 /**
 * @swagger
 * /api/products:
 *   post:
 *     summary: Додати новий товар (Лише для Адміна)
 *     tags: [Products]
 *     security:
 *       - bearerAuth: []
 *     requestBody:
 *       required: true
 *       content:
 *         application/json:
 *           schema:
 *             type: object
 *             required: [name, price, category_id]
 *             properties:
 *               name:
 *                 type: string
 *                 example: "Стильне крісло"
 *               price:
 *                 type: number
 *                 example: 3500
 *               category_id:
 *                 type: integer
 *                 example: 1
 *               description:
 *                 type: string
 *                 example: "М'яке крісло у скандинавському стилі"
 *               stock_status:
 *                 type: string
 *                 example: "В наявності"
 *     responses:
 *       200:
 *         description: Товар успішно створено
 *       400:
 *         description: Заповнені не всі обов'язкові поля
 */

 /**
 * @swagger
 * /api/products/{id}:
 *   put:
 *     summary: Оновити товар (Лише для Адміна)
 *     tags: [Products]
 *     security:
 *       - bearerAuth: []
 *     parameters:
 *       - in: path
 *         name: id
 *         required: true
 *         schema:
 *           type: integer
 *         description: ID товару
 *     requestBody:
 *       required: true
 *       content:
 *         application/json:
 *           schema:
 *             type: object
 *             properties:
 *               name:
 *                 type: string
 *               price:
 *                 type: number
 *               category_id:
 *                 type: integer
 *               description:
 *                 type: string
 *               stock_status:
 *                 type: string
 *     responses:
 *       200:
 *         description: Товар оновлено
 *       404:
 *         description: Товар не знайдено
 */

 /**
 * @swagger
 * /api/products/{id}:
 *   delete:
 *     summary: Видалити товар (Лише для Адміна)
 *     tags: [Products]
 *     security:
 *       - bearerAuth: []
 *     parameters:
 *       - in: path
 *         name: id
 *         required: true
 *         schema:
 *           type: integer
 *         description: ID товару
 *     responses:
 *       200:
 *         description: Товар видалено
 *       404:
 *         description: Товар не знайдено
 */
```

### 5. Локальне тестування через Swagger UI

Запускаємо сервер: npm start

Відкриваємо у браузері: http://localhost:3000/api-docs

Загальний вигляд сторінки
![](assets/labs/lab-6/1.png)

Наприклад, перевіримо GET /api/products. Сервер повертає статус 200 та масив товарів у форматі JSON
![](assets/labs/lab-6/2.png)

Захищені запити:

Наприклад, перевіримо GET /api/admin/users із валідними даними користувача. Але спочатку виконаємо цей запит без авторизації. У результаті отримаємо помилку, так як таку дію може виконати лише адмін, а ми навіть не авторизовані
![](assets/labs/lab-6/7.png)

Копіюємо accessToken адміну. Натискаємо Authorize у Swagger UI. Вставляємо токен у поле і натискаємо Authorize
![](assets/labs/lab-6/3.png)
![](assets/labs/lab-6/4.png)
![](assets/labs/lab-6/5.png)

Тепер виконуємо захищений запит. Запит пройшов успішно і дані записані в БД
![](assets/labs/lab-6/6.png)

### 6. Продемонструвати підсумковий проєкт: REST API з MySQL та Swagger документацією.

Платформа деплою додатка: хмарний сервіс Render (Веб-служба Node.js).  

Хостинг бази даних: облачний кластер MySQL на платформі Aiven із обов'язковим шифруванням підключення через SSL.

Публічне посилання на отриманий застосунок: https://interior-rudykhko.onrender.com/

Публічне посилання на документацію API (Swagger UI): https://interior-rudykhko.onrender.com/api-docs/

Перевірка відкритих маршрутів: GET-запит до ендпоінту /api/products (отримання каталогу товарів). Запит пройшов успішно (HTTP-статус 200 OK), сервер повернув JSON-масив з даними, що підтвердило стабільний зв'язок розгорнутого додатку з віддаленою БД Aiven та коректну роботу оптимізації через кешування.
![](assets/labs/lab-6/8.png)

Перевірка системи авторизації: POST-запит до /api/auth/login із дійсними обліковими даними адміністратора. Згенерований сервером JWT-токен доступу я застосувала у Swagger UI через вбудований механізм bearerAuth (кнопка Authorize), що дозволило зберегти авторизаційний контекст для подальших дій.
![](assets/labs/lab-6/9.png)

Тестування захищених CRUD-операцій: використовуючи токен доступу, я перевірила роботу POST, PUT та DELETE запитів для сутностей категорій і товарів. Наприклад, під час створення нової категорії (POST /api/categories) Swagger коректно сформував та передав JSON у requestBody, а сервер обробив запит і повернув статус 201 Created. Фактичне збереження нових записів я додатково підтвердила шляхом паралельного SQL-запиту (SELECT * FROM categories) до хмарної бази Aiven через програму MySQL Workbench.

Підсумковий проєкт повністю відповідає всім поставленим вимогам: REST API стабільно функціонує в Інтернеті, CRUD-операції з MySQL виконуються без помилок із дотриманням розмежування прав доступу, а всі ендпоінти, параметри та схеми відповідей детально задокументовані за стандартом OpenAPI 3.0.

## Висновки

Під час виконання лабораторної роботи проєкт було доведено до етапу повноцінного релізу. Локальну базу даних MySQL успішно мігровано у хмарне середовище Aiven, а серверну частину на Node.js розгорнуто на платформі Render. Також інтегровано інструмент Swagger, що забезпечило API зручною інтерактивною документацією.

Працездатність системи перевірено через публічне посилання на Render: відкриті маршрути, авторизація за допомогою JWT-токенів та захищені CRUD-операції функціонують стабільно та без помилок. Наразі розроблений бекенд повністю доступний в інтернеті та готовий до подальшої інтеграції з клієнтською частиною (фронтендом).
