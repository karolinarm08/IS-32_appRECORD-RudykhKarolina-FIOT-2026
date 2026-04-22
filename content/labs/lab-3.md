## Тема, Мета, Місце розташування

**Тема:** РОЗРОБКА ФУНКЦІОНАЛЬНОГО REST API. РЕЄСТРАЦІЯ ТА
АВТОРИЗАЦІЯ КОРИСТУВАЧІВ. ВАЛІДАЦІЯ ДАНИХ І ОБРОБКА ПОМИЛОК. 


**Мета:** Метою даної лабораторної роботи є вивчення принципів побудови REST API; набуття практичних навичок розробки серверного застосунку з використанням платформи Node.js і фреймворку Express; реалізувати механізми реєстрації та авторизації користувачів; забезпечити валідацію вхідних даних; забезпечити обробку помилок; організувати захищений доступ до ресурсів із використанням JWT-токенів і системи ролей користувачів.


**Місце розташування:**
- GitHub: https://github.com/karolinarm08/Interior_RudykhKO
- Live demo: https://karolinarm08.github.io/Interior_RudykhKO/

---

## Завдання лабораторної роботи

1. Встановити необхідні бібліотеки
2. Реалізувати реєстрацію та авторизацію користувача
3. Додати валідацію даних, обробку помилок
4. Реалізувати захищений маршрут
5. Протестувати API через Postman або curl
6. Проаналізувати отримані результати
7. Додати підтвердження пароля при реєстрації
8. Додати роль користувача (admin/user)
9. Реалізувати logout
10. Додати оновлення профілю
11. Зберігати користувачів у базі
12. Реалізувати refresh token
13. Додати логування помилок
14. Обмежити кількість спроб входу
15. Додати middleware для перевірки токена
16. Реалізувати зміну пароля
17. Реалізувати видалення користувача
18. Реалізувати відновлення пароля
19. Додати підтвердження email
20. Реалізувати OAuth (Google login)

---

### 1. Встановити необхідні бібліотеки

![](assets/labs/lab-3/11.png)
![](assets/labs/lab-3/12.png)
![](assets/labs/lab-3/13.png)


---

### 2. Реалізувати реєстрацію та авторизацію користувача

#### Реєстрація користувача
У застосунку було реалізовано маршрути POST /api/auth/register та POST /api/auth/login. Під час реєстрації користувач вводить ім’я, email, пароль і підтвердження пароля. Сервер виконує перевірку коректності вхідних даних, перевіряє, чи не існує вже користувач із таким email, хешує пароль за допомогою бібліотеки bcryptjs і зберігає нового користувача в таблиці users бази даних MySQL. Усім новим користувачам автоматично призначається роль user. Також під час реєстрації формується токен підтвердження email.

![](assets/labs/lab-3/21.png)


```bash
app.post(
  '/api/auth/register',
  [
    body('name')
      .trim()
      .notEmpty()
      .withMessage('Ім’я є обов’язковим'),

    body('email')
      .isEmail()
      .withMessage('Некоректний email'),

    body('password')
      .notEmpty()
      .withMessage('Пароль є обов’язковим')
      .isLength({ min: 8 })
      .withMessage('Пароль має містити мінімум 8 символів')
      .matches(/[A-Z]/)
      .withMessage('Пароль має містити хоча б одну велику літеру')
      .matches(/[a-z]/)
      .withMessage('Пароль має містити хоча б одну малу літеру')
      .matches(/[0-9]/)
      .withMessage('Пароль має містити хоча б одну цифру')
      .matches(/[!@#$%^&*()_\-+=[\]{};:'"\\|,.<>/?]/)
      .withMessage('Пароль має містити хоча б один спеціальний символ'),

    body('confirmPassword')
      .notEmpty()
      .withMessage('Підтвердження пароля є обов’язковим')
  ],
  async (req, res, next) => {
    try {
      const validation = sendValidationErrors(req, res);
      if (validation) return validation;

      const { name, email, password, confirmPassword } = req.body;

      if (password !== confirmPassword) {
        return res.status(400).json({
          message: 'Пароль і підтвердження пароля не співпадають'
        });
      }

      const db = await connectDB();

      const [existingUsers] = await db.execute(
        'SELECT id FROM users WHERE email = ?',
        [email.trim().toLowerCase()]
      );

      if (existingUsers.length) {
        await db.end();
        return res.status(400).json({
          message: 'Користувач з таким email вже існує'
        });
      }

      const hashedPassword = await bcrypt.hash(password, 10);
      const emailVerificationToken = createRandomToken();

      const [result] = await db.execute(
        `
        INSERT INTO users
        (name, email, password_hash, role, email_verification_token)
        VALUES (?, ?, ?, 'user', ?)
        `,
        [
          name.trim(),
          email.trim().toLowerCase(),
          hashedPassword,
          emailVerificationToken
        ]
      );

      const verifyLink = `${getBaseUrl(req)}/api/auth/verify-email/${emailVerificationToken}`;

      await sendEmailOrLog(
        email.trim().toLowerCase(),
        'Підтвердження email',
        `Перейдіть за посиланням для підтвердження email: ${verifyLink}`
      );

      await db.end();

      res.status(201).json({
        message: 'Користувача зареєстровано. Підтвердіть email.',
        userId: result.insertId,
        verifyLink
      });
    } catch (error) {
      next(error);
    }
  }
);

```

