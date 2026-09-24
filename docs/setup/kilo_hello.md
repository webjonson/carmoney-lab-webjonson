# kilo_hello

готов

1. Учебный сервис предварительной оценки заявки на заём под ПТС: принимает заявку (VIN, год, пробег, стоимость, сумма, срок), считает LTV и возвращает решение `approve` / `review` / `reject`; все данные синтетические (README.md, AGENTS.md, composer.json, kilo.jsonc).
2. Makefile: `make up` (поднять сервис и базу), `make down`, `make ps`, `make logs`, `make install`, `make test` (PHPUnit), `make lint` (`php -l`), `make seed`, `make help`; в docker-compose.yml команд не нашёл — там только сервисы `backend` (порт `${APP_PORT:-8080}`) и `db` (MySQL 8.0) с healthcheck.
3. Решение approve / review / reject считается в `backend/src/Domain/` — класс `DecisionEngine.php`, пороги берёт из `backend/config/rules.php`.

модель: training-2026-09-glm-5.3