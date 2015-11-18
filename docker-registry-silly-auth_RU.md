# Реестр Docker со слабой (silly) аутентификацией

В настоящее время docker поддерживает два способа [аутентификации][auth]:

* silly
* tocken

>Аутентификация на основе токена позволяет разделить реестр docker 
и систему аутентификации/авторизации. 
Данный способ обеспечивает повышенный уровень безопасности и рекомендован 
к использованию в боевой среде.

Подробнее этот способ описан в [документации][token auth].  

>**Слабая аутентификация предназначена только для тестирования и разработки**. 
Данный способ разрешает запрос при наличии заголовка `Authorization` 
в HTTP запросе. Содержимое заголовка при этом не проверяется. 
Если же заголовок отсутствует, сервер отправляет ответ 401, 
realm и сервис.

Все выглядит предельно просто. Установим реестр и попробуем поработать с ним, 
используя тип аутентификации silly.  
Воспользуемся [документацией][deploying] с официального сайта docker:  

1. [Установим docker][installation];
2. Сконфигурируем docker для работы с **небезопасным** реестром:  

  */etc/default/docker*:
  ```
  ...
  DOCKER_OPTS="$DOCKER_OPTS --insecure-registry 0.0.0.0:5555"
  ...
  ```

  Перезапускаем docker:

  ```bash
  sudo service docker restart
  ```

3. Запустим тестовый контейнер:

  ```bash
  docker run hello-world
  ```

4. Сконфигурируем docker реестр для использования слабой (silly) аутентификации. 

  */opt/config.yml*:  
  ```yaml
  version: 0.1
  log:
    level: debug
    fields:
      service: registry
  storage:
    cache:
      blobdescriptor: inmemory
      layerinfo: inmemory
    filesystem:
      rootdirectory: /registry-data
    maintenance:
      uploadpurging:
        enabled: true
        age: 168h
        interval: 24h
        dryrun: false
  auth:
    silly:
      realm: http://0.0.0.0
      service: registry
  http:
    addr: :5555
    secret: dhfjkahfjhafa
  ```

  Где:  

  | Параметр   | Описание                              |  
  | ---------: | :---------                            |  
  | realm      | Адрес сервера аутентификации          |  
  | service    | Имя аутентифицируемого сервиса        |  
  | secret     | Случайная последовательность символов |  

5. Запускаем официальный контейнер реестра - [registry:2.0]:

  ```bash
  docker run -d -p 0.0.0.0:5555:5555 \
    -v /opt/config.yml:/go/src/github.com/docker/distribution/cmd/registry/config.yml:ro \
    registry:2.0
  ```

6. Переименуем тестовый контейнер, 
добавив к имени префикс в виде адреса нашего приватного репозитория:  

  ```bash
  docker tag hello-world:latest 0.0.0.0:5555/hello:latest
  ```

7. И попытаемся отправить его на сервер:  

  ```bash
  docker push 0.0.0.0:5555/hello:latest
  ```

  И тут реестр docker выдает ошибку:  

  >The push refers to a repository [0.0.0.0:5555/hello] (len: 1) 
91c95931e552: Image push failed 
FATA[0000] Error pushing to registry: 
Get http://0.0.0.0?account=test&scope=repository%3Ahello%3Apull%2Cpush&service=registry: 
dial tcp 0.0.0.0:80: connection refused

  Так и есть, сервер не отвечает по 80 порту. 
  Попробовав указать порт 5555 в файле конфигурации, мы получим ту же ошибку.  

8. Попробуем разобраться. Выполним запрос к реестру docker с помощью утилиты curl:  

  ```bash
  curl -v -XGET http://0.0.0.0:5555/v2/
  ```

  И ... получим *401 Unauthorized*. Но так и должно быть, 
  ведь заголовок не содержит поля *Authorization*.  
  Следующий запрос должен вернуть успешный ответ:  

  ```bash
  curl -v -XGET -H 'Authorization: ;' http://0.0.0.0:5555/v2/
  ```