#### Авторизація користувача
Під час авторизації користувач вводить email і пароль. Сервер знаходить користувача в базі даних, перевіряє підтвердження email, порівнює введений пароль із хешем у базі даних і в разі успішної перевірки генерує accessToken та refreshToken за допомогою JWT. Отримані токени використовуються для подальшого доступу до захищених ресурсів системи.

![](assets/labs/lab-3/22.png)

```bash
app.post(
  '/api/auth/login',
  loginLimiter,
  [
    body('email').isEmail().withMessage('Некоректний email'),
    body('password').notEmpty().withMessage('Пароль є обов’язковим')
  ],
  async (req, res, next) => {
    try {
      const validation = sendValidationErrors(req, res);
      if (validation) return validation;

      const { email, password } = req.body;

      const db = await connectDB();

      const [rows] = await db.execute(
        'SELECT * FROM users WHERE email = ?',
        [email.trim().toLowerCase()]
      );

      if (!rows.length) {
        await db.end();
        return res.status(400).json({
          message: 'Користувача не знайдено'
        });
      }

      const user = rows[0];

      if (!user.is_email_confirmed) {
        await db.end();
        return res.status(403).json({
          message: 'Підтвердіть email перед входом'
        });
      }

      const isMatch = await bcrypt.compare(password, user.password_hash);

      if (!isMatch) {
        await db.end();
        return res.status(400).json({
          message: 'Невірний пароль'
        });
      }

      const accessToken = createAccessToken(user);
      const refreshToken = createRefreshToken(user);

      await db.execute(
        'UPDATE users SET refresh_token = ? WHERE id = ?',
        [refreshToken, user.id]
      );

      await db.end();

      res.json({
        message: 'Вхід успішний',
        accessToken,
        refreshToken,
        user: {
          id: user.id,
          name: user.name,
          email: user.email,
          role: user.role
        }
      });
    } catch (error) {
      next(error);
    }
  }
);
```

#### Генерація токенів

```bash
function createAccessToken(user) {
  return jwt.sign(
    {
      id: user.id,
      email: user.email,
      role: user.role
    },
    process.env.JWT_ACCESS_SECRET,
    { expiresIn: process.env.ACCESS_TOKEN_EXPIRES_IN || '15m' }
  );
}

function createRefreshToken(user) {
  return jwt.sign(
    {
      id: user.id,
      email: user.email,
      role: user.role
    },
    process.env.JWT_REFRESH_SECRET,
    { expiresIn: process.env.REFRESH_TOKEN_EXPIRES_IN || '7d' }
  );
}
```

---

### 3. Додати валідацію даних, обробку помилок

У розробленому застосунку було реалізовано перевірку вхідних даних користувача за допомогою бібліотеки express-validator. Валідація застосовується до основних маршрутів системи, зокрема реєстрації, авторизації, оновлення профілю та зміни пароля.

Під час реєстрації перевіряється:

- наявність імені користувача;
- коректність email-адреси;
- складність пароля (мінімальна довжина, наявність великої та малої літери, цифри та  спеціального символу);
- наявність підтвердження пароля.

У випадку некоректних даних сервер повертає HTTP-статус 400 Bad Request та список помилок, що дозволяє користувачу зрозуміти причину відмови.

Для централізованої обробки помилок було реалізовано middleware errorHandler, який перехоплює всі необроблені винятки та повертає клієнту повідомлення про внутрішню помилку сервера зі статусом 500 Internal Server Error.

![](assets/labs/lab-3/31.png)
![](assets/labs/lab-3/32.png)

#### Функція валідації

```bash
function sendValidationErrors(req, res) {
  const errors = validationResult(req);

  if (!errors.isEmpty()) {
    return res.status(400).json({
      message: 'Помилка валідації',
      errors: errors.array()
    });
  }

  return null;
}
```

#### Middleware обробки помилок

```bash
function errorHandler(err, req, res, next) {
  console.error(err);

  res.status(500).json({
    message: 'Внутрішня помилка сервера'
  });
}
```


---

### 4. Реалізувати захищений маршрут

У застосунку було реалізовано захищені маршрути, доступ до яких мають лише авторизовані користувачі. Для цього використано механізм JWT (JSON Web Token).

Після успішної авторизації користувач отримує accessToken, який передається у заголовку HTTP-запиту Authorization у форматі Bearer <token>. Для перевірки токена було реалізовано middleware verifyAccessToken, який декодує токен, перевіряє його дійсність та додає інформацію про користувача до об’єкта req.

У разі відсутності токена або його недійсності сервер повертає помилку 401 Unauthorized. Якщо токен коректний — доступ до маршруту дозволяється.

Також реалізовано перевірку ролей користувача (наприклад, admin), що дозволяє обмежити доступ до окремих маршрутів лише для адміністратора. У разі недостатніх прав сервер повертає помилку 403 Forbidden.

![](assets/labs/lab-3/41.png)
![](assets/labs/lab-3/42.png)

#### Middleware перевірки токена

