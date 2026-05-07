## Тема, Мета, Місце розташування

**Тема:** Розширені можливості Node.js-додатків: логування, завантаження файлів, моніторинг продуктивності


**Мета:** Ознайомитися з розширеними можливостями серверних застосунків на базі Node.js, а саме:
- реалізацією логування запитів і подій
- організацією завантаження файлів на сервер
- моніторингом продуктивності застосунку


**Місце розташування:**
- GitHub: https://github.com/karolinarm08/Interior_RudykhKO
- Live demo: https://karolinarm08.github.io/Interior_RudykhKO/

---

## Завдання лабораторної роботи

1. Ініціалізація проєкту
2. Логування HTTP-запитів
3. Файлове логування подій
4. Обробка помилок
5. Завантаження одного файлу
6. Завантаження кількох файлів
7. Валідація файлів
8. Моніторинг стану сервера
9. Вимірювання часу відповіді
10. Інтеграція менеджера процесів
11. Додаткове (за бажанням / бонус):
- Реалізувати ротацію логів
- Додати API для перегляду логів
- Побудувати просту панель моніторингу

---

### 1. Ініціалізація проєкту
 
Встановлюємо необхідні залежності для логування, завантаження файлів та моніторингу.

```bash
npm install morgan winston multer
npm install -g pm2
```
![](assets/labs/lab-4/1.png)

---

### 2. Логування HTTP-запитів

Підключаємо morgan, щоб усі HTTP-запити до сервера автоматично виводилися в консоль.

```bash
const morgan = require('morgan');
app.use(morgan('combined'));
```

### 3. Файлове логування подій

Інтегруємо winston для запису подій (рівні info та error) у файл app.log.

```bash
const winston = require('winston');

const logger = winston.createLogger({
    level: 'info',
    format: winston.format.combine(
        winston.format.timestamp(),
        winston.format.json()
    ),
    transports: [
        new winston.transports.File({ filename: 'app.log' }), // Запис у файл
        new winston.transports.Console() // Дублювання в консоль
    ]
});
logger.info('Сервер запущено, логер ініціалізовано');
```

![](assets/labs/lab-4/6.png)


### 4. Обробка помилок

Створюємо middleware для перехоплення помилок, логування їх через winston та повернення клієнту зрозумілого JSON.


```bash
app.use((err, req, res, next) => {
    logger.error(`Помилка: ${err.message}`);
    res.status(500).json({ 
        error: "Сталася помилка на сервері",
        details: err.message 
    });
});
```


### 5,6,7. Завантаження одного файлу. Завантаження кількох файлів. Валідація файлів 

Налаштовуємо multer для збереження файлів у папку uploads/. Додаємо перевірку: дозволено лише файли .jpg, .png, .pdf з обмеженням розміру до 2 МБ. Створюємо ендпоінти /upload (один файл) та /upload-multiple (кілька файлів).

```bash
const multer = require('multer');
const fs = require('fs');

const uploadDir = './uploads';
if (!fs.existsSync(uploadDir)) {
    fs.mkdirSync(uploadDir);
}

// Налаштування збереження
const storage = multer.diskStorage({
    destination: (req, file, cb) => {
        cb(null, 'uploads/');
    },
    filename: (req, file, cb) => {
        cb(null, Date.now() + '-' + file.originalname);
    }
});

// Валідація файлів
const fileFilter = (req, file, cb) => {
    const allowedTypes = ['image/jpeg', 'image/png', 'application/pdf'];
    if (allowedTypes.includes(file.mimetype)) {
        cb(null, true);
    } else {
        cb(new Error('Неправильний формат файлу (дозволено лише jpg, png, pdf)'), false);
    }
};

const upload = multer({ 
    storage: storage,
    limits: { fileSize: 2 * 1024 * 1024 }, // Ліміт 2 МБ
    fileFilter: fileFilter
});

// Ендпоінт для одного файлу
app.post('/upload', upload.single('file'), (req, res) => {
    res.json({ message: 'Файл завантажено', file: req.file });
});

// Ендпоінт для кількох файлів
app.post('/upload-multiple', upload.array('files', 5), (req, res) => {
    res.json({ message: 'Файли завантажено', files: req.files });
});
```

Відкриваємо папку public/uploads/ у редакторі коду і бачимо, що файли, додані на сайті, успішно збереглися
![](assets/labs/lab-4/7.png)


### 8. Моніторинг стану сервера

Створюємо ендпоінт /status, який повертає системну інформацію: uptime та використання пам'яті.

