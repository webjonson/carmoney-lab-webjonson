В первом случае (Ask) вывел информацию что можно сделать.

Во втором случае (Code) сразу изменил файд README.md и вывел: "Готово — в README.md после таблицы команд добавлен раздел «Как проверить, что сервис жив» с командами из Makefile (make ps, make logs, make test, make lint) и curl на /health, а также пояснением про healthcheck db из docker-compose.yml."
Разрешения на изменение не спрашивал.

Агент plan, модель MiniMax M3 - долго думал, выводил reasoning и другую инфу. В итоге завершилось ошибкой: Connection reset by server. "{
  "name": "APIError",
  "data": {
    "message": "Connection reset by server",
    "isRetryable": true,
    "metadata": {
      "code": "ECONNRESET",
      "syscall": "",
      "message": "The socket connection was closed unexpectedly. For more information, pass `verbose: true` in the second argument to fetch()"
    }
  }
}"

Агент debug, модель MiniMax M3. Написал, что уже добавленно:
"Раздел «Как проверить, что сервис жив» уже добавлен в README.md (строки 34–50) во время предыдущего захода в code-агенте. Содержимое соответствует запросу:"