```bash
function verifyAccessToken(req, res, next) {
  const authHeader = req.headers.authorization;

  if (!authHeader || !authHeader.startsWith('Bearer ')) {
    return res.status(401).json({
      message: 'Немає токена доступу'
    });
  }

  const token = authHeader.split(' ')[1];

  try {
    const decoded = jwt.verify(token, process.env.JWT_ACCESS_SECRET);
    req.user = decoded;
    next();
  } catch (error) {
    return res.status(401).json({
      message: 'Недійсний або прострочений токен'
    });
  }
}
```

#### Middleware перевірки ролі

```bash
function checkRole(role) {
  return (req, res, next) => {
    if (req.user.role !== role) {
      return res.status(403).json({
        message: 'Недостатньо прав доступу'
      });
    }
    next();
  };
}
```

---

### 5. Протестувати API

Для тестування REST API було використано інструмент Thunder Client, який є розширенням для середовища розробки Visual Studio Code та дозволяє надсилати HTTP-запити безпосередньо з редактора.

Під час тестування було перевірено основні маршрути системи:

- реєстрація користувача;
![](assets/labs/lab-3/51.png)
- підтвердження email;
![](assets/labs/lab-3/52.png)
- авторизація;
![](assets/labs/lab-3/56.png)
- доступ до захищених маршрутів;
![](assets/labs/lab-3/54.png)
![](assets/labs/lab-3/55.png)
- оновлення токенів (refresh token);
![](assets/labs/lab-3/53.png)
- вихід із системи;
- відновлення пароля;

У процесі тестування перевірялися:

- передача даних у форматі JSON;
- обробка заголовків (зокрема Authorization: Bearer <token>);
- правильність HTTP-відповідей;
- робота JWT-токенів;
- обробка помилкових ситуацій.


---

### 6. Проаналізувати отримані результати

Під час тестування розробленого REST API було перевірено основні функції системи: реєстрацію користувача, підтвердження email, авторизацію, доступ до захищених маршрутів, оновлення профілю, зміну пароля, вихід із системи, відновлення пароля, а також розмежування прав доступу між ролями admin і user.

У ході перевірки встановлено, що маршрут реєстрації коректно створює нового користувача лише за умови правильного заповнення всіх обов’язкових полів. При введенні некоректного email, занадто короткого пароля або невідповідності пароля та його підтвердження сервер повертає повідомлення про помилку валідації, що підтверджує правильність перевірки вхідних даних.

Після успішної реєстрації користувач повинен підтвердити email, і лише після цього авторизація стає доступною. Це показало, що механізм підтвердження email працює коректно та забезпечує додатковий рівень захисту облікового запису.

Під час тестування авторизації було отримано accessToken і refreshToken, що підтвердило коректність реалізації JWT-автентифікації. Захищений маршрут /api/profile без токена повертав помилку 401 Unauthorized, а з коректним токеном — дані поточного користувача. Це свідчить про правильну реалізацію захищеного доступу до ресурсів.

Було також перевірено систему ролей. Користувач із роллю admin мав доступ до адміністративних маршрутів, зокрема до керування користувачами, товарами та категоріями. Користувач із роллю user не мав доступу до цих дій, а сервер повертав помилку 403 Forbidden. Це підтверджує правильну реалізацію розмежування прав доступу.

Під час перевірки функцій оновлення профілю, зміни пароля, logout, refresh token і відновлення пароля система також показала коректну роботу. Дані профілю успішно змінювалися, пароль оновлювався лише після перевірки поточного пароля, а refresh token дозволяв отримати новий access token без повторної авторизації.


---

### 7. Додати підтвердження пароля при реєстрації

У застосунку було реалізовано механізм підтвердження пароля під час реєстрації користувача. Для цього у форму реєстрації додано поле confirmPassword, яке користувач повинен заповнити повторно.

На стороні сервера виконується перевірка відповідності значень полів password та confirmPassword. У випадку, якщо значення не співпадають, сервер повертає повідомлення про помилку та статус 400 Bad Request, що запобігає створенню облікового запису з некоректними даними.

![](assets/labs/lab-3/71.png)


#### Валідація поля confirmPassword

```bash
body('confirmPassword')
  .notEmpty()
  .withMessage('Підтвердження пароля є обов’язковим')
```

#### Основна перевірка паролів

```bash
if (password !== confirmPassword) {
  return res.status(400).json({
    message: 'Пароль і підтвердження пароля не співпадають'
  });
}
```

---

### 8. Додати роль користувача (admin/user)

У застосунку було реалізовано механізм розмежування прав доступу за допомогою ролей користувачів. Для цього в таблицю users бази даних додано поле role, яке може приймати значення admin або user.

Під час реєстрації всім новим користувачам автоматично призначається роль user. Роль адміністратора (admin) встановлюється вручну безпосередньо в базі даних.

Інформація про роль користувача включається до JWT-токена під час авторизації. Це дозволяє використовувати роль для перевірки доступу до захищених маршрутів.

Для обмеження доступу до адміністративних функцій було реалізовано middleware checkRole, яке перевіряє роль користувача перед виконанням маршруту. Якщо користувач не має необхідних прав, сервер повертає помилку 403 Forbidden.

#### Поле role в базі даних

