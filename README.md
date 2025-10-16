# **Программное средство управления требованиями к изделию на всех стадиях его жизненного цикла**

Основаная цель проекта: Предоставить возможность создать требования к изделию на всех стадиях его жизненного цикла и следить за их выполнением.

Backend: https://github.com/Mor1oc/backend-managing-requirements

Frontend: https://github.com/Mor1oc/frontend-managing-requirements

---

## **Содержание**

1. [Архитектура](#Архитектура)
	1. [C4-модель](#C4-модель)
	2. [Схема данных](#Схема_данных)
2. [Функциональные возможности](#Функциональные_возможности)
	1. [Диаграмма вариантов использования](#Диаграмма_вариантов_использования)
	2. [User-flow диаграммы](#User-flow_диаграммы)
3. [Детали реализации](#Детали_реализации)
	1. [UML-диаграммы](#UML-диаграммы)
	2. [Спецификация API](#Спецификация_API)
	3. [Безопасность](#Безопасность)
	4. [Оценка качества кода](#Оценка_качества_кода)
4. [Тестирование](#Тестирование)
	1. [Unit-тесты](#Unit-тесты)
	2. [Интеграционные тесты](#Интеграционные_тесты)
5. [Установка и  запуск](#installation)
	1. [Манифесты для сборки docker образов](#Манифесты_для_сборки_docker_образов)
	2. [Манифесты для развертывания k8s кластера](#Манифесты_для_развертывания_k8s_кластера)
6. [Лицензия](#Лицензия)
7. [Контакты](#Контакты)

---
## **Архитектура**

### C4-модель

Иллюстрация и описание архитектура ПС

**Контейнерный уровень**
![С4 контейнерный уровень](https://github.com/user-attachments/assets/b917ab62-c5d6-4c0b-b3e9-81bcf458d2e6)

На диаграмме изображено общее представление программного средства, пользователи и хранение данных.

**Компонентный уровень**
![С4 компонентный уровень](https://github.com/user-attachments/assets/5170ad6e-9a8a-4dbf-8f47-44cda07e088c)

Компонетная диаграмма является более подробным представлением контейнера "Бэкенд".

### Схема данных



Описание отношений и структур данных, используемых в ПС. Также представить скрипт (программный код), который необходим для генерации БД

Скрипт генерации БД:

-- 1. Пользователи
CREATE TABLE users (
    user_id SERIAL PRIMARY KEY,
    full_name VARCHAR(255) NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL,
    department VARCHAR(100),
    role VARCHAR(50) NOT NULL
);

-- 2. Проекты 
CREATE TABLE projects (
    project_id SERIAL PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    start_date DATE,
    end_date DATE
);

-- 3. Типы требований
CREATE TABLE requirement_types (
    type_id SERIAL PRIMARY KEY,
    name VARCHAR(50) NOT NULL UNIQUE
);

-- 4. Справочники статусов
CREATE TABLE document_statuses (
    status_id SERIAL PRIMARY KEY,
    status_code VARCHAR(20) NOT NULL UNIQUE
);

CREATE TABLE requirement_statuses (
    status_id SERIAL PRIMARY KEY,
    status_code VARCHAR(20) NOT NULL UNIQUE
);

CREATE TABLE verification_statuses (
    status_id SERIAL PRIMARY KEY,
    status_code VARCHAR(20) NOT NULL UNIQUE
);

CREATE TABLE approval_statuses (
    status_id SERIAL PRIMARY KEY,
    status_code VARCHAR(20) NOT NULL UNIQUE
);

CREATE TABLE change_request_statuses (
    status_id SERIAL PRIMARY KEY,
    status_code VARCHAR(20) NOT NULL UNIQUE
);

CREATE TABLE change_order_statuses (
    status_id SERIAL PRIMARY KEY,
    status_code VARCHAR(20) NOT NULL UNIQUE
);

-- 5. Документы 
CREATE TABLE documents (
    document_id SERIAL,
    external_ref VARCHAR(100),
    title TEXT NOT NULL,
    description TEXT,
    document_type VARCHAR(50) NOT NULL,
    is_external BOOLEAN DEFAULT TRUE,
    version_number INTEGER NOT NULL DEFAULT 1,
    file_path VARCHAR(255),
    uploaded_by INTEGER NOT NULL REFERENCES users(user_id) ON DELETE SET NULL,
    uploaded_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    status_id INTEGER NOT NULL REFERENCES document_statuses(status_id),
    
    PRIMARY KEY (document_id, version_number)
);

-- 6. Связь документов и проектов (многие-ко-многим)
CREATE TABLE document_projects (
    document_id INTEGER NOT NULL,
    document_version INTEGER NOT NULL,
    project_id INTEGER NOT NULL REFERENCES projects(project_id) ON DELETE CASCADE,
    
    PRIMARY KEY (document_id, document_version, project_id),
    FOREIGN KEY (document_id, document_version)
        REFERENCES documents(document_id, version_number)
        ON DELETE CASCADE
);

-- 7. Требования (с версионированием)
CREATE TABLE requirements (
    requirement_id SERIAL,
    external_id VARCHAR(50),
    project_id INTEGER NOT NULL REFERENCES projects(project_id) ON DELETE CASCADE,
    type_id INTEGER NOT NULL REFERENCES requirement_types(type_id) ON DELETE RESTRICT,
    parent_id INTEGER, -- рекурсивная ссылка (только на ID, без версии)
    created_by INTEGER NOT NULL REFERENCES users(user_id) ON DELETE SET NULL,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    version_number INTEGER NOT NULL DEFAULT 1,
    title TEXT NOT NULL,
    description TEXT,
    source_document_id INTEGER,
    source_document_version INTEGER,
    source_clause TEXT, -- остаётся: уникальный текст (п. 4.1.2 и т.д.)
    status_id INTEGER NOT NULL REFERENCES requirement_statuses(status_id),
    verification_status_id INTEGER NOT NULL REFERENCES verification_statuses(status_id),
    is_baseline BOOLEAN DEFAULT FALSE,
    change_reason TEXT,
    changed_by INTEGER NOT NULL REFERENCES users(user_id) ON DELETE SET NULL,
    changed_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,

    PRIMARY KEY (requirement_id, version_number),
    
    FOREIGN KEY (source_document_id, source_document_version)
        REFERENCES documents(document_id, version_number)
        ON DELETE SET NULL
);

-- Рекурсивная ссылка на родительское требование
ALTER TABLE requirements
ADD CONSTRAINT fk_requirement_parent
FOREIGN KEY (parent_id) REFERENCES requirements(requirement_id) ON DELETE SET NULL;

-- 8. Согласования
CREATE TABLE approvals (
    approval_id SERIAL PRIMARY KEY,
    requirement_id INTEGER NOT NULL,
    version_number INTEGER NOT NULL,
    approver_id INTEGER NOT NULL REFERENCES users(user_id) ON DELETE CASCADE,
    status_id INTEGER NOT NULL REFERENCES approval_statuses(status_id),
    comment TEXT,
    requested_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    responded_at TIMESTAMP,

    FOREIGN KEY (requirement_id, version_number)
        REFERENCES requirements(requirement_id, version_number)
        ON DELETE CASCADE
);

-- 9. Запросы на изменение (ECR)
CREATE TABLE change_requests (
    ecr_id SERIAL PRIMARY KEY,
    title TEXT NOT NULL,
    description TEXT,
    requester_id INTEGER NOT NULL REFERENCES users(user_id) ON DELETE SET NULL,
    project_id INTEGER NOT NULL REFERENCES projects(project_id) ON DELETE CASCADE,
    status_id INTEGER NOT NULL REFERENCES change_request_statuses(status_id),
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    resolved_at TIMESTAMP
);

-- 10. Инженерные приказы (ECO)
CREATE TABLE change_orders (
    eco_id SERIAL PRIMARY KEY,
    ecr_id INTEGER NOT NULL REFERENCES change_requests(ecr_id) ON DELETE CASCADE,
    title TEXT NOT NULL,
    justification TEXT,
    assigned_to INTEGER NOT NULL REFERENCES users(user_id) ON DELETE SET NULL,
    status_id INTEGER NOT NULL REFERENCES change_order_statuses(status_id),
    effective_date DATE,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);

-- 11. Связь ECO с требованиями
CREATE TABLE eco_requirement_links (
    eco_id INTEGER NOT NULL REFERENCES change_orders(eco_id) ON DELETE CASCADE,
    requirement_id INTEGER NOT NULL,
    old_version INTEGER NOT NULL,
    new_version INTEGER NOT NULL,

    PRIMARY KEY (eco_id, requirement_id, old_version),
    
    FOREIGN KEY (requirement_id, old_version)
        REFERENCES requirements(requirement_id, version_number)
        ON DELETE RESTRICT,
    FOREIGN KEY (requirement_id, new_version)
        REFERENCES requirements(requirement_id, version_number)
        ON DELETE RESTRICT
);


---

## **Функциональные возможности**

### Диаграмма вариантов использования

Диаграмма вариантов использования и ее описание

### User-flow диаграммы

User-flow диаграмма руководителя (например, главный инженер)

<img width="7908" height="6266" alt="image" src="https://github.com/user-attachments/assets/a071047c-a463-4c8e-910f-00386aa81453" />

User-flow диаграмма специалиста (например, инженер по качеству)

<img width="4292" height="2696" alt="image" src="https://github.com/user-attachments/assets/521522f9-2d26-430b-ac4f-f64dd173130e" />

---

## **Детали реализации**

### UML-диаграммы

Представить все UML-диаграммы , которые позволят более точно понять структуру и детали реализации ПС

### Спецификация API

Представить описание реализованных функциональных возможностей ПС с использованием Open API (можно представить либо полный файл спецификации, либо ссылку на него)

### Безопасность

Описать подходы, использованные для обеспечения безопасности, включая описание процессов аутентификации и авторизации с примерами кода из репозитория сервера

### Оценка качества кода

Используя показатели качества и метрики кода, оценить его качество

---

## **Тестирование**

### Unit-тесты

Представить код тестов для пяти методов и его пояснение

### Интеграционные тесты

Представить код тестов и его пояснение

---

## **Установка и  запуск**

### Манифесты для сборки docker образов

Представить весь код манифестов или ссылки на файлы с ними (при необходимости снабдить комментариями)

### Манифесты для развертывания k8s кластера

Представить весь код манифестов или ссылки на файлы с ними (при необходимости снабдить комментариями)

---

## **Лицензия**

Этот проект лицензирован по лицензии MIT - подробности представлены в файле [[License.md|LICENSE.md]]

---

## **Контакты**

Автор: email
