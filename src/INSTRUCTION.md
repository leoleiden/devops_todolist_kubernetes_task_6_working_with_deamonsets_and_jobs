# Інструкції з розгортання та валідації Kubernetes-ресурсів для Django ToDo App

Цей документ містить інструкції з підготовки Docker образу, розгортання додатка Django ToDo, а також `DaemonSet` та `CronJob` для нього в Kubernetes кластері. Також включені кроки для валідації їхньої коректної роботи.

Перед початком переконайтеся, що у вас встановлено та налаштовано `kubectl` для роботи з вашим Kubernetes кластером, а також встановлено Docker.

## Попередні умови

1.  **Створення Docker образу Django ToDo App:**
    Переконайтеся, що ви знаходитесь в кореневій директорії вашого Django ToDo проекту, де знаходиться `Dockerfile` для вашого додатка:
    ```bash
    cd ~/devops_todolist_kubernetes_task_6_working_with_deamonsets_and_jobs/src
    ```
    Створіть образ Docker:
    ```bash
    docker build -t leoleiden/todoapp:latest .
    ```

2.  **Завантаження образу Django ToDo App до Docker Hub (або вашого реєстру):**
    Увійдіть до Docker Hub, якщо ще не робили цього:
    ```bash
    docker login
    ```
    Потім завантажте образ:
    ```bash
    docker push leoleiden/todoapp:latest
    ```

3.  **Створення Namespace:**
    Переконайтеся, що у вашому кластері існує простір імен `mateapp`. Якщо ні, створіть його:

    ```bash
    kubectl create namespace mateapp
    ```

4.  **Підготовка образу `busyboxplus:curl`:**
    Згідно з вимогами завдання, `DaemonSet` та `CronJob` використовують образ `busyboxplus:curl`. Якщо ви не використовуєте спеціалізований реєстр або цей образ не є загальнодоступним, вам потрібно переконатися, що він доступний.
    У цьому випадку, як було виявлено, успішно використовується образ `radial/busyboxplus:curl`. Вам не потрібно його будувати чи пушити, оскільки він загальнодоступний.

## Розгортання Django ToDo App та допоміжних ресурсів

Для повноцінного функціонування `DaemonSet` та `CronJob`, спочатку необхідно розгорнути сам додаток Django ToDo.

1.  **Створіть файл `todoapp-deployment.yml`** з наступним вмістом:

    ```yaml
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: todoapp-deployment
      namespace: mateapp
      labels:
        app: todoapp
    spec:
      replicas: 1 # Можете змінити кількість реплік за потребою
      selector:
        matchLabels:
          app: todoapp
      template:
        metadata:
          labels:
            app: todoapp
        spec:
          containers:
          - name: todoapp-container
            image: leoleiden/todoapp:latest
            ports:
            - containerPort: 8080 # Порт, на якому Django-додаток слухає (з Dockerfile)
            env:
            - name: DATABASE_URL # Приклад змінної для бази даних, якщо ви її використовуватимете
              value: "sqlite:///db.sqlite3" # Для продакшену зазвичай використовують зовнішню БД
            resources:
              requests:
                memory: "256Mi"
                cpu: "200m"
              limits:
                memory: "512Mi"
                cpu: "500m"
            livenessProbe: # Перевірка живості
              httpGet:
                path: /api/health
                port: 8080
              initialDelaySeconds: 15
              periodSeconds: 20
            readinessProbe: # Перевірка готовності
              httpGet:
                path: /api/ready
                port: 8080
              initialDelaySeconds: 20
              periodSeconds: 10
    ```
    Застосуйте його:
    ```bash
    kubectl apply -f todoapp-deployment.yml
    ```

2.  **Створіть файл `todoapp-service.yml`** з наступним вмістом:

    ```yaml
    apiVersion: v1
    kind: Service
    metadata:
      name: todoapp-service
      namespace: mateapp
    spec:
      selector:
        app: todoapp # Ця мітка має відповідати міткам вашого Deployment
      ports:
        - protocol: TCP
          port: 8080 # Порт, на якому сервіс доступний в кластері
          targetPort: 8080 # Порт, на якому Django-додаток слухає всередині контейнера
      type: ClusterIP # Додаток доступний тільки всередині кластера
    ```
    Застосуйте його:
    ```bash
    kubectl apply -f todoapp-service.yml
    ```