```bash
role ENUM('admin', 'user') DEFAULT 'user'
```

#### Призначення ролі при реєстрації

```bash
INSERT INTO users
(name, email, password_hash, role, email_verification_token)
VALUES (?, ?, ?, 'user', ?)
```

#### Додавання ролі в JWT

```bash
function createAccessToken(user) {
  return jwt.sign(
    {
      id: user.id,
      email: user.email,
      role: user.role
    },
    process.env.JWT_ACCESS_SECRET,
    { expiresIn: process.env.ACCESS_TOKEN_EXPIRES_IN || '15m' }
  );
}
```

#### Middleware перевірки ролі

```bash
function checkRole(role) {
  return (req, res, next) => {
    if (req.user.role !== role) {
      return res.status(403).json({
        message: 'Недостатньо прав доступу'
      });
    }
    next();
  };
}
```

---

### 9. Реалізувати logout

Після отримання запиту сервер визначає поточного користувача за JWT access token, а потім очищає збережений у базі даних refresh_token. Це означає, що після виходу користувач більше не може оновлювати токени без повторної авторизації.

На стороні клієнта під час виходу також очищаються дані з localStorage, а саме:

- accessToken
- refreshToken
- user

Після цього користувач вважається неавторизованим і перенаправляється на головну сторінку або сторінку входу.

#### Backend-маршрут logout

```bash
app.post('/api/auth/logout', verifyAccessToken, async (req, res, next) => {
  try {
    const db = await connectDB();

    await db.execute(
      'UPDATE users SET refresh_token = NULL WHERE id = ?',
      [req.user.id]
    );

    await db.end();

    res.json({
      message: 'Вихід виконано успішно'
    });
  } catch (error) {
    next(error);
  }
});
```

#### Frontend-очищення localStorage

```bash
function logoutUser() {
  localStorage.removeItem('accessToken');
  localStorage.removeItem('refreshToken');
  localStorage.removeItem('user');
  window.location.href = 'index.html';
}
```

#### Виклик logout із фронту

```bash
logoutLink.addEventListener('click', async (e) => {
  e.preventDefault();

  try {
    const token = getAccessToken();
    if (token) {
      await fetch('/api/auth/logout', {
        method: 'POST',
        headers: {
          'Authorization': `Bearer ${token}`
        }
      });
    }
  } catch (error) {
    console.error(error);
  }

  logoutUser();
});
```

---

### 10. Додати оновлення профілю

Користувач може змінювати ім’я та email, перед збереженням виконується валідація та перевірка унікальності email.

![](assets/labs/lab-3/10-1.png)

#### Маршрут оновлення профілю

```bash
app.put(
  '/api/profile',
  verifyAccessToken,
  [
    body('name')
      .trim()
      .notEmpty()
      .withMessage('Ім’я є обов’язковим'),

    body('email')
      .isEmail()
      .withMessage('Некоректний email')
  ],
  async (req, res, next) => {
    try {
      const validation = sendValidationErrors(req, res);
      if (validation) return validation;

      const { name, email } = req.body;

      const db = await connectDB();

      const [existing] = await db.execute(
        'SELECT id FROM users WHERE email = ? AND id != ?',
        [email.trim().toLowerCase(), req.user.id]
      );

      if (existing.length) {
        await db.end();
        return res.status(400).json({
          message: 'Такий email уже використовується'
        });
      }

      await db.execute(
        'UPDATE users SET name = ?, email = ? WHERE id = ?',
        [name.trim(), email.trim().toLowerCase(), req.user.id]
      );

      await db.end();

      res.json({
        message: 'Профіль успішно оновлено'
      });
    } catch (error) {
      next(error);
    }
  }
);
```

#### Відправка запиту з фронту

```bash
profileForm.addEventListener('submit', async (e) => {
  e.preventDefault();

  const payload = {
    name: document.getElementById('profileName').value.trim(),
    email: document.getElementById('profileEmail').value.trim()
  };

  try {
    const response = await fetch('/api/profile', {
      method: 'PUT',
      headers: {
        'Content-Type': 'application/json',
        'Authorization': `Bearer ${getAccessToken()}`
      },
      body: JSON.stringify(payload)
    });

    const data = await parseResponse(response);

    showMessage(data.message);
  } catch (error) {
    showMessage(error.message, true);
  }
});
```

---

### 11. Зберігати користувачів у базі

Користувачі зберігаються не в масиві, а в таблиці users бази даних MySQL. Під час реєстрації запис додається в БД, а під час авторизації користувач шукається саме там.

#### Структура таблиці users

```bash
CREATE TABLE IF NOT EXISTS users (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    email VARCHAR(255) NOT NULL UNIQUE,
    password_hash VARCHAR(255) NOT NULL,
    role ENUM('admin', 'user') DEFAULT 'user',
    is_email_confirmed BOOLEAN DEFAULT FALSE,
    email_verification_token VARCHAR(255) NULL,
    password_reset_token VARCHAR(255) NULL,
    password_reset_expires DATETIME NULL,
    refresh_token TEXT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);
```

#### Підключення до бази даних

