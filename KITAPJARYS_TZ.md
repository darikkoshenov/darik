# KitapJarys — техническая спецификация (MVP + расширение)

## 1) Цель системы

KitapJarys — веб-платформа для онлайн-соревнований по знаниям с регистрацией по номеру телефона, подтверждением через WhatsApp OTP, ограниченным временем тестирования, автоматическим подсчетом результатов и админ-панелью рейтингов.

Ключевые цели:
- надежная авторизация по номеру телефона (1 номер = 1 аккаунт);
- проведение тестов по расписанию (start/end window);
- контроль времени попытки с авто-submit;
- прозрачный рейтинг и экспорт результатов;
- мультиязычность (RU / KZ);
- стабильная работа при 100–200 одновременных пользователей.

---

## 2) Технологический стек

### Backend (предпочтительно)
- **NestJS** (Node.js, TypeScript)
- Модули: Auth, OTP, Tests, Attempts, Results, Admin, i18n
- Валидация: `class-validator`
- Документация API: Swagger (`@nestjs/swagger`)

### База данных
- **PostgreSQL**
- ORM: Prisma / TypeORM (рекомендуется Prisma для скорости MVP)

### Frontend
- **Next.js** (App Router)
- UI: Tailwind CSS / любой совместимый UI-kit
- i18n: `next-intl` (или аналог)

### Авторизация
- JWT: **access + refresh**
- Refresh-token хранить в httpOnly cookie (рекомендуется)

### WhatsApp OTP
- Провайдер: **Twilio WhatsApp** или **Meta WhatsApp Cloud API**
- Абстракция провайдера через интерфейс `OtpGateway`

---

## 3) Роли и права

### Student
- регистрация/вход по OTP;
- просмотр активного теста;
- старт одной попытки;
- прохождение теста;
- просмотр собственного результата.

### Admin
- создание/редактирование тестов;
- добавление вопросов;
- просмотр участников и результатов;
- экспорт Excel.

---

## 4) Функциональные требования

## 4.1 Регистрация и OTP
Поля регистрации:
- `full_name` (обязательно)
- `school` (необязательно)
- `phone_number` (обязательно, уникально)
- `language` (`ru` | `kz`)

Поток:
1. Пользователь вводит номер.
2. Система генерирует 6-значный OTP.
3. OTP отправляется в WhatsApp.
4. Пользователь вводит код.
5. Код валидируется (hash + срок + лимит попыток).
6. Создается/подтверждается аккаунт, выдается JWT.

Ограничения:
- 1 номер = 1 аккаунт;
- OTP действует 5 минут;
- resend не чаще 1 раза/60 сек;
- максимум 5 попыток ввода OTP.

## 4.2 Логика тестирования
Пользователь может стартовать тест, если:
- `now >= start_time`;
- `now <= end_time`;
- попытка еще не создана для пары (`user_id`, `test_id`).

При старте:
- создается `attempt`;
- фиксируется `start_time`;
- `finish_time = start_time + duration_minutes`.

Таймер:
- отображается на frontend;
- истинность времени проверяется **только backend**;
- при истечении времени — авто-submit.

Оценка:
- 1 правильный ответ = 1 балл;
- максимум = 30 (или по количеству вопросов теста).

## 4.3 Админ-панель
- CRUD тестов;
- CRUD вопросов;
- таблица результатов;
- сортировка:
```sql
ORDER BY score DESC, total_time_seconds ASC;
```
- экспорт Excel (`.xlsx`).

## 4.4 Мультиязычность
- тексты вопросов и тестов хранятся в 2 языках в БД;
- язык переключается на frontend;
- язык пользователя хранится в `users.language`.

## 4.5 Анти-чит
Обязательные:
- одна попытка;
- IP логирование;
- перемешивание вопросов;
- перемешивание вариантов;
- авто-submit по времени.

Дополнительные:
- фиксация смены вкладки;
- ограничение копирования текста;
- флаг подозрения при смене IP.

---

## 5) Нефункциональные требования

- До 200 одновременных пользователей.
- Средний ответ API <= 500 мс на типовых операциях.
- Индексы: `phone_number`, `test_id`, `user_id`.
- HTTPS обязательно (prod).
- Rate limiting на auth-эндпоинты.

---

## 6) Схема БД (PostgreSQL)