3.  **Створіть файл `daemonset.yml`** з наступним вмістом:

    ```yaml
    apiVersion: apps/v1
    kind: DaemonSet
    metadata:
      name: todoapp-daemonset
      namespace: mateapp
    spec:
      selector:
        matchLabels:
          app: todoapp-daemon
      template:
        metadata:
          labels:
            app: todoapp-daemon
        spec:
          containers:
          - name: busyboxplus-curl
            image: radial/busyboxplus:curl # Використовуємо коректний образ
            resources:
              requests:
                memory: "64Mi"
                cpu: "50m"
              limits:
                memory: "128Mi"
                cpu: "100m"
            command: ["/bin/sh", "-c"]
            args:
              # Змінено ендпоінт на /api/health для коректного health check
              - while true; do
                  curl -s http://todoapp-service:8080/api/health || echo "Curl failed";
                  sleep 5;
                done;
          restartPolicy: Always
    ```
    Застосуйте його:
    ```bash
    kubectl apply -f daemonset.yml
    ```

4.  **Створіть файл `cronjob.yml`** з наступним вмістом:

    ```yaml
    apiVersion: batch/v1
    kind: CronJob
    metadata:
      name: todoapp-health-check-cronjob
      namespace: mateapp
    spec:
      schedule: "*/4 * * * *"
      concurrencyPolicy: Allow
      successfulJobsHistoryLimit: 10
      failedJobsHistoryLimit: 5
      jobTemplate:
        spec:
          template:
            spec:
              containers:
              - name: busyboxplus-curl
                image: radial/busyboxplus:curl # Використовуємо коректний образ
                resources:
                  requests:
                    memory: "32Mi"
                    cpu: "25m"
                  limits:
                    memory: "64Mi"
                    cpu: "50m"
                command: ["curl", "-s", "http://todoapp-service:8080/api/health"]
              restartPolicy: OnFailure
    ```
    Застосуйте його:
    ```bash
    kubectl apply -f cronjob.yml
    ```

## Валідація рішення

Після розгортання ви можете перевірити коректність роботи всіх компонентів.

### Валідація Django ToDo App

1.  **Перевірте статус Deployment:**

    ```bash
    kubectl get deployment todoapp-deployment -n mateapp
    ```
    Вивід логів:
    ```
    NAME                 READY   UP-TO-DATE   AVAILABLE   AGE
    todoapp-deployment   1/1     1            1           40s
    ```

2.  **Перевірте поди Django ToDo App:**

    ```bash
    kubectl get pods -l app=todoapp -n mateapp
    ```
    Вивід логів:
    ```
    NAME                                  READY   STATUS    RESTARTS   AGE
    todoapp-deployment-7858c74f46-kgzhr   1/1     Running   0          71s
    ```

3.  **Перевірте Service Django ToDo App:**

    ```bash
    kubectl get service todoapp-service -n mateapp
    ```
    Вивід логів:
    ```
    NAME              TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
    todoapp-service   ClusterIP   10.109.55.170   <none>        8080/TCP   32s
    ```

4.  **Перегляньте логи поду Django ToDo App:**
    Знайдіть ім'я вашого поду Django (`kubectl get pods -l app=todoapp -n mateapp`) та перегляньте його логи:
    ```bash
    kubectl logs todoapp-deployment-7858c74f46-kgzhr -n mateapp
    ```
    Ви повинні побачити логи запуску Django сервера.

### Валідація DaemonSet

1.  **Перевірте статус DaemonSet:**

    ```bash
    kubectl get daemonset -n mateapp
    ```
    Вивід логів:
    ```
    NAME                DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
    todoapp-daemonset   1         1         1       1            1           <none>          44m
    ```

2.  **Перевірте поди DaemonSet:**

    ```bash
    kubectl get pods -l app=todoapp-daemon -n mateapp
    ```
    Вивід логів:
    ```
    NAME                      READY   STATUS    RESTARTS   AGE
    todoapp-daemonset-4dm9c   1/1     Running   0          <X>m
    ```

3.  **Перегляньте логи поду DaemonSet:**
    Знайдіть ім'я вашого поду DaemonSet (`kubectl get pods -l app=todoapp-daemon -n mateapp`) та перегляньте його логи:
    ```bash
    kubectl logs <daemonset-pod-name> -n mateapp
    ```
    **Вивід логів:**
    ```
    Health OK
    ```
    *Примітка: Якщо вивід не з'являється одразу, спробуйте почекати кілька секунд або перевірте доступність сервісу `todoapp-service` безпосередньо з поду DaemonSet за допомогою `kubectl exec -it <daemonset-pod-name> -n mateapp -- /bin/sh` та виконавши `curl http://todoapp-service:8080/api/health`.*
