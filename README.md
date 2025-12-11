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
![С4 контейнерный уровень исправленая](https://github.com/user-attachments/assets/c79c9b68-df9c-43e5-8aec-3fcf98afeac5)

На диаграмме изображено общее представление о программном средстве.

**Компонентный уровень**
![С4 компонентный уровень](https://github.com/user-attachments/assets/5170ad6e-9a8a-4dbf-8f47-44cda07e088c)

Компонетная диаграмма является более подробным представлением контейнера "Бэкенд".

### Схема данных

<img width="2504" height="1172" alt="Untitled" src="https://github.com/user-attachments/assets/5f7092d0-39b7-43ed-93d8-9e11cdb59cf3" />

### UML-диаграммы

Диаграмма вариантов использования

![Диаграма вариантов использования](https://github.com/user-attachments/assets/30ebb07e-630a-4085-ace2-9adbabaf00cf)

Диаграмма состояний

![Диаграмма состояний](https://github.com/user-attachments/assets/0ffcc889-f61d-4984-bfaa-ff717a263582)

Диаграмма деятельности 

![Диаграмма деятельности](https://github.com/user-attachments/assets/0a0ce986-0170-466a-8214-d769744928da)

Диаграмма пакетов

![Диаграмма пакетов](https://github.com/user-attachments/assets/8f581194-e443-408e-91d2-ac4f394d059b)

Диаграмма размещения

![Диаграмма размещения](https://github.com/user-attachments/assets/d05e7f78-584f-4d48-8c84-9859fa692754)


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

Тест проверяет метод ChangeStatus, который отвечает за перевод требования из одного состояния в другое (например, Draft -> Review или Review -> Approved). Для обеспечения изоляции тестирования, зависимости от базы данных и сервиса уведомлений заменяются на Mock-объекты. Проверяются следующие сценарии:

1 Успешный переход: Переход из Draft в Review разрешен.

2 Недопустимый переход: Попытка перехода из Approved в Draft запрещена правилами Workflow.

3 Ошибка валидации: Требование не найдено в репозитории.

	type MockRequirementRepo struct {
		mock.Mock
	}
	
	func (m *MockRequirementRepo) GetByID(ctx context.Context, id int) (*models.Requirement, error) {
		args := m.Called(ctx, id)
		if args.Get(0) == nil {
			return nil, args.Error(1)
		}
		return args.Get(0).(*models.Requirement), args.Error(1)
	}
	
	func (m *MockRequirementRepo) UpdateStatus(ctx context.Context, id int, status models.RequirementStatus) error {
		args := m.Called(ctx, id, status)
		return args.Error(0)
	}
	
	func TestRequirementService_ChangeStatus_Unit(t *testing.T) {
		mockRepo := new(MockRequirementRepo)
		reqService := service.NewRequirementService(mockRepo, nil, nil) 
	
		ctx := context.Background()
		reqID := 101
	
		tests := []struct {
			name          string
			initialStatus models.RequirementStatus
			targetStatus  models.RequirementStatus
			mockBehavior  func(mockRepo *MockRequirementRepo)
			expectedError error
		}{
			{
				name:          "Success Transition: Draft -> Review",
				initialStatus: models.StatusDraft,
				targetStatus:  models.StatusReview,
				mockBehavior: func(m *MockRequirementRepo) {
					m.On("GetByID", ctx, reqID).Return(&models.Requirement{
						RequirementID: reqID, Status: models.StatusDraft, 
					}, nil).Once()
					m.On("UpdateStatus", ctx, reqID, models.StatusReview).Return(nil).Once()
				},
				expectedError: nil,
			},
			{
				name:          "Invalid Transition: Approved -> Draft",
				initialStatus: models.StatusApproved,
				targetStatus:  models.StatusDraft,
				mockBehavior: func(m *MockRequirementRepo) {
					m.On("GetByID", ctx, reqID).Return(&models.Requirement{
						RequirementID: reqID, Status: models.StatusApproved,
					}, nil).Once()
				},
				expectedError: service.ErrInvalidStatusTransition,
			{
				name:          "Requirement Not Found",
				initialStatus: models.StatusDraft,
				targetStatus:  models.StatusReview,
				mockBehavior: func(m *MockRequirementRepo) {
					m.On("GetByID", ctx, reqID).Return(nil, repository.ErrRequirementNotFound).Once()
				},
				expectedError: repository.ErrRequirementNotFound,
			},
		}
	
		for _, tt := range tests {
			t.Run(tt.name, func(t *testing.T) {
				mockRepo.Calls = nil 
				tt.mockBehavior(mockRepo)
	
				err := reqService.ChangeStatus(ctx, reqID, tt.targetStatus)
	
				if tt.expectedError != nil {
					assert.ErrorIs(t, err, tt.expectedError)
				} else {
					assert.NoError(t, err)
				}
				
				mockRepo.AssertExpectations(t) 
			})
		}
	}


### Интеграционные тесты

Интеграционный тест проверяет, как HTTP-слой обрабатывает запрос на изменение статуса требования. Тест имитирует HTTP-запрос (обычно с заголовком HX-Request: true от HTMX) и проверяет:

1 Корректный статус: Возврат 200 OK.

2 Формат ответа: Ответ должен быть HTML-фрагментом, а не JSON-объектом.

3 Корректность данных: Проверяется, что возвращенный HTML содержит новый статус требования (например, Review).


	type MockRequirementService struct {
		mock.Mock
	}
	
	func (m *MockRequirementService) ChangeStatus(ctx context.Context, id int, status models.RequirementStatus) error {
		args := m.Called(ctx, id, status)
		return args.Error(0)
	}
	
	func TestRequirementHandler_ChangeStatus_HTMX_Integration(t *testing.T) {
		mockService := new(MockRequirementService)
		mockRenderer := NewMockRenderer() 
		
		handler := handlers.NewRequirementHandler(nil, mockService, mockRenderer)
	
		router := mux.NewRouter()
		router.HandleFunc("/api/requirements/{id}/status", handler.ChangeStatus).Methods("POST")
	
		t.Run("Success Status Change via HTMX", func(t *testing.T) {
			reqID := "101"
			newStatus := "Approved"
	
			mockService.On("ChangeStatus", mock.Anything, 101, models.StatusApproved).Return(nil).Once()
			
			mockRenderer.On("RenderRequirementRow", mock.Anything, mock.Anything).Return("<tr id='req-101'><td>Approved</td></tr>", nil).Once()
	
			req, _ := http.NewRequest("POST", "/api/requirements/"+reqID+"/status?status="+newStatus, nil)
			req.Header.Set("HX-Request", "true") 
			rr := httptest.NewRecorder()
			
			router.ServeHTTP(rr, req)
	
			assert.Equal(t, http.StatusOK, rr.Code)
			assert.Equal(t, "text/html", rr.Header().Get("Content-Type"))
			assert.True(t, strings.Contains(rr.Body.String(), "<tr id='req-101'><td>Approved</td></tr>"))
			
			mockService.AssertExpectations(t)
		})
	
		t.Run("Invalid Transition", func(t *testing.T) {
			reqID := "102"
			newStatus := "Draft"
			
			mockService.On("ChangeStatus", mock.Anything, 102, models.StatusDraft).
				Return(service.ErrInvalidStatusTransition).Once()
			
			mockRenderer.On("RenderErrorFragment", mock.Anything, mock.Anything).Return("<div class='error'>Invalid status transition</div>", nil).Once()
	
			req, _ := http.NewRequest("POST", "/api/requirements/"+reqID+"/status?status="+newStatus, nil)
			req.Header.Set("HX-Request", "true") 
			rr := httptest.NewRecorder()
	
			router.ServeHTTP(rr, req)
			
			assert.Equal(t, http.StatusBadRequest, rr.Code) 
			assert.True(t, strings.Contains(rr.Body.String(), "Invalid status transition"))
	
			mockService.AssertExpectations(t)
		})
	}
	
	type MockRenderer struct {
		mock.Mock
	}
	
	func NewMockRenderer() *MockRenderer { return &MockRenderer{} }
	
	func (m *MockRenderer) RenderRequirementRow(w http.ResponseWriter, data *models.Requirement) error {
		args := m.Called(w, data)
		w.Write([]byte(args.String(0)))
		return args.Error(1)
	}
	func (m *MockRenderer) RenderErrorFragment(w http.ResponseWriter, errMsg string) error {
		args := m.Called(w, errMsg)
		w.WriteHeader(http.StatusBadRequest)	w.Write([]byte(args.String(0)))
		return args.Error(1)
	}


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
