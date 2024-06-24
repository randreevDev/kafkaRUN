# kafkaRUN

Запуск стенда осуществляется командами:
````
sudo docker-compose build
sudo docker-compose up
````

Подключиться к контейнерам можно командой:
````
sudo docker-compose exec kafka-1 /bin/bash
sudo docker-compose zookeeper-1 /bin/bash
````

Файлы контейнеров сохраняются в разделе докера.

Kafka располагает рядом утилит для ее администрирования, ниже приведены примеры использования этих команд.
#### Список топиков
````
kafka-topics.sh --list --bootstrap-server localhost:9092
````

#### Информации о топике
Для партиций выводится брокер лидер для партиции(*Leader*),
брокеры на которых размещены реплики партиций(*Replicas*) и отсинхронизированные реплики(*Isr*).
````
kafka-topics.sh --describe --topic test_topic --bootstrap-server localhost:9092
````

#### Информация о партиции
````
kafka-log-dirs.sh --bootstrap-server localhost:9092 --topic-list test_topic --describe
````

#### Создание топика
Если не указано количество партиций и фактор репликации то их значения будут браться из дефолтов заданных в файле конфигурации.  
Также при создании для топика можно переопределить значения конфигурации, например размер сегмента или ретеншен.
````
kafka-topics.sh --create --topic 'Название топика' --bootstrap-server localhost:9092 --partitions 'количество партиций' --replication-factor 'количество реплик'
````

#### Изменение настроек топика
````
kafka-configs.sh --bootstrap-server localhost:9092 --alter --entity-type topics --entity-name topic_test --add-config retention.ms=604800000
````
Применять изменения нужно на 1 любом брокере. Изменения применятся ко всему кластеру.
*ВАЖНО\! Изменения применяются динамически и сразу же. Ничего перезагружать не нужно.*

#### Удаление топика
````
kafka-topics.sh --delete --topic topic_test --bootstrap-server localhost:9092
````
#### Настройки топика
Не все параметры для существующих топиков можно уменьшать, количество партиций для существующего топика уменьшить невозможно никак.
#### Добавить/Изменить
````
kafka-configs.sh --alter --bootstrap-server localhost:9092 --entity-type topics --entity-name test_topic --add-config segment.bytes='999999'
````
#### Удалить настройки
````
kafka-configs.sh --bootstrap-server localhost:9092 --alter --entity-type topics --entity-name test_topic--delete-config segment.bytes
````

#### Изменение фактора репликации
Изменить реплипликаю как простую настройку нельзя.
Cоздаем фаил increase-replication-factor.json
````
cat <<EOF> increase-replication-factor.json
{"version":1,
  "partitions":[
     {"topic":"test_topic","partition":0,"replicas":[1,2]},
     {"topic":"test_topic","partition":1,"replicas":[1,2]},
     {"topic":"test_topic","partition":2,"replicas":[1,2]},
     {"topic":"test_topic","partition":3,"replicas":[1,2]},
     {"topic":"test_topic","partition":4,"replicas":[1,2]},
     {"topic":"test_topic","partition":5,"replicas":[1,2]}
]}
EOF
````
и применяем
````
kafka-reassign-partitions.sh --zookeeper localhost:2181 --reassignment-json-file increase-replication-factor.json --execute
````

#### Чтение топика
Для просмотра в реальном времени сообщений, которые идут в топиках есть специальная утилита _kafka-console-consumer.sh_
просмотр не будет деструктивным и сообщения останутся в топике и будут доступны консумерам из других групп.

Пример, будет отображать все поступающие сообщения с момента запуска:
````
kafka-console-consumer.sh --topic test_topic --bootstrap-server localhost:9092 --property print.key=true 
````

Также можно посмотреть все сохраненные сообщения с начала топика, для этого нужно указать ключ _from-beginning_
````
kafka-console-consumer.sh --topic test_topic --from-beginning --bootstrap-server localhost:9092 --property print.key=true 
````