```bash
app.get('/status', (req, res) => {
    const memoryUsage = process.memoryUsage();
    const uptime = process.uptime();
    res.json({
        uptime: uptime,
        memoryUsage: memoryUsage
    });
});
```

Відкриваємо у браузері http://localhost:3000/status. Бачимо метрики uptime та використання пам'яті process.memoryUsage().
![](assets/labs/lab-4/8.png)


### 9. Вимірювання часу відповіді

Створюємо middleware, яке фіксує час отримання запиту та відправки відповіді, а результат записує в лог.

```bash
app.use((req, res, next) => {
    const start = Date.now();
    res.on('finish', () => {
        const duration = Date.now() - start;
        logger.info(`${req.method} ${req.url} - ${duration}ms`);
    });
    next();
});
```

На рядках в логах, бачимо час у мілісекундах. Це доводить роботу middleware для вимірювання часу обробки запиту.
![](assets/labs/lab-4/9.png)

### 10. Інтеграція менеджера процесів

Запускаємо застосунок через PM2, перевіряємо авторестарт та логи.

```bash
pm2 start app.js --name "interior-app"
```

![](assets/labs/lab-4/2.png)

```bash
pm2 monit
```

![](assets/labs/lab-4/3.png)

```bash
pm2 logs
```

![](assets/labs/lab-4/4.png)

```bash
pm2 restart interior-app
```

![](assets/labs/lab-4/5.png)

### 11. Додаткове

#### Реалізувати ротацію логів

Встановлюємо пакет для ротації логів

```bash
npm install winston-daily-rotate-file
```

Замінюємо блок transports у winston на цей

```bash
const DailyRotateFile = require('winston-daily-rotate-file');

const logger = winston.createLogger({
    level: 'info',
    format: winston.format.combine(
        winston.format.timestamp(),
        winston.format.json()
    ),
    transports: [
        new winston.transports.File({ 
            filename: 'app.log',
            maxsize: 5242880, // Максимальний розмір файлу: 5 МБ (в байтах)
            maxFiles: 5,      // Зберігати максимум 5 старих файлів
            tailable: true    // Нові логи завжди писатимуться в app.log
        }), 
        new winston.transports.Console()
    ]
});
logger.info('Сервер запущено, логер ініціалізовано');
```

![](assets/labs/lab-4/10.png)


#### Додати API для перегляду логів

```bash
app.get('/api/logs', (req, res) => {
    // Читаємо стандартний файл логів
    fs.readFile('app.log', 'utf8', (err, data) => {
        if (err) return res.status(500).json({ error: 'Логи не знайдені' });
        // Розбиваємо рядки і віддаємо як масив JSON
        const logs = data.trim().split('\n').map(line => JSON.parse(line));
        res.json(logs);
    });
});
```

![](assets/labs/lab-4/11.png)

#### Побудувати просту панель моніторингу

```bash
app.get('/dashboard', (req, res) => {
    res.send(`
        <html>
            <head><title>Моніторинг</title></head>
            <body style="font-family: Arial; padding: 20px;">
                <h1>Панель моніторингу сервера</h1>
                <div id="stats">Завантаження...</div>
                <script>
                    setInterval(() => {
                        fetch('/status')
                            .then(r => r.json())
                            .then(data => {
                                document.getElementById('stats').innerHTML = 
                                    '<p><b>Uptime:</b> ' + data.uptime.toFixed(2) + ' сек</p>' +
                                    '<p><b>Пам\\'ять (Heap Used):</b> ' + (data.memoryUsage.heapUsed / 1024 / 1024).toFixed(2) + ' MB</p>';
                            });
                    }, 2000); // Оновлення кожні 2 секунди
                </script>
            </body>
        </html>
    `);
});
```

![](assets/labs/lab-4/12.png)


## Висновки

Під час виконання лабораторної роботи я практично засвоїла розширені можливості серверних застосунків на Node.js. За допомогою бібліотеки Morgan я налаштувала логування HTTP-запитів, а через Winston реалізувала запис подій та помилок у файл app.log. Використання middleware Multer дозволило мені організувати безпечне завантаження файлів із застосуванням валідації за форматом та розміром. Також я впровадила моніторинг продуктивності: створила ендпоінт для відстеження використання пам'яті та uptime, а розроблений middleware забезпечив вимірювання часу обробки запитів. Завершальним етапом стала інтеграція менеджера процесів PM2, що забезпечило фонову роботу застосунку та його автоматичний рестарт у разі збоїв.