9. Чтобы заставить docker отправлять нужный нам заголовок *Authorization*, 
нужно эту самую аутентификацию пройти:  

  ```bash
  docker login -e dummy@example.com -u dummy -p dummy 0.0.0.0:5555
  ```

  И снова ошибка:  

  >FATA[0000] Error response from daemon: 
no successful auth challenge for http://0.0.0.0:5555/v2/ - errors: 
[Get http://0.0.0.0?account=dummy&service=registry: 
dial tcp 0.0.0.0:80: connection refused] 

  Как видно, реестр отправляет запрос на адрес realm, 
  пытаясь аутентифицировать пользователя, и ждет ответ следующего вида:  

  ```json
  {"token": "eyJ0eXAiOiJKV1QiLCJhbG9t3w"}
  ```

10. Таким образом, нам необходим сервис, отдающий token по запросу на адрес realm. 
Я воспользовался контейнером [shinymayhem/micro-token-gen] - 
простым token-генератором, написанным на node.js 
(можно использовать любой аналогичный контейнер):  
  
  ```bash
  docker run -d -p 5556:80 shinymayhem/micro-token-gen
  ```

  Обновим */opt/config.yml*:

  ```yaml
  auth:
    silly:
      realm: http://0.0.0.0:5556
      service: registry
  ```

11. Снова запускаем контейнер (убив и удалив перед этим старый):

  ```bash
  docker run -d -p 0.0.0.0:5555:5555 \
    -v /opt/config.yml:/go/src/github.com/docker/distribution/cmd/registry/config.yml:ro \
    registry:2.0
  ```

12. Пробуем залогиниться в реестр:  

  ```bash
  docker login -e dummy@example.com -u dummy -p dummy 0.0.0.0:5555
  ```

  И получаем успешный ответ сервера:

  >Login Succeeded

13. Отправляем переименованный на шаге **6** образ в реестр:  

  ```bash
  docker push 0.0.0.0:5555/hello:latest
  ```

  Ура, успешный ответ сервера:  

  >The push refers to a repository [0.0.0.0:5555/hello] (len: 1)
91c95931e552: Image already exists 
a8219747be10: Image successfully pushed 
Digest: sha256:e9b3ec20992c6aa7b77787cab87cdaaeb3df7fc997431376ef0fac5d78307e97

14. Теперь удаляем локальный образ:  

  ```bash
  docker rmi hello-world:latest
  ```

15. И запускаем образ из нашего приватного репозитория:  

  ```bash
  docker run 0.0.0.0:5555/hello
  ```

16. **Не используйте этот способ аутентификации в боевой среде!**  
Для надежного способа аутентификации используйте один из следующих способов:  

  * [token + .htpasswd][htpasswd]
  * [token + ldap][h3nrik/registry-ldap-auth]
  * [nginx proxy + ldap][h3nrik/nginx-ldap]

[auth]: https://docs.docker.com/registry/configuration/#auth "configuration: auth"
[token auth]: https://docs.docker.com/registry/spec/auth/token/ "auth: token"
[installation]: https://docs.docker.com/installation/#installation "install docker"
[deploying]: https://docs.docker.com/registry/deploying/ "deploying docker registry"
[shinymayhem/micro-token-gen]: https://registry.hub.docker.com/u/shinymayhem/micro-token-gen/ "node.js token generator"
[htpasswd]: http://container-solutions.com/2015/04/running-secured-docker-registry-2-0/ "token + .htpasswd"
[h3nrik/registry-ldap-auth]: https://registry.hub.docker.com/u/h3nrik/registry-ldap-auth/ "token + ldap"
[h3nrik/nginx-ldap]: https://registry.hub.docker.com/u/h3nrik/nginx-ldap/ "nginx proxy + ldap"
[registry:2.0]: https://registry.hub.docker.com/_/registry/ "registry:2.0"
