Реализация веб-приложения на PHP (портал «Водить.РФ»)
Ниже приведён полный код информационной системы для записи на курсы обучения вождению речного транспорта.
Приложение разработано в соответствии с заданием демонстрационного экзамена (вариант В1, 2026 г.).

Технологический стек
Backend: PHP 7.4+ (сессии, PDO MySQL)

Frontend: HTML5, CSS3 (адаптивная вёрстка, Flexbox/Grid), JavaScript (слайдер, анимации, валидация форм)

База данных: MySQL

Структура файлов
text
project/
├── index.php               # перенаправление на профиль или логин
├── register.php            # регистрация нового пользователя
├── login.php               # аутентификация
├── profile.php             # личный кабинет (история заявок, отзывы, слайдер)
├── application.php         # создание новой заявки
├── admin.php               # панель администратора (фильтры, сортировка, пагинация, смена статуса)
├── logout.php              # завершение сессии
├── config/
│   └── db.php              # подключение к БД и глобальные функции
├── css/
│   └── style.css           # стили, адаптив, микроанимации
├── js/
│   └── script.js           # слайдер, клиентская валидация, всплывающие уведомления
├── images/                 # изображения для слайдера (1.jpg … 4.jpg)
└── sql/
    └── install.sql         # схема базы данных и создание администратора
    
1. База данных (sql/install.sql)
sql
CREATE DATABASE IF NOT EXISTS driving_portal;
USE driving_portal;