```bash
const mysql = require('mysql2/promise');
require('dotenv').config();

async function connectDB() {
  return mysql.createConnection({
    host: process.env.DB_HOST || 'localhost',
    user: process.env.DB_USER || 'root',
    password: process.env.DB_PASSWORD || '',
    database: process.env.DB_NAME || 'interior_shop',
    port: Number(process.env.DB_PORT || 3306)
  });
}

module.exports = connectDB;
```

#### Додавання користувача в базу під час реєстрації

```bash
const [result] = await db.execute(
  `
  INSERT INTO users
  (name, email, password_hash, role, email_verification_token)
  VALUES (?, ?, ?, 'user', ?)
  `,
  [
    name.trim(),
    email.trim().toLowerCase(),
    hashedPassword,
    emailVerificationToken
  ]
);
```

#### Отримання користувача з бази під час авторизації

```bash
const [rows] = await db.execute(
  'SELECT * FROM users WHERE email = ?',
  [email.trim().toLowerCase()]
);
```

---

### 12. Реалізувати refresh token

Після логіну система генерує access token і refresh token. Refresh token зберігається в базі даних і використовується для отримання нових токенів через маршрут /api/auth/refresh без повторного введення пароля.

#### Функція створення refresh token

```bash
function createRefreshToken(user) {
  return jwt.sign(
    {
      id: user.id,
      email: user.email,
      role: user.role
    },
    process.env.JWT_REFRESH_SECRET,
    { expiresIn: process.env.REFRESH_TOKEN_EXPIRES_IN || '7d' }
  );
}
```

#### Збереження refresh token у базі після входу

```bash
const refreshToken = createRefreshToken(user);

await db.execute(
  'UPDATE users SET refresh_token = ? WHERE id = ?',
  [refreshToken, user.id]
);
```

#### Маршрут оновлення токенів

```bash
app.post('/api/auth/refresh', async (req, res, next) => {
  try {
    const { refreshToken } = req.body;

    if (!refreshToken) {
      return res.status(400).json({
        message: 'Refresh token обов’язковий'
      });
    }

    let decoded;
    try {
      decoded = jwt.verify(refreshToken, process.env.JWT_REFRESH_SECRET);
    } catch (error) {
      return res.status(401).json({
        message: 'Недійсний refresh token'
      });
    }

    const db = await connectDB();

    const [rows] = await db.execute(
      'SELECT * FROM users WHERE id = ? AND refresh_token = ?',
      [decoded.id, refreshToken]
    );

    if (!rows.length) {
      await db.end();
      return res.status(401).json({
        message: 'Refresh token не знайдено'
      });
    }

    const user = rows[0];
    const newAccessToken = createAccessToken(user);
    const newRefreshToken = createRefreshToken(user);

    await db.execute(
      'UPDATE users SET refresh_token = ? WHERE id = ?',
      [newRefreshToken, user.id]
    );

    await db.end();

    res.json({
      message: 'Токени оновлено',
      accessToken: newAccessToken,
      refreshToken: newRefreshToken
    });
  } catch (error) {
    next(error);
  }
});
```


---

### 13. Додати логування помилок

Під час виникнення помилки вона перехоплюється централізованим middleware errorHandler, після чого передається до функції логування. У файл журналу (logs/error.log) записується така інформація:

- дата та час виникнення помилки;
- HTTP-метод запиту;
- URL маршруту;
- текст помилки та стек викликів.

Користувачу при цьому повертається узагальнене повідомлення про помилку зі статусом 500 Internal Server Error, без розкриття технічних деталей.
#### Функція логування

```bash
const fs = require('fs');
const path = require('path');

function logError(err, req) {
  const logMessage = `
-------------------------
${new Date().toISOString()}
${req.method} ${req.originalUrl}
${err.stack || err.message}
`;

  const logPath = path.join(__dirname, '../logs/error.log');

  fs.appendFileSync(logPath, logMessage);
}

module.exports = { logError };
```

#### Middleware обробки помилок

```bash
const { logError } = require('../utils/logger');

function errorHandler(err, req, res, next) {
  console.error(err);

  logError(err, req);

  res.status(500).json({
    message: 'Внутрішня помилка сервера'
  });
}

module.exports = errorHandler;
```

#### Підключення middleware в app.js

```bash
app.use(errorHandler);
```


---

### 14. Обмежити кількість спроб входу

У поточній реалізації дозволено не більше 5 спроб входу протягом 15 хвилин. Якщо цей ліміт перевищено, сервер повертає повідомлення про те, що спроб входу забагато і потрібно спробувати пізніше.

#### Створення ліміту запитів

```bash
const rateLimit = require('express-rate-limit');

const loginLimiter = rateLimit({
  windowMs: 15 * 60 * 1000,
  max: 5,
  standardHeaders: true,
  legacyHeaders: false,
  message: {
    message: 'Забагато спроб входу. Спробуйте пізніше.'
  }
});
```

#### Підключення ліміту до маршруту входу

```bash
app.post(
  '/api/auth/login',
  loginLimiter,
  [
    body('email').isEmail().withMessage('Некоректний email'),
    body('password').notEmpty().withMessage('Пароль є обов’язковим')
  ],
  async (req, res, next) => {
    ...
  }
);
```


---


### 15. Додати middleware для перевірки токена

