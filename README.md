# Docker Compose шаблон для PHP проекта
> Шаблон основан на репозитории https://github.com/sherifabdlnaby/kubephp.
> Отдельное спасибо Паше.

## Конфигурация

| image      | version |
|------------|---------|
| PHP        | 8.3.6   |
| Xdebug     | 3.3.2   |
| Nginx      | 1.25    |
| PostgreSQL | 16      | 

## Для каких проектов подходит этот шаблон?
Шаблон подходит для проектов, у которых публичные файлы находятся папке `{путь до корня проекта}/public/`.
В этой папке должны находиться такие файлы как:
1. index.php;
2. изображения и другие медиа файлы;
3. подключаемые css, js файлы;
4. другие файлы, к которым пользователь может получить доступ.

Примеры фреймворков, которые легко можно использовать в этом шаблоне:
1. Symfony
2. Laravel
3. Slim
4. Codeigniter

<details>
<summary>Для работы Yii</summary>

Чтобы работать с Yii, необходимо заменить путь */app/public* на */app/web*
1. изменить конфигурацию nginx
   в файле *docker/web/nginx/conf.d/app.conf* в 16 строчке заменить `root $base/public;` -> `root $base/web;`
2. в compose-файле изменить volume  `${APP_BASE_DIR-.}/public:/app/public` -> `${APP_BASE_DIR-.}/web:/app/web` 
3. в файле *docker/web/Dockerfile* в 47 строчке заменить `COPY --chown=www-data:www-data --from=app /app/public /app/public` -> `COPY --chown=www-data:www-data --from=app /app/web /app/web`

> Мог где-то что-то упустить. Разворачивать Yii не пробовал.
</details>

> В частных случаях может потребоваться доработка конфигураций.

## Инструкция
1. Клонируй проект себе на машину:

   ```git clone https://github.com/DevChernykh/web-docker-template.git -b php8.3.6```

   Вместо php8.3.6 можешь задать другой доступный тэг (если такой имеется).
2. При необхоидмости можешь переименовать папку с проектом.
3. Перейди в папку с проектом:

   ```cd web-docker-template```
5. Если шаблон необходим для проекта, можешь удалить папку `.git`.
6. Создай .env на основе .env.example и укажи необходимые параметры.
7. Если будет нужен xdebug (только для разработки), то позаботься о задании значения `XDEBUG_IDE_KEY` в .env. Чтобы не пришлось потом пересобирать контейнер.
8. После, можешь воспользоваться make-командами, для работы с контейнерами (up, down, build, etc..).

> Желетаельно пользоваться именно make-командами.
>
> Если по каким-то причинам не установлен cmake и взаимодействие происходит напрямую командой `docker compose`, то перед командой необходимо добавить `DOCKER_BUILDKIT=1 COMPOSE_DOCKER_CLI_BUILD=1`.
> Особенно это касается сборки для prod. В других случаях, пока что, не обязательно.
>
> Пример команды: `DOCKER_BUILDKIT=1 COMPOSE_DOCKER_CLI_BUILD=1 docker compose up -d`.

## Настройка Xdebug (PHPStorm)
<details>
<summary>Инструкция</summary>

1. Задать значение в *.env* для *XDEBUG_IDE_KEY*. Например: *xdebug_app_key*.
   При изменении настроек, необходимо пересобрать контейнер с php.
   <hr>
2. Settings -> Build, Execution, Deployment -> Docker:

   Нажимаем "+", указываем своё имя и настраиваем подключение в зависимости от Ваших настроек Docker.
   Внизу должна появиться надпись *Connection successful*.
   <hr>
3. Settings -> PHP -> CLI Interpreter -> 3 точки:
 
   Нажимаем слева "+" -> *From Docker, Vargant ...*, выбираем пункт *Docker Compose*.

   Server -> выбираем тот, что создали на прошлом шаге.
   
   Configuration files -> выбираем *compose.yaml* из этого проекта.
   
   Service -> выбираем контейнер с PHP.

   OK

   Остальное на Ваш вкус, но в графе Lifecycle лучше оставить connect to existing container.
   <hr>
4. Settings -> PHP -> CLI Interpreter:
   
   Выбрать только что созданный конфиг.
   <hr>
5. Settings -> PHP -> Servers:

   Нажимаем "+", вводим названием (его нужно запомнить), указываем хост (127.0.0.1) и порт, который указали в *.env* для nginx.

   Включаем галочку *Use path mappings* и раскрываем папку *Projects Files*, на против *app* указываем */app*.
   <hr>
6. Settings -> PHP -> Debug:

   В блоке *Xdebug* находим поле *Debug port* и добавляем туда *9001*.
   <hr>
7. Run -> Edit configurations:
   
   Нажимаем на "+", переходим в *PHP Remote Debug*, указываем название.

   В поле Server указываем тот, что создали в п.5

   IDE key (session_id) указываем тот, что указали в *.env* в поле *XDEBUG_IDE_KEY*

После, на верхней панели, где включается debug, выбираем вариант из п.7.
Теперь можно ставить breakpoints, запускать debug и посылать запросы.
</details>