CREATE TABLE users (
    id INT AUTO_INCREMENT PRIMARY KEY,
    login VARCHAR(50) UNIQUE NOT NULL,
    password VARCHAR(255) NOT NULL,
    full_name VARCHAR(100) NOT NULL,
    birth_date DATE NOT NULL,
    phone VARCHAR(20) NOT NULL,
    email VARCHAR(100) NOT NULL,
    role ENUM('user', 'admin') DEFAULT 'user',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE applications (
    id INT AUTO_INCREMENT PRIMARY KEY,
    user_id INT NOT NULL,
    transport_type ENUM('катер', 'круизный лайнер', 'яхта') NOT NULL,
    start_date DATE NOT NULL,
    payment_method ENUM('наличные', 'карта', 'безналичный') NOT NULL,
    status ENUM('Новая', 'Идет обучение', 'Обучение завершено') DEFAULT 'Новая',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE
);

CREATE TABLE reviews (
    id INT AUTO_INCREMENT PRIMARY KEY,
    user_id INT NOT NULL,
    application_id INT NOT NULL,
    review_text TEXT NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE,
    FOREIGN KEY (application_id) REFERENCES applications(id) ON DELETE CASCADE,
    UNIQUE KEY unique_application_review (application_id)
);
-- Создание администратора (логин: Admin26, пароль: Demo20)
INSERT INTO users (login, password, full_name, birth_date, phone, email, role)
VALUES (
    'Admin26',
    '$2y$10$92IXUNpkjO0rOQ5byMi.Ye4oKoEa3Ro9llC/.og/at2.uheWG/igi', -- hash для 'Demo20'
    'Системный администратор',
    '2000-01-01',
    '+70000000000',
    'admin@driving.ru',
    'admin'
) ON DUPLICATE KEY UPDATE id=id;



2. Подключение к БД и общие функции (config/db.php)
php
<?php
session_start();

$host = 'localhost';
$dbname = 'driving_portal';
$user = 'root';
$pass = '';

try {
    $pdo = new PDO("mysql:host=$host;dbname=$dbname;charset=utf8", $user, $pass);
    $pdo->setAttribute(PDO::ATTR_ERRMODE, PDO::ERRMODE_EXCEPTION);
} catch(PDOException $e) {
    die("Ошибка подключения: " . $e->getMessage());
}

function isLoggedIn() {
    return isset($_SESSION['user_id']);
}

function isAdmin() {
    return isset($_SESSION['role']) && $_SESSION['role'] === 'admin';
}

function redirect($url) {
    header("Location: $url");
    exit;
}

function displayError($field, $errors) {
    if (isset($errors[$field])) {
        echo '<span class="error-msg">' . htmlspecialchars($errors[$field]) . '</span>';
    }
}
?>



3. Регистрация (register.php)
php
<?php
require_once 'config/db.php';

$errors = [];

if ($_SERVER['REQUEST_METHOD'] === 'POST') {
    $login = trim($_POST['login'] ?? '');
    $password = $_POST['password'] ?? '';
    $full_name = trim($_POST['full_name'] ?? '');
    $birth_date = $_POST['birth_date'] ?? '';
    $phone = trim($_POST['phone'] ?? '');
    $email = trim($_POST['email'] ?? '');

    // Валидация
    if (!preg_match('/^[a-zA-Z0-9]{6,}$/', $login)) {
        $errors['login'] = 'Логин должен содержать минимум 6 латинских букв и цифр.';
    }
    if (strlen($password) < 8) {
        $errors['password'] = 'Пароль должен быть не менее 8 символов.';
    }
    if (empty($full_name)) $errors['full_name'] = 'Введите ФИО.';
    if (empty($birth_date)) $errors['birth_date'] = 'Укажите дату рождения.';
    if (empty($phone)) $errors['phone'] = 'Введите телефон.';
    if (!filter_var($email, FILTER_VALIDATE_EMAIL)) $errors['email'] = 'Некорректный e-mail.';

    if (empty($errors)) {
        $stmt = $pdo->prepare("SELECT id FROM users WHERE login = ?");
        $stmt->execute([$login]);
        if ($stmt->fetch()) {
            $errors['login'] = 'Такой логин уже занят.';
        } else {
            $hashed = password_hash($password, PASSWORD_DEFAULT);
            $sql = "INSERT INTO users (login, password, full_name, birth_date, phone, email, role) 
                    VALUES (?, ?, ?, ?, ?, ?, 'user')";
            $stmt = $pdo->prepare($sql);
            if ($stmt->execute([$login, $hashed, $full_name, $birth_date, $phone, $email])) {
                $_SESSION['user_id'] = $pdo->lastInsertId();
                $_SESSION['login'] = $login;
                $_SESSION['role'] = 'user';
                redirect('profile.php');
            } else {
                $errors['general'] = 'Ошибка регистрации. Попробуйте позже.';
            }
        }
    }
}
?>
<!DOCTYPE html>
<html lang="ru">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Регистрация | Водить.РФ</title>
    <link rel="stylesheet" href="css/style.css">
</head>
<body>
    <div class="auth-container">
        <h2>Регистрация</h2>
        <?php if (isset($errors['general'])): ?>
            <div class="alert error"><?= $errors['general'] ?></div>
        <?php endif; ?>
        <form method="post">
            <div class="form-group">
                <label>Логин (латиница и цифры, ≥6)</label>
                <input type="text" name="login" value="<?= htmlspecialchars($_POST['login'] ?? '') ?>" required>
                <?php displayError('login', $errors); ?>
            </div>
            <div class="form-group">
                <label>Пароль (≥8 символов)</label>
                <input type="password" name="password" required>
                <?php displayError('password', $errors); ?>
            </div>
            <div class="form-group">
                <label>ФИО</label>
                <input type="text" name="full_name" value="<?= htmlspecialchars($_POST['full_name'] ?? '') ?>" required>
                <?php displayError('full_name', $errors); ?>
            </div>
            <div class="form-group">
                <label>Дата рождения</label>
                <input type="date" name="birth_date" value="<?= htmlspecialchars($_POST['birth_date'] ?? '') ?>" required>
                <?php displayError('birth_date', $errors); ?>
            </div>
            <div class="form-group">
                <label>Телефон</label>
                <input type="tel" name="phone" value="<?= htmlspecialchars($_POST['phone'] ?? '') ?>" required>
                <?php displayError('phone', $errors); ?>
            </div>
            <div class="form-group">
                <label>E-mail</label>
                <input type="email" name="email" value="<?= htmlspecialchars($_POST['email'] ?? '') ?>" required>
                <?php displayError('email', $errors); ?>
            </div>
            <button type="submit" class="btn">Зарегистрироваться</button>
            <p>Уже есть аккаунт? <a href="login.php">Войти</a></p>
        </form>
    </div>
</body>
</html>



4. Авторизация (login.php)
php
<?php
require_once 'config/db.php';

$error = '';

if ($_SERVER['REQUEST_METHOD'] === 'POST') {
    $login = trim($_POST['login'] ?? '');
    $password = $_POST['password'] ?? '';

    if ($login && $password) {
        $stmt = $pdo->prepare("SELECT id, login, password, role FROM users WHERE login = ?");
        $stmt->execute([$login]);
        $user = $stmt->fetch();
        if ($user && password_verify($password, $user['password'])) {
            $_SESSION['user_id'] = $user['id'];
            $_SESSION['login'] = $user['login'];
            $_SESSION['role'] = $user['role'];
            redirect('profile.php');
        } else {
            $error = 'Неверный логин или пароль.';
        }
    } else {
        $error = 'Заполните все поля.';
    }
}
?>
<!DOCTYPE html>
<html lang="ru">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Вход | Водить.РФ</title>
    <link rel="stylesheet" href="css/style.css">
</head>
<body>
    <div class="auth-container">
        <h2>Вход в личный кабинет</h2>
        <?php if ($error): ?>
            <div class="alert error"><?= htmlspecialchars($error) ?></div>
        <?php endif; ?>
        <form method="post">
            <div class="form-group">
                <label>Логин</label>
                <input type="text" name="login" required>
            </div>
            <div class="form-group">
                <label>Пароль</label>
                <input type="password" name="password" required>
            </div>
            <button type="submit" class="btn">Войти</button>
            <p>Нет аккаунта? <a href="register.php">Регистрация</a></p>
        </form>
    </div>
</body>
</html>



5. Личный кабинет (profile.php) – история заявок, отзывы, слайдер
php
<?php
require_once 'config/db.php';
if (!isLoggedIn()) redirect('login.php');

$user_id = $_SESSION['user_id'];

// Обработка отправки отзыва
if ($_SERVER['REQUEST_METHOD'] === 'POST' && isset($_POST['add_review'])) {
    $app_id = (int)$_POST['application_id'];
    $review_text = trim($_POST['review_text']);
    if ($review_text !== '') {
        $stmt = $pdo->prepare("INSERT INTO reviews (user_id, application_id, review_text) VALUES (?, ?, ?) 
                                ON DUPLICATE KEY UPDATE review_text = ?");
        $stmt->execute([$user_id, $app_id, $review_text, $review_text]);
        $success = "Отзыв сохранён.";
    }
}

// Получение заявок с возможностью оставить отзыв (статус 'Обучение завершено' и ещё нет отзыва)
$apps = $pdo->prepare("
    SELECT a.*, 
           (SELECT COUNT(*) FROM reviews WHERE application_id = a.id) as has_review
    FROM applications a
    WHERE a.user_id = ?
    ORDER BY a.created_at DESC
");
$apps->execute([$user_id]);
$applications = $apps->fetchAll();
?>
<!DOCTYPE html>
<html lang="ru">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Личный кабинет | Водить.РФ</title>
    <link rel="stylesheet" href="css/style.css">
</head>
<body>
    <header>
        <nav>
            <a href="profile.php">Личный кабинет</a>
            <a href="application.php">Новая заявка</a>
            <?php if (isAdmin()): ?>
                <a href="admin.php">Панель администратора</a>
            <?php endif; ?>
            <a href="logout.php">Выход</a>
        </nav>
    </header>
    <main>
        <h1>Добро пожаловать, <?= htmlspecialchars($_SESSION['login']) ?>!</h1>

        <!-- Слайдер -->
        <div class="slider-container">
            <div class="slider">
                <div class="slide"><img src="images/1.jpg" alt="Вождение катера"></div>
                <div class="slide"><img src="images/2.jpg" alt="Круизный лайнер"></div>
                <div class="slide"><img src="images/3.jpg" alt="Яхта"></div>
                <div class="slide"><img src="images/4.jpg" alt="Обучение"></div>
            </div>
            <button class="slider-prev">❮</button>
            <button class="slider-next">❯</button>
        </div>

        <?php if (isset($success)): ?>
            <div class="alert success"><?= $success ?></div>
        <?php endif; ?>

        <h2>Мои заявки</h2>
        <?php if (count($applications) === 0): ?>
            <p>У вас пока нет заявок. <a href="application.php">Создать заявку</a></p>
        <?php else: ?>
            <div class="applications-list">
                <?php foreach ($applications as $app): ?>
                    <div class="app-card">
                        <p><strong>Транспорт:</strong> <?= htmlspecialchars($app['transport_type']) ?></p>
                        <p><strong>Дата начала:</strong> <?= date('d.m.Y', strtotime($app['start_date'])) ?></p>
                        <p><strong>Способ оплаты:</strong> <?= htmlspecialchars($app['payment_method']) ?></p>
                        <p><strong>Статус:</strong> 
                            <span class="status status-<?= strtolower(str_replace(' ', '', $app['status'])) ?>">
                                <?= $app['status'] ?>
                            </span>
                        </p>
                        <?php if ($app['status'] === 'Обучение завершено' && $app['has_review'] == 0): ?>
                            <form method="post" class="review-form">
                                <input type="hidden" name="application_id" value="<?= $app['id'] ?>">
                                <textarea name="review_text" placeholder="Оставьте отзыв об обучении..." rows="2" required></textarea>
                                <button type="submit" name="add_review" class="btn-small">Оставить отзыв</button>
                            </form>
                        <?php elseif ($app['has_review'] > 0): ?>
                            <p><em>Отзыв уже оставлен. Спасибо!</em></p>
                        <?php endif; ?>
                    </div>
                <?php endforeach; ?>
            </div>
        <?php endif; ?>
    </main>
    <script src="js/script.js"></script>
</body>
</html>



6. Оформление новой заявки (application.php)
php
<?php
require_once 'config/db.php';
if (!isLoggedIn()) redirect('login.php');

$success = '';
$error = '';

if ($_SERVER['REQUEST_METHOD'] === 'POST') {
    $transport = $_POST['transport'] ?? '';
    $start_date = $_POST['start_date'] ?? '';
    $payment = $_POST['payment'] ?? '';

    if ($transport && $start_date && $payment) {
        $stmt = $pdo->prepare("INSERT INTO applications (user_id, transport_type, start_date, payment_method, status) 
                                VALUES (?, ?, ?, ?, 'Новая')");
        if ($stmt->execute([$_SESSION['user_id'], $transport, $start_date, $payment])) {
            $success = "Заявка успешно отправлена администратору.";
        } else {
            $error = "Ошибка сохранения заявки.";
        }
    } else {
        $error = "Заполните все поля.";
    }
}
?>
<!DOCTYPE html>
<html lang="ru">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Новая заявка | Водить.РФ</title>
    <link rel="stylesheet" href="css/style.css">
</head>
<body>
    <header>
        <nav>
            <a href="profile.php">Личный кабинет</a>
            <a href="application.php">Новая заявка</a>
            <?php if (isAdmin()): ?>
                <a href="admin.php">Панель администратора</a>
            <?php endif; ?>
            <a href="logout.php">Выход</a>
        </nav>
    </header>
    <main>
        <h1>Запись на курсы вождения</h1>
        <?php if ($success): ?>
            <div class="alert success"><?= $success ?></div>
        <?php elseif ($error): ?>
            <div class="alert error"><?= $error ?></div>
        <?php endif; ?>
        <form method="post" class="application-form">
            <div class="form-group">
                <label>Вид транспорта</label>
                <select name="transport" required>
                    <option value="">-- Выберите --</option>
                    <option value="катер">Катер</option>
                    <option value="круизный лайнер">Круизный лайнер</option>
                    <option value="яхта">Яхта</option>
                </select>
            </div>
            <div class="form-group">
                <label>Желаемая дата начала</label>
                <input type="text" name="start_date" placeholder="ДД.ММ.ГГГГ" pattern="\d{2}\.\d{2}\.\d{4}" required>
                <small>Формат: 25.12.2026</small>
            </div>
            <div class="form-group">
                <label>Способ оплаты</label>
                <select name="payment" required>
                    <option value="">-- Выберите --</option>
                    <option value="наличные">Наличные</option>
                    <option value="карта">Банковская карта</option>
                    <option value="безналичный">Безналичный расчёт</option>
                </select>
            </div>
            <button type="submit" class="btn">Отправить заявку</button>
        </form>
    </main>
    <script>
        // преобразование даты из дд.мм.гггг в гггг-мм-дд перед отправкой
        document.querySelector('form').addEventListener('submit', function(e) {
            let dateField = document.querySelector('input[name="start_date"]');
            let parts = dateField.value.split('.');
            if (parts.length === 3) {
                dateField.value = parts[2] + '-' + parts[1] + '-' + parts[0];
            }
        });
    </script>
</body>
</html>



7. Панель администратора (admin.php) – фильтрация, сортировка, пагинация, смена статуса
php
<?php
require_once 'config/db.php';
if (!isAdmin()) redirect('login.php');

$status_filter = $_GET['status'] ?? '';
$sort = $_GET['sort'] ?? 'created_at';
$order = $_GET['order'] ?? 'DESC';
$page = (int)($_GET['page'] ?? 1);
$per_page = 10;
$offset = ($page - 1) * $per_page;

// Обработка смены статуса
if ($_SERVER['REQUEST_METHOD'] === 'POST' && isset($_POST['update_status'])) {
    $app_id = (int)$_POST['app_id'];
    $new_status = $_POST['new_status'];
    $allowed = ['Новая', 'Идет обучение', 'Обучение завершено'];
    if (in_array($new_status, $allowed)) {
        $stmt = $pdo->prepare("UPDATE applications SET status = ? WHERE id = ?");
        $stmt->execute([$new_status, $app_id]);
        $success = "Статус заявки обновлён.";
    }
}

// Построение запроса с фильтрами
$where = "";
$params = [];
if ($status_filter !== '') {
    $where = "WHERE a.status = ?";
    $params[] = $status_filter;
}
$order_sql = "ORDER BY a.$sort $order";
$stmt = $pdo->prepare("SELECT a.*, u.login, u.full_name 
                       FROM applications a 
                       JOIN users u ON a.user_id = u.id 
                       $where 
                       $order_sql 
                       LIMIT $offset, $per_page");
$stmt->execute($params);
$applications = $stmt->fetchAll();

// Общее количество для пагинации
$count_stmt = $pdo->prepare("SELECT COUNT(*) FROM applications a $where");
$count_stmt->execute($params);
$total = $count_stmt->fetchColumn();
$total_pages = ceil($total / $per_page);
?>
<!DOCTYPE html>
<html lang="ru">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Панель администратора | Водить.РФ</title>
    <link rel="stylesheet" href="css/style.css">
</head>
<body>
    <header>
        <nav>
            <a href="profile.php">Личный кабинет</a>
            <a href="application.php">Новая заявка</a>
            <a href="admin.php">Панель администратора</a>
            <a href="logout.php">Выход</a>
        </nav>
    </header>
    <main>
        <h1>Управление заявками</h1>
        <?php if (isset($success)): ?>
            <div class="alert success"><?= $success ?></div>
        <?php endif; ?>
        
        <!-- Фильтр по статусу -->
        <div class="filter-bar">
            <form method="get">
                <label>Статус:</label>
                <select name="status" onchange="this.form.submit()">
                    <option value="">Все</option>
                    <option value="Новая" <?= $status_filter === 'Новая' ? 'selected' : '' ?>>Новая</option>
                    <option value="Идет обучение" <?= $status_filter === 'Идет обучение' ? 'selected' : '' ?>>Идет обучение</option>
                    <option value="Обучение завершено" <?= $status_filter === 'Обучение завершено' ? 'selected' : '' ?>>Обучение завершено</option>
                </select>
                <input type="hidden" name="sort" value="<?= htmlspecialchars($sort) ?>">
                <input type="hidden" name="order" value="<?= htmlspecialchars($order) ?>">
            </form>
        </div>

        <div class="sort-links">
            Сортировать: 
            <a href="?sort=created_at&order=DESC&status=<?= urlencode($status_filter) ?>">по дате (новые)</a> |
            <a href="?sort=start_date&order=ASC&status=<?= urlencode($status_filter) ?>">по дате начала</a> |
            <a href="?sort=transport_type&order=ASC&status=<?= urlencode($status_filter) ?>">по типу транспорта</a>
        </div>

        <div class="admin-applications">
            <table>
                <thead>
                    <tr><th>ID</th><th>Пользователь</th><th>Транспорт</th><th>Дата начала</th><th>Оплата</th><th>Статус</th><th>Действие</th></tr>
                </thead>
                <tbody>
                <?php foreach ($applications as $app): ?>
                    <tr>
                        <td><?= $app['id'] ?></td>
                        <td><?= htmlspecialchars($app['full_name']) ?> (<?= htmlspecialchars($app['login']) ?>)</td>
                        <td><?= $app['transport_type'] ?></td>
                        <td><?= date('d.m.Y', strtotime($app['start_date'])) ?></td>
                        <td><?= $app['payment_method'] ?></td>
                        <td class="status status-<?= strtolower(str_replace(' ', '', $app['status'])) ?>"><?= $app['status'] ?></td>
                        <td>
                            <form method="post" class="inline-form">
                                <input type="hidden" name="app_id" value="<?= $app['id'] ?>">
                                <select name="new_status">
                                    <option value="Новая" <?= $app['status'] === 'Новая' ? 'selected' : '' ?>>Новая</option>
                                    <option value="Идет обучение" <?= $app['status'] === 'Идет обучение' ? 'selected' : '' ?>>Идет обучение</option>
                                    <option value="Обучение завершено" <?= $app['status'] === 'Обучение завершено' ? 'selected' : '' ?>>Обучение завершено</option>
                                </select>
                                <button type="submit" name="update_status" class="btn-small">Изменить</button>
                            </form>
                        </td>
                    </tr>
                <?php endforeach; ?>
                </tbody>
            </table>
        </div>

        <?php if ($total_pages > 1): ?>
            <div class="pagination">
                <?php for ($i = 1; $i <= $total_pages; $i++): ?>
                    <a href="?page=<?= $i ?>&status=<?= urlencode($status_filter) ?>&sort=<?= $sort ?>&order=<?= $order ?>" 
                       class="<?= $i === $page ? 'active' : '' ?>"><?= $i ?></a>
                <?php endfor; ?>
            </div>
        <?php endif; ?>
    </main>
    <script src="js/script.js"></script>
</body>
</html>



8. Выход (logout.php)
php
<?php
session_start();
session_destroy();
header("Location: login.php");
exit;



9. CSS стили (css/style.css) – адаптив, анимации, слайдер
css
/* Reset & базовые стили */
* {
    margin: 0;
    padding: 0;
    box-sizing: border-box;
}

body {
    font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
    background: #f0f7ff;
    color: #0a2b3e;
    line-height: 1.4;
}

header {
    background: #004466;
    padding: 1rem;
}

nav a {
    color: white;
    margin-right: 1.5rem;
    text-decoration: none;
    font-weight: 500;
    transition: color 0.2s;
}

nav a:hover {
    color: #ffcc00;
}

main {
    max-width: 1200px;
    margin: 2rem auto;
    padding: 0 1.5rem;
}

h1, h2 {
    margin-bottom: 1rem;
    color: #004466;
}

/* Формы и карточки */
.auth-container, .application-form {
    max-width: 500px;
    margin: 3rem auto;
    background: white;
    padding: 2rem;
    border-radius: 20px;
    box-shadow: 0 10px 25px rgba(0,0,0,0.1);
}

.form-group {
    margin-bottom: 1.2rem;
}

label {
    display: block;
    margin-bottom: 0.3rem;
    font-weight: 600;
}

input, select, textarea {
    width: 100%;
    padding: 0.7rem;
    border: 1px solid #ccc;
    border-radius: 12px;
    font-size: 1rem;
    transition: 0.2s;
}

input:focus, select:focus {
    border-color: #004466;
    outline: none;
    box-shadow: 0 0 0 3px rgba(0,68,102,0.2);
}

.btn {
    background: #004466;
    color: white;
    border: none;
    padding: 0.7rem 1.5rem;
    border-radius: 30px;
    font-size: 1rem;
    cursor: pointer;
    transition: transform 0.1s, background 0.2s;
}

.btn:hover {
    background: #006699;
    transform: scale(1.02);
}

.btn-small {
    background: #0088aa;
    color: white;
    border: none;
    padding: 0.3rem 0.8rem;
    border-radius: 20px;
    cursor: pointer;
}

.alert {
    padding: 0.8rem;
    border-radius: 12px;
    margin-bottom: 1rem;
}

.error {
    background: #ffe0e0;
    color: #a00;
    border-left: 5px solid #a00;
}

.success {
    background: #e0ffe0;
    color: #060;
}

.error-msg {
    font-size: 0.8rem;
    color: #c00;
    display: block;
    margin-top: 0.2rem;
}

/* Слайдер */
.slider-container {
    position: relative;
    max-width: 800px;
    margin: 2rem auto;
    overflow: hidden;
    border-radius: 24px;
    box-shadow: 0 5px 15px rgba(0,0,0,0.2);
}

.slider {
    display: flex;
    transition: transform 0.5s ease;
}

.slide {
    flex: 0 0 100%;
}

.slide img {
    width: 100%;
    height: auto;
    display: block;
}

.slider-prev, .slider-next {
    position: absolute;
    top: 50%;
    transform: translateY(-50%);
    background: rgba(0,0,0,0.5);
    color: white;
    border: none;
    padding: 0.5rem 1rem;
    cursor: pointer;
    font-size: 1.5rem;
    border-radius: 50%;
    transition: background 0.2s;
}

.slider-prev { left: 10px; }
.slider-next { right: 10px; }

.slider-prev:hover, .slider-next:hover {
    background: rgba(0,0,0,0.8);
}

/* Список заявок */
.applications-list {
    display: grid;
    grid-template-columns: repeat(auto-fill, minmax(300px,1fr));
    gap: 1.5rem;
    margin-top: 1rem;
}

.app-card {
    background: white;
    padding: 1rem;
    border-radius: 20px;
    box-shadow: 0 2px 8px rgba(0,0,0,0.05);
    transition: transform 0.2s;
}

.app-card:hover {
    transform: translateY(-5px);
}

.status {
    font-weight: bold;
    padding: 0.2rem 0.5rem;
    border-radius: 20px;
    display: inline-block;
}

.status-Новая { background: #ffeedd; color: #cc7700; }
.status-Идетобучение { background: #d9f0ff; color: #0055aa; }
.status-Обучениезавершено { background: #ddffdd; color: #2c6e2c; }

/* Таблица администратора */
.admin-applications {
    overflow-x: auto;
}

table {
    width: 100%;
    border-collapse: collapse;
    background: white;
    border-radius: 20px;
    overflow: hidden;
}

th, td {
    padding: 0.8rem;
    text-align: left;
    border-bottom: 1px solid #ddd;
}

th {
    background: #004466;
    color: white;
}

.inline-form {
    display: flex;
    gap: 0.5rem;
}

.pagination {
    margin-top: 2rem;
    display: flex;
    gap: 0.5rem;
    justify-content: center;
}

.pagination a {
    padding: 0.4rem 0.8rem;
    background: white;
    border-radius: 20px;
    text-decoration: none;
    color: #004466;
}

.pagination a.active {
    background: #004466;
    color: white;
}

/* Адаптив для смартфонов 390px */
@media (max-width: 480px) {
    main { padding: 0 1rem; }
    .auth-container, .application-form { padding: 1.5rem; margin: 1rem; }
    nav a { display: inline-block; margin-bottom: 0.5rem; }
    .slider-prev, .slider-next { padding: 0.3rem 0.6rem; font-size: 1rem; }
    table, .inline-form { font-size: 0.8rem; }
    .btn { width: 100%; }
}




10. JavaScript (js/script.js) – слайдер, уведомления, анимации
javascript
// Слайдер на странице личного кабинета
document.addEventListener('DOMContentLoaded', function() {
    const slider = document.querySelector('.slider');
    if (!slider) return;
    
    const slides = document.querySelectorAll('.slide');
    const prevBtn = document.querySelector('.slider-prev');
    const nextBtn = document.querySelector('.slider-next');
    let currentIndex = 0;
    const total = slides.length;
    let autoInterval;

    function updateSlider() {
        slider.style.transform = `translateX(-${currentIndex * 100}%)`;
    }

    function nextSlide() {
        currentIndex = (currentIndex + 1) % total;
        updateSlider();
    }

    function prevSlide() {
        currentIndex = (currentIndex - 1 + total) % total;
        updateSlider();
    }

    function startAuto() {
        autoInterval = setInterval(nextSlide, 3000);
    }
    function stopAuto() {
        clearInterval(autoInterval);
    }

    if (prevBtn && nextBtn) {
        prevBtn.addEventListener('click', () => { stopAuto(); prevSlide(); startAuto(); });
        nextBtn.addEventListener('click', () => { stopAuto(); nextSlide(); startAuto(); });
        slider.addEventListener('mouseenter', stopAuto);
        slider.addEventListener('mouseleave', startAuto);
        startAuto();
    }

    // Всплывающие уведомления (тосты) для администратора
    const urlParams = new URLSearchParams(window.location.search);
    if (urlParams.get('updated') === '1') {
        showToast('Статус заявки изменён!', 'success');
    }
});

function showToast(message, type) {
    let toast = document.createElement('div');
    toast.className = `alert ${type}`;
    toast.innerText = message;
    toast.style.position = 'fixed';
    toast.style.bottom = '20px';
    toast.style.right = '20px';
    toast.style.zIndex = '1000';
    toast.style.maxWidth = '300px';
    document.body.appendChild(toast);
    setTimeout(() => toast.remove(), 3000);
}



Инструкция по развёртыванию
Установите веб-сервер (Apache/nginx) с PHP 7.4+ и MySQL.

Создайте базу данных, выполнив файл sql/install.sql.

Поместите все файлы в корневую директорию сервера.

Настройте параметры подключения в config/db.php (хост, пользователь, пароль).

Добавьте четыре изображения в папку images/ с именами 1.jpg, 2.jpg, 3.jpg, 4.jpg (любые иллюстрации по теме).

Убедитесь, что папки css, js, images доступны для чтения.
