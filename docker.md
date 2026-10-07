

### **1. Введение в контейнеризацию и Docker**

**Основы контейнеризации:**

* **Что такое контейнеры?** (различие с виртуализацией, преимущества контейнеров)
* **Что такое Docker?** (Docker Engine, Docker Images, Docker Containers, Docker CLI)
* **Контейнеры vs Виртуальные машины**: разница и преимущества.

**Рекомендуемые ресурсы:**

* Официальная документация Docker: [https://docs.docker.com/get-started/](https://docs.docker.com/get-started/)
* Книга "Docker для разработчиков" (можно найти онлайн бесплатно или в бумажной версии)

### **2. Основные компоненты Docker**

**Знакомство с ключевыми компонентами Docker:**

* **Docker Images** — что это такое, как создавать, работать с реестрами образов (например, Docker Hub).
* **Docker Containers** — что такое контейнеры, как запускать, останавливать и управлять ими.
* **Docker Volumes** — работа с томами для хранения данных между контейнерами.
* **Docker Networks** — создание и настройка сетей для контейнеров.

**Рекомендуемые ресурсы:**

* Docker CLI Cheatsheet (обзор команд Docker): [https://dockerlabs.collabnix.com/docker/cheat-sheet/](https://dockerlabs.collabnix.com/docker/cheat-sheet/)

### **3. Основные команды Docker и создание контейнеров**

**Изучение Docker CLI и команд для работы с контейнерами:**

* **docker build**, **docker run**, **docker ps**, **docker stop**, **docker exec**, **docker logs**, **docker rm**, **docker images**.
* Создание и запуск простых контейнеров, работа с интерактивными контейнерами.

**Рекомендуемые ресурсы:**

* Официальный учебник по Docker: [https://www.docker.com/resources/what-container](https://www.docker.com/resources/what-container)

### **4. Создание собственных образов с использованием Dockerfile**

**Основы создания образов:**

* **Что такое Dockerfile?** Как писать Dockerfile для создания образов.
* Обзор инструкций Dockerfile: **FROM**, **RUN**, **COPY**, **WORKDIR**, **CMD**, **EXPOSE**, **ENTRYPOINT**, **ENV**.
* Многослойная сборка (multi-stage builds) — для оптимизации и уменьшения размера образов.

**Рекомендуемые ресурсы:**

* [Dockerfile Reference](https://docs.docker.com/engine/reference/builder/)

### **5. Как запускать сервисы с помощью Docker Compose**

**Docker Compose для работы с многоконтейнерными приложениями:**

* **Что такое Docker Compose?** Как описать многоконтейнерные приложения в файле `docker-compose.yml`.
* Сервисы, тома, сети в Compose.
* Запуск нескольких контейнеров (например, frontend и backend сервисы, базы данных).

**Рекомендуемые ресурсы:**

* Официальная документация Docker Compose: [https://docs.docker.com/compose/](https://docs.docker.com/compose/)

### **6. Отладка приложений при работе в Docker**

**Как отлаживать контейнеры и приложения:**

* Логи контейнеров с помощью **docker logs**.
* Взаимодействие с контейнерами через **docker exec**.
* Использование **docker-compose logs** для отладки многоконтейнерных приложений.
* Настройка перехвата ошибок в контейнерах.

**Рекомендуемые ресурсы:**

* [Docker Debugging Documentation](https://docs.docker.com/config/containers/logging/)

### **7. Публикация образов на Docker Hub**

**Как загружать образы в Docker Hub и работать с реестрами:**

* Создание аккаунта на Docker Hub.
* Загрузка собственных образов в Docker Hub.
* Использование публичных и приватных образов.

**Рекомендуемые ресурсы:**

* [Docker Hub Official Guide](https://docs.docker.com/docker-hub/)

### **8. Работа с базами данных и томами**

**Подключение томов и баз данных в Docker:**

* Использование томов для сохранения данных контейнеров.
* Подключение баз данных (например, **MySQL**, **PostgreSQL**) к контейнерам.
* Работа с данными в многоконтейнерных приложениях (например, с использованием Docker Compose).

**Рекомендуемые ресурсы:**

* Официальная документация Docker Volumes: [https://docs.docker.com/storage/volumes/](https://docs.docker.com/storage/volumes/)

### **9. Применение на практике (примеры)**

* **Time App**: Разворачиваем многоконтейнерное приложение с базой данных и фронтендом.
* Создание Dockerfile для frontend и backend сервисов.
* Создание и настройка `docker-compose.yml` для запуска приложения.

### **10. Kubernetes (инструмент для оркестрации контейнеров)**

**Основы Kubernetes:**

* **Что такое Kubernetes?** Введение в Kubernetes и его компоненты.
* Разница между Docker и Kubernetes.
* Основные объекты Kubernetes: Pods, Deployments, Services, Namespaces.
* Как развернуть приложение в Kubernetes: манифесты и настройки для pod'ов.

**Рекомендуемые ресурсы:**

* [Официальная документация Kubernetes](https://kubernetes.io/docs/tutorials/kubernetes-basics/)
* [Kubernetes Up and Running (книга)](https://www.amazon.com/Kubernetes-Up-Running-Deploy-Applications/dp/1098110203)

### **11. Дополнительные темы и инструменты**

* **CI/CD для Docker**: Настройка интеграции Docker с Jenkins, GitLab CI, GitHub Actions.
* **Podman**: Альтернатива Docker для управления контейнерами (с открытым исходным кодом).
* **Helm**: Пакетный менеджер для Kubernetes, который помогает управлять приложениями в Kubernetes.
* **Docker на сервере и в продакшн-среде**: Настройка Docker на серверах, использование Docker в продакшн-средах.

### **12. Применение на практике в разработке**

* Разворачивай реальные приложения, тестируй их с использованием Docker и Kubernetes.
* Применяй полученные знания для решения реальных задач, таких как создание и развертывание микросервисов, настройка масштабируемых и отказоустойчивых систем.

### **Заключение:**

Этот роадмап предлагает структуру, чтобы последовательно пройти путь от основ Docker и Compose до более сложных тем, таких как Kubernetes и CI/CD для контейнерных приложений. Начни с изучения базовых компонентов и инструментов, затем переходи к более продвинутым темам и практическим проектам.
