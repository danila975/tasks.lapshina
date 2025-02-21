
# описание

для запуска системы нужны сервисы docker и docker-compose. я запускал на опирационной системе debian 12. для первоначального создания всех сервисов в папке с проектом нужно исполнить команду:

docker-compose up

эта команда запустит файл docker-compose.yml, который находится в папке проекта.

дальше переходим в контейнер с базой источником данных, в моём случае он называется mongo1 ис помощью команды:

docker exec -it mongo1 /bin/bash

теперь, находясь в контейнере, добавляем коннектор, который будет считывать данные командой:

curl -X POST \
     -H "Content-Type: application/json" \
     --data '
     {"name": "mongo-source",
      "config": {
         "connector.class":"com.mongodb.kafka.connect.MongoSourceConnector",
         "connection.uri":"mongodb://mongo1:27017/?replicaSet=rs0",
         "database":"test",
         "collection":"users",
         "pipeline":"[{\"$match\": {\"operationType\": \"insert\"}}]"
         }
     }
     ' \
     http://connect:8083/connectors -w "\n"

дальше если всё проходит без ошибок с помощью команды:

exit

закрываем контейнер с источником данных и такой же командой как и в контейнер с источником данных входим в контейнер получатель данных, который называется mongo2. там добавляем коннектор, считывающий данные с kafka такой командой:

curl -X POST \
     -H "Content-Type: application/json" \
     --data '
     {"name": "mongo-sink",
      "config": {
         "connector.class":"com.mongodb.kafka.connect.MongoSinkConnector",
         "connection.uri":"mongodb://mongo2:27017/?replicaSet=rs0",
         "database":"test",
         "collection":"users",
         "topics":"test.users",
         "change.data.capture.handler": "com.mongodb.kafka.connect.sink.cdc.mongodb.ChangeStreamHandler"
         }
     }
     ' \
     http://connect:8083/connectors -w "\n"

если всё прошло без ошибок, то закрываем контейнер получателя базы данных и переходим в контейнер источник данных. в нём подключаемся к mongo командой:

mongosh mongodb://mongo1:27017/?replicaSet=rs0

дальше работаем как с обычной mongodb базой.
для проверки получения данных открываем той же командой как и источник получатель данных, только вместо mongo1 используем mongo2.