```sql
-- users
CREATE TABLE users (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  full_name TEXT NOT NULL,
  school TEXT,
  phone_number TEXT UNIQUE NOT NULL,
  role TEXT NOT NULL CHECK (role IN ('admin', 'student')) DEFAULT 'student',
  language TEXT NOT NULL CHECK (language IN ('ru', 'kz')) DEFAULT 'ru',
  created_at TIMESTAMP NOT NULL DEFAULT now()
);

-- otp_codes
CREATE TABLE otp_codes (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  phone_number TEXT NOT NULL,
  code_hash TEXT NOT NULL,
  expires_at TIMESTAMP NOT NULL,
  attempts INT NOT NULL DEFAULT 0,
  is_used BOOLEAN NOT NULL DEFAULT FALSE,
  created_at TIMESTAMP NOT NULL DEFAULT now()
);

-- tests
CREATE TABLE tests (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  title_ru TEXT NOT NULL,
  title_kz TEXT NOT NULL,
  start_time TIMESTAMP NOT NULL,
  end_time TIMESTAMP NOT NULL,
  duration_minutes INT NOT NULL,
  is_active BOOLEAN NOT NULL DEFAULT TRUE,
  created_by UUID NOT NULL REFERENCES users(id),
  created_at TIMESTAMP NOT NULL DEFAULT now()
);

-- questions
CREATE TABLE questions (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  test_id UUID NOT NULL REFERENCES tests(id) ON DELETE CASCADE,
  question_text_ru TEXT NOT NULL,
  question_text_kz TEXT NOT NULL,
  option_a_ru TEXT NOT NULL,
  option_b_ru TEXT NOT NULL,
  option_c_ru TEXT NOT NULL,
  option_d_ru TEXT NOT NULL,
  option_e_ru TEXT NOT NULL,
  option_a_kz TEXT NOT NULL,
  option_b_kz TEXT NOT NULL,
  option_c_kz TEXT NOT NULL,
  option_d_kz TEXT NOT NULL,
  option_e_kz TEXT NOT NULL,
  correct_option CHAR(1) NOT NULL CHECK (correct_option IN ('A','B','C','D','E'))
);

-- attempts
CREATE TABLE attempts (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID NOT NULL REFERENCES users(id),
  test_id UUID NOT NULL REFERENCES tests(id),
  start_time TIMESTAMP NOT NULL,
  finish_time TIMESTAMP NOT NULL,
  score INT NOT NULL DEFAULT 0,
  total_time_seconds INT,
  ip_address INET,
  is_submitted BOOLEAN NOT NULL DEFAULT FALSE,
  created_at TIMESTAMP NOT NULL DEFAULT now(),
  UNIQUE (user_id, test_id)
);

-- answers
CREATE TABLE answers (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  attempt_id UUID NOT NULL REFERENCES attempts(id) ON DELETE CASCADE,
  question_id UUID NOT NULL REFERENCES questions(id),
  selected_option CHAR(1) NOT NULL CHECK (selected_option IN ('A','B','C','D','E')),
  is_correct BOOLEAN NOT NULL
);

-- индексы
CREATE INDEX idx_users_phone_number ON users(phone_number);
CREATE INDEX idx_questions_test_id ON questions(test_id);
CREATE INDEX idx_attempts_user_id ON attempts(user_id);
CREATE INDEX idx_attempts_test_id ON attempts(test_id);
```

---

## 7) API контракт (MVP)

### Auth
- `POST /auth/send-otp`
  - body: `{ phone_number }`
  - response: `{ success: true, resend_in: 60 }`
- `POST /auth/verify-otp`
  - body: `{ phone_number, code, full_name?, school?, language? }`
  - response: `{ access_token, refresh_token, user }`
- `POST /auth/refresh`
  - body/cookie: refresh token
  - response: `{ access_token }`

### Student
- `GET /tests/active`
- `POST /tests/:id/start`
- `POST /tests/:id/submit`
- `GET /results/me`

### Admin
- `POST /admin/tests`
- `POST /admin/questions`
- `GET /admin/results`
- `GET /admin/export`

---

## 8) Безопасность

- OTP хранить только в виде hash (`bcrypt/argon2`).
- JWT с коротким сроком жизни access-токена.
- Refresh-token ротация.
- Ограничение запросов на `/auth/send-otp` и `/auth/verify-otp`.
- Backend-only валидация таймера и финализации попытки.
- Защита от повторного submit (идемпотентность + флаг `is_submitted`).

---

## 9) UX страницы (MVP)

1. Главная
2. Регистрация (номер телефона + профиль)
3. Ввод OTP
4. Личный кабинет
5. Страница теста (таймер, вопросы)
6. Результаты
7. Админ-панель

---

## 10) План реализации MVP (по этапам)

### Этап 1: Инфраструктура
- Поднять NestJS + PostgreSQL + Prisma.
- Настроить миграции, линтер, базовый CI.

### Этап 2: Auth/OTP
- `send-otp`, `verify-otp`, `refresh`.
- Интеграция WhatsApp провайдера.
- Ограничения resend/attempts.

### Этап 3: Тесты и попытки
- Модели `tests`, `questions`, `attempts`, `answers`.
- Старт/submit с серверной проверкой времени.
- Автоподсчет баллов.

### Этап 4: Админ
- Создание тестов и вопросов.
- Рейтинг + сортировка.
- Экспорт в Excel.

### Этап 5: Frontend
- Страницы MVP.
- i18n RU/KZ.
- Таймер на клиенте с синхронизацией с backend.

---

## 11) Критерии приемки MVP

- Регистрация и вход по WhatsApp OTP работают end-to-end.
- Один пользователь не может пройти тест более 1 раза.
- Тест завершается по таймеру даже при refresh страницы.
- Рейтинг отображается в правильной сортировке.
- Интерфейс доступен на RU и KZ.
- Базовая нагрузка 100–200 одновременных пользователей выдерживается.