Middleware verifyAccessToken перевіряє наявність заголовка Authorization у запиті, витягує токен у форматі Bearer <token> та виконує його валідацію за допомогою бібліотеки jsonwebtoken. У разі успішної перевірки декодована інформація про користувача додається до об’єкта req, що дозволяє використовувати ці дані в наступних обробниках запиту.

Якщо токен відсутній або недійсний, сервер повертає помилку 401 Unauthorized і забороняє доступ до ресурсу.

#### Middleware перевірки токена

```bash
function verifyAccessToken(req, res, next) {
  const authHeader = req.headers.authorization;

  if (!authHeader || !authHeader.startsWith('Bearer ')) {
    return res.status(401).json({
      message: 'Немає токена доступу'
    });
  }

  const token = authHeader.split(' ')[1];

  try {
    const decoded = jwt.verify(token, process.env.JWT_ACCESS_SECRET);
    req.user = decoded;
    next();
  } catch (error) {
    return res.status(401).json({
      message: 'Недійсний або прострочений токен'
    });
  }
}
```

#### Приклад використання middleware

```bash
app.get('/api/profile', verifyAccessToken, async (req, res, next) => {
  ...
});
```

---

### 16. Реалізувати зміну пароля

Зміну пароля реалізовано через захищений маршрут PATCH /api/profile/change-password. Користувач вводить поточний і новий пароль, сервер перевіряє правильність поточного пароля, хешує новий і зберігає його в базі.

![](assets/labs/lab-3/16-1.png)

#### Маршрут зміни пароля

```bash
app.patch(
  '/api/profile/change-password',
  verifyAccessToken,
  [
    body('currentPassword')
      .notEmpty()
      .withMessage('Поточний пароль є обов’язковим'),
    body('newPassword')
      .isLength({ min: 6 })
      .withMessage('Новий пароль має містити мінімум 6 символів'),
    body('confirmNewPassword')
      .notEmpty()
      .withMessage('Підтвердження нового пароля є обов’язковим')
  ],
  async (req, res, next) => {
    try {
      const validation = sendValidationErrors(req, res);
      if (validation) return validation;

      const { currentPassword, newPassword, confirmNewPassword } = req.body;

      if (newPassword !== confirmNewPassword) {
        return res.status(400).json({
          message: 'Новий пароль і підтвердження не співпадають'
        });
      }

      const db = await connectDB();

      const [rows] = await db.execute(
        'SELECT * FROM users WHERE id = ?',
        [req.user.id]
      );

      if (!rows.length) {
        await db.end();
        return res.status(404).json({
          message: 'Користувача не знайдено'
        });
      }

      const user = rows[0];
      const isMatch = await bcrypt.compare(currentPassword, user.password_hash);

      if (!isMatch) {
        await db.end();
        return res.status(400).json({
          message: 'Поточний пароль невірний'
        });
      }

      const hashedPassword = await bcrypt.hash(newPassword, 10);

      await db.execute(
        `
        UPDATE users
        SET password_hash = ?, refresh_token = NULL
        WHERE id = ?
        `,
        [hashedPassword, req.user.id]
      );

      await db.end();

      res.json({
        message: 'Пароль успішно змінено. Увійдіть повторно.'
      });
    } catch (error) {
      next(error);
    }
  }
);
```

#### Відправка запиту з фронту

```bash
changePasswordForm.addEventListener('submit', async (e) => {
  e.preventDefault();

  try {
    await authFetch('/api/profile/change-password', {
      method: 'PATCH',
      headers: {
        'Content-Type': 'application/json'
      },
      body: JSON.stringify({
        currentPassword: document.getElementById('currentPassword').value,
        newPassword: document.getElementById('newPassword').value,
        confirmNewPassword: document.getElementById('confirmNewPassword').value
      })
    });

    showProfileMessage('Пароль змінено. Виконай повторний вхід.');
    changePasswordForm.reset();
  } catch (error) {
    showProfileMessage(error.message, true);
  }
});
```


---

### 17. Реалізувати видалення користувача

Видалення користувача реалізовано через маршрут DELETE /api/users/:id. Доступ до нього має лише адміністратор. Перед видаленням перевіряється існування користувача, а потім запис видаляється з таблиці users.

![](assets/labs/lab-3/17-1.png)

#### Маршрут видалення користувача

```bash
app.delete(
  '/api/users/:id',
  verifyAccessToken,
  allowRoles('admin'),
  async (req, res, next) => {
    try {
      const userId = Number(req.params.id);

      if (!userId) {
        return res.status(400).json({
          message: 'Некоректний id користувача'
        });
      }

      const db = await connectDB();

      const [rows] = await db.execute(
        'SELECT id FROM users WHERE id = ?',
        [userId]
      );

      if (!rows.length) {
        await db.end();
        return res.status(404).json({
          message: 'Користувача не знайдено'
        });
      }

      await db.execute('DELETE FROM users WHERE id = ?', [userId]);
      await db.end();

      res.json({
        message: 'Користувача видалено'
      });
    } catch (error) {
      next(error);
    }
  }
);
```


---

### 18. Реалізувати відновлення пароля

Відновлення пароля реалізовано у два етапи: спочатку система генерує токен і посилання для скидання, а потім за цим токеном користувач встановлює новий пароль. Новий пароль хешується і зберігається в базі.