Помимо чтения всего топика можно выполнить чтение только конкретной партиции для этого нужно указать ключ _partition_ и номер партиции
````
kafka-console-consumer.sh --topic test_topic --bootstrap-server localhost:9092 --partition 3 --property print.key=true 
````

Еще можно производить чтение с определенной партиции и конкретного смешения в ней.
````
kafka-console-consumer.sh --topic test_topic --bootstrap-server localhost:9092 --partition 3 --offset 3 --property print.key=true 
````

#### Просмотр консумеров топика
С помощью утилиты можно посмотреть кто сейчас читает кафку и статистику по читателям.
Для этого сначала нужно получить список активных читателей (групп консумеров)
````
./kafka-consumer-groups.sh  --list --bootstrap-server localhost:9092
````

Затем запросить статистику по конкретной группе из списка
````
./kafka-consumer-groups.sh --describe --group test_topic_group --bootstrap-server localhost:9092
````
В результате по каждому читателю входящему в группу будет возвращена информация о теме и конкретной партиции которую он читает,
последнем прочитанном смещении в партиции, максимальном смещении партиции и разнице между ними (LAG), плюс информация идентифицирующая читателя.

Тут стоит уточнить что некоторые драйвера ЯП читаю из топик не через группы консюмеров, а сначала у кластера запрашиваю
всех брокеров и партиции на них и потом к каждой партиции каждого брокера устанавливают отдельное соединение(так допустим делаю
некоторые драйверы по erlang)

#### Добавление нового брокера
Самостоятельно kafka старается не заниматься распределением партиций по брокерам(хотя это как настроить).
При создании нового топика партиции равномерно распределяются по имеющимся в наличии брокерам.
При вводе новых брокеров партиции уже имеющихся топиков остаются на старых брокерах и для полноценного ввода в работу новых брокеров,
партиции существующих тем нужно перераспределить.
Для этого нужно создать JSON файл с новым маппингом партиций и скормить его кафке.
У кафки есть утилиты упрощающие этот процесс. Порядок действий при добавлении брокеров:

1. Создать JSON файл в котором перечислить топики, партиции которых нужно перераспределить. Пример файла:
````
cat <<EOF> test.json
{
  "topics": [
    {
      "topic": "test_topic"
    }
  ],
  "version": 1
}
EOF
````
2. Сгенерировать новый маппинг партиций по брокерам, для топиков перечисленных в файле.
````
./kafka-reassign-partitions.sh --bootstrap-server localhost:9092 --broker-list 1,2 --topics-to-move-json-file test.json --generate > mapping.json && cat mapping.json
````
узнать номер брокер можно вот такой командой
````
kafka-log-dirs.sh --bootstrap-server localhost:9092 --topic-list test_topic --describe |grep '^{' | jq . |grep broker
````
3. В результирующем файле будет две секции, сначала текущий маппинг(чтобы если что откатится можно было), затем новый.
   По этому нужно вырезать из него только вторую часть, например так:
````
cat mapping.json | grep version | tail -n 1 > news_mapping.json
````

4. Запустить процесс переноса партиций с файлом нового мэппинга
````
./kafka-reassign-partitions.sh --bootstrap-server localhost:9092 --reassignment-json-file news_mapping.json --execute
````

5. Процесс переноса выполняется в фоне и может быть не быстрым, в зависимости от текущей нагрузки и размера партиций.
   Текущее состояние прогресса можно контролировать запуская:
````
./kafka-reassign-partitions.sh --bootstrap-server localhost:9092 --reassignment-json-file news_mapping.json --verify
````

Данная процедура может использоваться не только при подключении нового брокера, но и например, для балансировки нагрузки.
Файлы мэппинга можно редактировать в ручную и например переносить нагруженные партиции на свободных брокеров и т.п.
В пакете _kafka-tools_ реализовано несколько способов перебалансировки партиций на основе размера и т.п. с помощью этого механизма.