**Вивід логів:**
    ```
    Health OK
### Валідація CronJob

1.  **Перевірте статус CronJob:**

    ```bash
    kubectl get cronjob -n mateapp
    ```
    Вивід логів:
    ```
    NAME                          SCHEDULE      SUSPEND   ACTIVE   LAST SCHEDULE   AGE
    todoapp-health-check-cronjob  */4 * * * *   False     4        2m19s           20m
    ```
    (`ACTIVE` може бути 0, 1 або більше, якщо Job виконується в момент перевірки. `LAST SCHEDULE` покаже, коли він востаннє запускався.)

2.  **Перевірте події CronJob (для підтвердження запусків):**

    ```bash
    kubectl get events -n mateapp | grep todoapp-health-check-cronjob
    ```
    **Вивід подій (показує успішні створення та завершення Jobs):**
````    
13m         Normal    SuccessfulCreate          job/todoapp-health-check-cronjob-1751008560         Created pod: todoapp-health-check-cronjob-1751008560-qvtb7
9m          Normal    Scheduled                 pod/todoapp-health-check-cronjob-1751008800-xz4qz   Successfully assigned mateapp/todoapp-health-check-cronjob-1751008800-xz4qz to docker-desktop
8m59s       Normal    Pulled                    pod/todoapp-health-check-cronjob-1751008800-xz4qz   Container image "radial/busyboxplus:curl" already present on machine
8m59s       Normal    Created                   pod/todoapp-health-check-cronjob-1751008800-xz4qz   Created container busyboxplus-curl
8m58s       Normal    Started                   pod/todoapp-health-check-cronjob-1751008800-xz4qz   Started container busyboxplus-curl
9m1s        Normal    SuccessfulCreate          job/todoapp-health-check-cronjob-1751008800         Created pod: todoapp-health-check-cronjob-1751008800-xz4qz
8m54s       Normal    Completed                 job/todoapp-health-check-cronjob-1751008800         Job completed
5m          Normal    Scheduled                 pod/todoapp-health-check-cronjob-1751009040-rt2hl   Successfully assigned mateapp/todoapp-health-check-cronjob-1751009040-rt2hl to docker-desktop
4m58s       Normal    Pulled                    pod/todoapp-health-check-cronjob-1751009040-rt2hl   Container image "radial/busyboxplus:curl" already present on machine
4m58s       Normal    Created                   pod/todoapp-health-check-cronjob-1751009040-rt2hl   Created container busyboxplus-curl
4m58s       Normal    Started                   pod/todoapp-health-check-cronjob-1751009040-rt2hl   Started container busyboxplus-curl
5m1s        Normal    SuccessfulCreate          job/todoapp-health-check-cronjob-1751009040         Created pod: todoapp-health-check-cronjob-1751009040-rt2hl
4m53s       Normal    Completed                 job/todoapp-health-check-cronjob-1751009040         Job completed
59s         Normal    Scheduled                 pod/todoapp-health-check-cronjob-1751009280-sxl97   Successfully assigned mateapp/todoapp-health-check-cronjob-1751009280-sxl97 to docker-desktop
58s         Normal    Pulled                    pod/todoapp-health-check-cronjob-1751009280-sxl97   Container image "radial/busyboxplus:curl" already present on machine
58s         Normal    Created                   pod/todoapp-health-check-cronjob-1751009280-sxl97   Created container busyboxplus-curl
58s         Normal    Started                   pod/todoapp-health-check-cronjob-1751009280-sxl97   Started container busyboxplus-curl
60s         Normal    SuccessfulCreate          job/todoapp-health-check-cronjob-1751009280         Created pod: todoapp-health-check-cronjob-1751009280-sxl97
53s         Normal    Completed                 job/todoapp-health-check-cronjob-1751009280         Job completed
25m         Normal    SuccessfulCreate          cronjob/todoapp-health-check-cronjob                Created job todoapp-health-check-cronjob-1751007840
21m         Normal    SuccessfulCreate          cronjob/todoapp-health-check-cronjob                Created job todoapp-health-check-cronjob-1751008080
17m         Normal    SuccessfulCreate          cronjob/todoapp-health-check-cronjob                Created job todoapp-health-check-cronjob-1751008320
13m         Normal    SuccessfulCreate          cronjob/todoapp-health-check-cronjob                Created job todoapp-health-check-cronjob-1751008560
9m1s        Normal    SuccessfulCreate          cronjob/todoapp-health-check-cronjob                Created job todoapp-health-check-cronjob-1751008800
8m51s       Normal    SawCompletedJob           cronjob/todoapp-health-check-cronjob                Saw completed job: todoapp-health-check-cronjob-1751008800, status: Complete       
5m1s        Normal    SuccessfulCreate          cronjob/todoapp-health-check-cronjob                Created job todoapp-health-check-cronjob-1751009040
4m51s       Normal    SawCompletedJob           cronjob/todoapp-health-check-cronjob                Saw completed job: todoapp-health-check-cronjob-1751009040, status: Complete       
60s         Normal    SuccessfulCreate          cronjob/todoapp-health-check-cronjob                Created job todoapp-health-check-cronjob-1751009280
50s         Normal    SawCompletedJob           cronjob/todoapp-health-check-cronjob                Saw completed job: todoapp-health-check-cronjob-1751009280, status: Complete
   ````

3.  **Отримайте логи завершеного Job CronJob:**
    Оскільки Jobs можуть завершуватися швидко і зникати зі списку `kubectl get jobs`, вам потрібно знайти завершений под, який був створений CronJob.

    Спочатку виведіть логи подій:
    ```bash
    kubectl $ kubectl get events -n mateapp --field-selector involvedObject.name=todoapp-health-check-cronjob
    ```
    
````
LAST SEEN   TYPE     REASON             OBJECT                                 MESSAGE
51m         Normal   SuccessfulCreate   cronjob/todoapp-health-check-cronjob   Created job todoapp-health-check-cronjob-1751007840
47m         Normal   SuccessfulCreate   cronjob/todoapp-health-check-cronjob   Created job todoapp-health-check-cronjob-1751008080
42m         Normal   SuccessfulCreate   cronjob/todoapp-health-check-cronjob   Created job todoapp-health-check-cronjob-1751008320
38m         Normal   SuccessfulCreate   cronjob/todoapp-health-check-cronjob   Created job todoapp-health-check-cronjob-1751008560
34m         Normal   SuccessfulCreate   cronjob/todoapp-health-check-cronjob   Created job todoapp-health-check-cronjob-1751008800
34m         Normal   SawCompletedJob    cronjob/todoapp-health-check-cronjob   Saw completed job: todoapp-health-check-cronjob-1751008800, status: Complete
30m         Normal   SuccessfulCreate   cronjob/todoapp-health-check-cronjob   Created job todoapp-health-check-cronjob-1751009040
30m         Normal   SawCompletedJob    cronjob/todoapp-health-check-cronjob   Saw completed job: todoapp-health-check-cronjob-1751009040, status: Complete
26m         Normal   SuccessfulCreate   cronjob/todoapp-health-check-cronjob   Created job todoapp-health-check-cronjob-1751009280
26m         Normal   SawCompletedJob    cronjob/todoapp-health-check-cronjob   Saw completed job: todoapp-health-check-cronjob-1751009280, status: Complete
22m         Normal   SuccessfulCreate   cronjob/todoapp-health-check-cronjob   Created job todoapp-health-check-cronjob-1751009520
22m         Normal   SawCompletedJob    cronjob/todoapp-health-check-cronjob   Saw completed job: todoapp-health-check-cronjob-1751009520, status: Complete
18m         Normal   SuccessfulCreate   cronjob/todoapp-health-check-cronjob   Created job todoapp-health-check-cronjob-1751009760
18m         Normal   SawCompletedJob    cronjob/todoapp-health-check-cronjob   Saw completed job: todoapp-health-check-cronjob-1751009760, status: Complete
2m57s       Normal   SuccessfulCreate   cronjob/todoapp-health-check-cronjob   (combined from similar events): Created job todoapp-health-check-cronjob-1751010720
14m         Normal   SawCompletedJob    cronjob/todoapp-health-check-cronjob   Saw completed job: todoapp-health-check-cronjob-1751010000, status: Complete
10m         Normal   SawCompletedJob    cronjob/todoapp-health-check-cronjob   Saw completed job: todoapp-health-check-cronjob-1751010240, status: Complete
6m47s       Normal   SawCompletedJob    cronjob/todoapp-health-check-cronjob   Saw completed job: todoapp-health-check-cronjob-1751010480, status: Complete
2m47s       Normal   SawCompletedJob    cronjob/todoapp-health-check-cronjob   Saw completed job: todoapp-health-check-cronjob-1751010720, status: Complete
````

Потім перегляньте логи одного з цих подів:
```bash
$ kubectl logs todoapp-health-check-cronjob-1751010720-kt9zn -n mateapp
```
**Вивід логів:**
    ```
    Health OK
    ```