![](assets/labs/lab-3/18-1.png)

#### Маршрут створення токена відновлення

```bash
app.post(
  '/api/auth/forgot-password',
  [body('email').isEmail().withMessage('Некоректний email')],
  async (req, res, next) => {
    try {
      const validation = sendValidationErrors(req, res);
      if (validation) return validation;

      const { email } = req.body;
      const db = await connectDB();

      const [rows] = await db.execute(
        'SELECT id, email FROM users WHERE email = ?',
        [email.trim().toLowerCase()]
      );

      if (!rows.length) {
        await db.end();
        return res.status(404).json({
          message: 'Користувача з таким email не знайдено'
        });
      }

      const token = createRandomToken();
      const expires = new Date(Date.now() + 60 * 60 * 1000);

      await db.execute(
        `
        UPDATE users
        SET password_reset_token = ?, password_reset_expires = ?
        WHERE email = ?
        `,
        [token, expires, email.trim().toLowerCase()]
      );

      const resetLink = `${getBaseUrl(req)}/api/auth/reset-password/${token}`;

      await sendEmailOrLog(
        email.trim().toLowerCase(),
        'Відновлення пароля',
        `Перейдіть за посиланням для скидання пароля: ${resetLink}`
      );

      await db.end();

      res.json({
        message: 'Посилання для відновлення пароля створено',
        resetLink
      });
    } catch (error) {
      next(error);
    }
  }
);
```

#### Маршрут встановлення нового пароля

```bash
app.post(
  '/api/auth/reset-password/:token',
  [
    body('newPassword')
      .isLength({ min: 6 })
      .withMessage('Новий пароль має містити мінімум 6 символів'),
    body('confirmNewPassword')
      .notEmpty()
      .withMessage('Підтвердження нового пароля є обов’язковим')
  ],
  async (req, res, next) => {
    try {
      const validation = sendValidationErrors(req, res);
      if (validation) return validation;

      const { newPassword, confirmNewPassword } = req.body;

      if (newPassword !== confirmNewPassword) {
        return res.status(400).json({
          message: 'Новий пароль і підтвердження не співпадають'
        });
      }

      const db = await connectDB();

      const [rows] = await db.execute(
        `
        SELECT * FROM users
        WHERE password_reset_token = ?
          AND password_reset_expires IS NOT NULL
          AND password_reset_expires > NOW()
        `,
        [req.params.token]
      );

      if (!rows.length) {
        await db.end();
        return res.status(400).json({
          message: 'Недійсний або прострочений токен відновлення'
        });
      }

      const user = rows[0];
      const hashedPassword = await bcrypt.hash(newPassword, 10);

      await db.execute(
        `
        UPDATE users
        SET password_hash = ?,
            password_reset_token = NULL,
            password_reset_expires = NULL,
            refresh_token = NULL
        WHERE id = ?
        `,
        [hashedPassword, user.id]
      );

      await db.end();

      res.json({
        message: 'Пароль успішно змінено'
      });
    } catch (error) {
      next(error);
    }
  }
);
```

---

### 19. Додати підтвердження email

Після реєстрації система генерує токен підтвердження email і формує посилання. Після переходу за ним у користувача в базі встановлюється ознака підтвердження, і тільки після цього він може увійти в систему.

![](assets/labs/lab-3/19-1.png)
![](assets/labs/lab-3/19-2.png)

#### Генерація токена під час реєстрації

```bash
const emailVerificationToken = createRandomToken();
```

#### Збереження токена в базі даних

```bash
const [result] = await db.execute(
  `
  INSERT INTO users
  (name, email, password_hash, role, email_verification_token)
  VALUES (?, ?, ?, 'user', ?)
  `,
  [
    name.trim(),
    email.trim().toLowerCase(),
    hashedPassword,
    emailVerificationToken
  ]
);
```

#### Формування посилання для підтвердження

```bash
const verifyLink = `${getBaseUrl(req)}/api/auth/verify-email/${emailVerificationToken}`;
```

#### Надсилання або виведення посилання

```bash
await sendEmailOrLog(
  email.trim().toLowerCase(),
  'Підтвердження email',
  `Перейдіть за посиланням для підтвердження email: ${verifyLink}`
);
```


#### Маршрут підтвердження email

```bash
app.get('/api/auth/verify-email/:token', async (req, res, next) => {
  try {
    const db = await connectDB();

    const [rows] = await db.execute(
      'SELECT id, is_email_confirmed FROM users WHERE email_verification_token = ?',
      [req.params.token]
    );

    if (!rows.length) {
      await db.end();
      return res.status(400).json({
        message: 'Недійсний токен підтвердження email'
      });
    }

    await db.execute(
      `
      UPDATE users
      SET is_email_confirmed = 1,
          email_verification_token = NULL
      WHERE id = ?
      `,
      [rows[0].id]
    );

    await db.end();

    res.json({
      message: 'Email успішно підтверджено'
    });
  } catch (error) {
    next(error);
  }
});
```


#### Перевірка підтвердження під час авторизації

```bash
if (!user.is_email_confirmed) {
  await db.end();
  return res.status(403).json({
    message: 'Підтвердіть email перед входом'
  });
}
```

