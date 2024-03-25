# Домашнее задание к занятию «Troubleshooting» - `Мальцев Виктор`

---

Задание. При деплое приложение web-consumer не может подключиться к auth-db. Необходимо это исправить

    1. Установить приложение по команде:
        kubectl apply -f https://raw.githubusercontent.com/netology-code/kuber-homeworks/main/3.5/files/task.yaml

    2. Выявить проблему и описать.
    3. Исправить проблему, описать, что сделано.
    4. Продемонстрировать, что проблема решена.

Ответ:

1. Приложение установлено

    [alt text](https://github.com/vmmaltsev/screenshot/blob/main/Screenshot_170.png)

    [alt text](https://github.com/vmmaltsev/screenshot/blob/main/Screenshot_171.png)

2. Так как приложение web-consumer не может подключиться, то необходимо посмотреть логи этого пода командой 

    ```

    kubectl logs <POD_NAME> -n web
    
    ```

    [alt text](https://github.com/vmmaltsev/screenshot/blob/main/Screenshot_172.png)

    Сообщение об ошибке, показанное на вашем скриншоте, говорит о том, что под web-consumer не может разрешить имя хоста auth-db. Это обычно означает, что в кластере Kubernetes не работает разрешение DNS для сервиса auth-db.

3. Поскольку ошибка говорила о том, что web-consumer не может разрешить имя хоста auth-db, проблема скорее всего в DNS разрешении внутри кластера Kubernetes. 
Сервисы в Kubernetes создают DNS записи, которые позволяют подам обращаться к сервисам по имени.

Возможные причины проблемы и способы их решения:

    1. Проблема с DNS: Возможно, в кластере есть проблемы с сервисом CoreDNS, который отвечает за DNS разрешение внутри кластера. Вам нужно проверить статус и логи подов CoreDNS. Если выявлены проблемы, может потребоваться перезапустить поды CoreDNS.

        [alt text](https://github.com/vmmaltsev/screenshot/blob/main/Screenshot_173.png)

        Поды CoreDNS находятся в статусе Running

        [alt text](https://github.com/vmmaltsev/screenshot/blob/main/Screenshot_174.png)

        Видно сообщение лога от CoreDNS, которое говорит: "Still waiting on: 'kubernetes'". Это может означать, что CoreDNS еще не завершил инициализацию и ожидает готовности некоторых компонентов Kubernetes, чтобы начать обработку DNS-запросов.

        

    2. Тестирование DNS-запросов внутри кластера:

        Запуск временного пода и теста разрешения DNS-имя сервиса auth-db изнутри кластера, чтобы убедиться, что CoreDNS правильно отвечает на запросы

        kubectl run --namespace default -i --tty --rm debug --image=busybox --restart=Never -- sh

        nslookup auth-db.data.svc.cluster.local

        [alt text](https://github.com/vmmaltsev/screenshot/blob/main/Screenshot_175.png)

        На предоставленном скриншоте видно, что выполнение команды nslookup для auth-db.data.svc.cluster.local успешно возвращается с IP-адресом сервиса auth-db. Это подтверждает, что DNS-разрешение внутри кластера работает правильно.

Теперь, когда подтверждено, что DNS работает корректно, следующим шагом будет убедиться, что под web-consumer использует правильное полное DNS-имя для доступа к сервису auth-db.

Проверка переменных окружения пода web-consumer:
Проверка, что в переменных окружения, которые использует web-consumer для подключения к auth-db, указан полный DNS-адрес сервиса.


kubectl describe pod <имя_пода_web-consumer> -n web

Проверка раздела Environment Variables для переменных, связанных с подключением к базе данных.

[alt text](https://github.com/vmmaltsev/screenshot/blob/main/Screenshot_176.png)

Можно видеть, что внутри контейнера выполняется команда curl auth-db, которая пытается обратиться к auth-db, не используя полное доменное имя. Исходя из результатов nslookup, который вы выполнили ранее, полное доменное имя для auth-db должно быть auth-db.data.svc.cluster.local.

Вот почему команда curl внутри web-consumer не может разрешить имя хоста auth-db — она не использует полное доменное имя, требуемое для доступа к сервисам в других namespaces в Kubernetes.

Чтобы решить эту проблему, нужно обновить команду в манифесте пода web-consumer (или в его конфигурации запуска, если она задается где-то еще), чтобы использовать полное доменное имя auth-db.data.svc.cluster.local вместо просто auth-db.

После того как внесены изменения, нужно будет применить обновленный манифест в вашем кластере с помощью kubectl apply и убедиться, что под web-consumer пересоздается, чтобы применить новую конфигурацию.

Для корректировки манифеста деплоймента web-consumer и изменения команды в контейнере выполните следующие шаги:

1. Измените манифест деплоймента:
    Сначала вам нужно получить манифест деплоймента в формате YAML:

    kubectl get deployment web-consumer -n web -o yaml > web-consumer-deployment.yaml

    Это сохранит текущую конфигурацию деплоймента в файл web-consumer-deployment.yaml.

2. Редактирование файла:
    Открываем web-consumer-deployment.yaml в текстовом редакторе и найдем раздел containers, который содержит команду curl.

    Измените команду:
    Измените команду на следующую, чтобы использовать полное доменное имя:

```
command:
- sh
- -c
- while true; do curl auth-db.data.svc.cluster.local; sleep 5; done

```

Применение изменения:
После редактирования файла сохраните его и примените изменения:

    kubectl apply -f web-consumer-deployment.yaml


Проверка, что под пересоздался:


kubectl get pods -n web

Проверка логов нового пода:
После пересоздания пода проверьте его логи, чтобы убедиться, что ошибка разрешения DNS исправлена:


kubectl logs <новое_имя_пода> -n web
Замените <новое_имя_пода> на имя пода, которое вы получили после выполнения команды kubectl get pods -n web.

[alt text](https://github.com/vmmaltsev/screenshot/blob/main/Screenshot_177.png)