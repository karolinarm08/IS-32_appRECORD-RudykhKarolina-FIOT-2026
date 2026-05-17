## Тема, Мета, Місце розташування

**Тема:** Безпека та продуктивність серверних додатків Безпека Node.js-додатків Оптимізація запитів і кешування Тестування API


**Мета:** 
- забезпечувати базову безпеку Node.js-додатків;
- оптимізувати продуктивність REST API;
- використовувати кешування для зменшення навантаження на сервер;
- тестувати API за допомогою сучасних інструментів;
- аналізувати продуктивність backend-застосунків.

**Місце розташування:**
- GitHub: https://github.com/karolinarm08/Interior_RudykhKO
- Live demo: https://karolinarm08.github.io/Interior_RudykhKO/

---

## Завдання лабораторної роботи

1. Створити REST API на Node.js та Express.
2. Реалізувати захист API:
- Helmet;
- rate-limit;
- валідацію даних.
3. Реалізувати кешування відповідей.
4. Оптимізувати один із маршрутів API.
5. Провести тестування API.
6. Проаналізувати продуктивність до та після оптимізації.
7. Оформити звіт.

---

### 1. Встановлення необхідних модулів

![](assets/labs/lab-5/1.png)
![](assets/labs/lab-5/2.png)
![](assets/labs/lab-5/3.png)

helmet: захищає HTTP-заголовки. express-rate-limit: обмежує кількість запитів від одного користувача (захист від DDoS/перебору). express-validator: перевіряє дані, що приходять від клієнта. node-cache: просте кешування в пам'яті серверу.

### 2. Захист API

#### Підключення бібліотек безпеки

```bash
const helmet = require('helmet');
const rateLimit = require('express-rate-limit');
const { body, validationResult } = require('express-validator');
const NodeCache = require('node-cache');
const jwt = require('jsonwebtoken');
const bcrypt = require('bcryptjs');
const swaggerUi = require('swagger-ui-express');
const swaggerJsdoc = require('swagger-jsdoc');
```

#### Підключення middleware

```bash
app.use(helmet());
```

#### Обмеження запитів (Rate Limiting)
Обмежує кількість запитів до API та захищає від DDoS-атак.
```bash
const limiter = rateLimit({
    windowMs: 1 * 60 * 1000,
    max: 1000
});

app.use('/api', limiter);
});

app.use(limiter);
```

#### JWT-автентифікація
Реалізовано:
- access token
- refresh token
- перевірка токенів через middleware

```bash
function createAccessToken(user) {
  return jwt.sign(
    { id: user.id, email: user.email, role: user.role },
    process.env.JWT_ACCESS_SECRET,
    { expiresIn: '15m' }
  );
}
```

Авторизація ролей
```bash
allowRoles('admin')
```

### 3. Кешування

Ініціалізація кешу

```bash
const NodeCache = require('node-cache');
const cache = new NodeCache({ stdTTL: 60 });
```
Запис у кеш
```bash
cache.set(cacheKey, rows);
```
Очищення кешу при додаванні, оновленні товару, видаленні товару
```bash
cache.del('products');
```
Перевірка кешування. Відкриваю одну сторінку два рази підряд. Другий запит працює швидше.
![](assets/labs/lab-5/7.png)

### 4. Оптимізація маршруту API
Було оптимізовано маршрут: GET /api/products
До оптимізації: кожен запит звертається до бази даних, високе навантаження
Після оптимізації (з кешем):
```bash
const cachedProducts = cache.get(cacheKey);

if (cachedProducts) {
    return res.json(cachedProducts);
}
```
```bash
cache.set(cacheKey, rows);
```
Тепер зменшено кількість звернень до БД, пришвидшено відповідь API, знижено навантаження на сервер

### 5. Swagger документація 

```bash
const swaggerOptions = {
  definition: {
    openapi: '3.0.0',
    info: {
      title: 'Interior API',
      version: '1.0.0',
      description: 'API для магазину меблів'
    },
    servers: [
      {
        url: `http://localhost:${PORT}`
      }
    ]
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
API документовано через OpenAPI 3.0.
![](assets/labs/lab-5/11.png)

### 6. Тестування API
Створення тестового файлу server.test.js
```bash
const request = require('supertest');

describe('API testing', () => {

  test('GET /products', async () => {

    const response = await request('http://localhost:3000')
      .get('/products');

    expect(response.statusCode).toBe(200);
  });

});
```
Запуск тестів
![](assets/labs/lab-5/4.png)

Перевірка навантаження.
![](assets/labs/lab-5/5.png)
![](assets/labs/lab-5/6.png)

Було проведено тестування API за допомогою:

ThunderClient
- GET /api/products
![](assets/labs/lab-5/10.png)
Swagger (дозволяє тестувати API через веб-інтерфейс)
![](assets/labs/lab-5/9.png)

Перевірка 404
![](assets/labs/lab-5/8.png)

### 7. Аналіз продуктивності
Без кешування:
- кожен запит звертається до бази даних
- високе навантаження
- повільніша відповідь

З кешуванням:
- дані беруться з кешу
- зменшено навантаження на БД
- швидша відповідь API

## Висновки

У ході виконання лабораторної роботи було розроблено повноцінний REST API на Node.js з використанням Express, який забезпечує роботу з користувачами, товарами та категоріями. У межах роботи було реалізовано механізми захисту API, зокрема використано Helmet для захисту HTTP-заголовків, rate limiting для обмеження кількості запитів до сервера, а також валідацію вхідних даних за допомогою express-validator, що дозволило зменшити ризики некоректних або шкідливих запитів.

Також було реалізовано JWT-автентифікацію з використанням access та refresh токенів, що забезпечує безпечний доступ до захищених маршрутів і можливість оновлення сесії користувача без повторного входу в систему. Додатково впроваджено рольову модель доступу, яка дозволяє розмежовувати права користувачів та адміністраторів.

У процесі виконання роботи було реалізовано кешування даних за допомогою NodeCache, що дозволило зменшити кількість звернень до бази даних та підвищити швидкість обробки запитів. Також було виконано оптимізацію одного з основних маршрутів API — отримання списку товарів, що позитивно вплинуло на продуктивність системи.

Окремо було реалізовано автоматизоване тестування API із використанням Jest та Supertest, для чого було створено окремий тестовий файл server.test.js. Крім того, проведено ручне тестування через Postman та навантажувальне тестування, що дозволило оцінити роботу системи під різними умовами.