---

### 20. Реалізувати OAuth (Google login)

Я реалізувала OAuth-авторизацію через Google на рівні коду за допомогою passport і passport-google-oauth20. Для повного запуску потрібно підставити GOOGLE_CLIENT_ID і GOOGLE_CLIENT_SECRET у .env, але вся логіка авторизації вже реалізована.

#### Підключення GoogleStrategy

```bash
const passport = require('passport');
const GoogleStrategy = require('passport-google-oauth20').Strategy;
const crypto = require('crypto');
const connectDB = require('../db');
```

#### Перевірка, чи налаштований Google OAuth

```bash
const googleEnabled = Boolean(
  process.env.GOOGLE_CLIENT_ID &&
  process.env.GOOGLE_CLIENT_SECRET &&
  process.env.GOOGLE_CALLBACK_URL
);
```
#### Налаштування GoogleStrategy

```bash
if (googleEnabled) {
  passport.use(
    new GoogleStrategy(
      {
        clientID: process.env.GOOGLE_CLIENT_ID,
        clientSecret: process.env.GOOGLE_CLIENT_SECRET,
        callbackURL: process.env.GOOGLE_CALLBACK_URL
      },
      async (accessToken, refreshToken, profile, done) => {
        try {
          const email = profile.emails?.[0]?.value;
          if (!email) {
            return done(new Error('Google не повернув email'));
          }

          const db = await connectDB();
          const [rows] = await db.execute(
            'SELECT * FROM users WHERE email = ?',
            [email]
          );

          let user;

          if (rows.length) {
            user = rows[0];
          } else {
            const randomPassword = crypto.randomBytes(16).toString('hex');

            const [result] = await db.execute(
              `
              INSERT INTO users (name, email, password_hash, role, is_email_confirmed)
              VALUES (?, ?, ?, 'user', 1)
              `,
              [profile.displayName || 'Google User', email, randomPassword]
            );

            const [newRows] = await db.execute(
              'SELECT * FROM users WHERE id = ?',
              [result.insertId]
            );

            user = newRows[0];
          }

          await db.end();
          done(null, user);
        } catch (error) {
          done(error);
        }
      }
    )
  );
}
```
#### Маршрут ініціалізації входу через Google

```bash
app.get('/api/auth/google', (req, res, next) => {
  if (!googleEnabled) {
    return res.status(400).json({
      message: 'Google OAuth не налаштований у .env'
    });
  }

  passport.authenticate('google', {
    scope: ['profile', 'email'],
    session: false
  })(req, res, next);
});
```
#### Callback-маршрут

```bash
app.get('/api/auth/google/callback', (req, res, next) => {
  if (!googleEnabled) {
    return res.status(400).json({
      message: 'Google OAuth не налаштований у .env'
    });
  }

  passport.authenticate('google', { session: false }, async (err, user) => {
    try {
      if (err) {
        return next(err);
      }

      if (!user) {
        return res.status(401).json({
          message: 'Google авторизація не вдалася'
        });
      }

      const accessToken = createAccessToken(user);
      const refreshToken = createRefreshToken(user);

      const db = await connectDB();
      await db.execute(
        'UPDATE users SET refresh_token = ? WHERE id = ?',
        [refreshToken, user.id]
      );
      await db.end();

      res.json({
        message: 'Google login успішний',
        accessToken,
        refreshToken,
        user: {
          id: user.id,
          name: user.name,
          email: user.email,
          role: user.role
        }
      });
    } catch (error) {
      next(error);
    }
  })(req, res, next);
});
```
#### Callback-маршрут

```bash
app.get('/api/auth/google/callback', (req, res, next) => {
  if (!googleEnabled) {
    return res.status(400).json({
      message: 'Google OAuth не налаштований у .env'
    });
  }

  passport.authenticate('google', { session: false }, async (err, user) => {
    try {
      if (err) {
        return next(err);
      }

      if (!user) {
        return res.status(401).json({
          message: 'Google авторизація не вдалася'
        });
      }

      const accessToken = createAccessToken(user);
      const refreshToken = createRefreshToken(user);

      const db = await connectDB();
      await db.execute(
        'UPDATE users SET refresh_token = ? WHERE id = ?',
        [refreshToken, user.id]
      );
      await db.end();

      res.json({
        message: 'Google login успішний',
        accessToken,
        refreshToken,
        user: {
          id: user.id,
          name: user.name,
          email: user.email,
          role: user.role
        }
      });
    } catch (error) {
      next(error);
    }
  })(req, res, next);
});
```

---


## Висновки

У ході виконання лабораторної роботи я реалізувала серверну частину веб-застосунку з системою реєстрації та авторизації користувачів. Було створено базу даних для зберігання користувачів, реалізовано валідацію даних, хешування паролів, підтвердження email та відновлення пароля. Також я додала JWT-авторизацію, захищені маршрути, refresh token, систему ролей (admin/user) та обмеження доступу. Реалізовано зміну пароля, оновлення профілю, logout і видалення користувачів. Для підвищення безпеки додано обмеження спроб входу та логування помилок. Я закріпила навички роботи з Node.js, базами даних і створення безпечних веб-застосунків. Мету роботи досягнуто.