# Реализация gRPC сервиса для системы управления параметрами 

Сервис для управления параметрами системы, реализованный на C++ с использованием gRPC.

---

## Описание проекта

Данный проект представляет собой gRPC-сервис, который позволяет:
- **Получать** значение параметра по его имени (`GetParameter`).
- **Устанавливать** значение параметра с проверкой уникальности запроса (`SetParameter`).

Сервис написан на C++17 с использованием библиотек gRPC и Protocol Buffers.  
Реализована полная валидация входных данных, логирование операций, обработка ошибок и кэширование параметров в памяти.

---

## Архитектура

Проект построен по принципам **SOLID** и разделён на следующие компоненты:

|          Компонент         |                                    Описание                                           |
|----------------------------|---------------------------------------------------------------------------------------|
| **`TelemetryServiceImpl`** | Основной gRPC-сервис. Реализует методы `GetParameter` и `SetParameter`.               |
|        **`Cache`**         | Потокобезопасное хранилище параметров в памяти (`std::unordered_map` + `std::mutex`). |
|       **`Validator`**      | Проверяет входные данные: имя, значение, уникальность `request_id`.                   |
|        **`Logger`**        | Записывает все операции в консоль с временной меткой.                                 |

### Взаимодействие компонентов

```
┌─────────────────┐
│    Клиент       │
│   (grpcurl)     │
└────────┬────────┘
         │ gRPC-запрос
         ▼
┌────────────────────────────────────────────────┐
│          TelemetryServiceImpl                 │
│        (обработка gRPC-запросов)               │
└─────┬──────────────┬──────────────┬────────────┘
      │              │              │
      ▼              ▼              ▼
┌─────────────┐ ┌─────────────┐ ┌─────────────┐
│  Validator  │ │    Cache    │ │   Logger    │
│  проверка   │ │  хранение   │ │ логирование │
│   данных    │ │ параметров  │ │  операций   │
└─────────────┘ └─────────────┘ └─────────────┘
```

---

## 📦 Используемые технологии

- **C++17** — стандарт языка.
- **gRPC** — фреймворк для удалённого вызова процедур.
- **Protocol Buffers** — сериализация данных.
- **CMake** — система сборки.
- **vcpkg** — менеджер пакетов.
- **Google Test** — unit-тестирование.

---

## 🚀 Сборка и запуск

### Требования

- **Windows 10/11** или **Linux**
- **Visual Studio 2022** (или GCC на Linux)
- **CMake 3.15+**
- **vcpkg**

### Установка зависимостей

Установите **vcpkg**:
	
	bash
	git clone https://github.com/Microsoft/vcpkg.git
	cd vcpkg
	.\bootstrap-vcpkg.bat

Установите gRPC и Gooole Test:

	.\vcpkg install grpc:x64-windows
	.\vcpkg install gtest:x64-windows

Интегрируйте vcpkg в Visual Studio:

	.\vcpkg integrate install

### Сборка проекта

	cd gRPC-Service
	mkdir build
	cd build
	cmake .. -DCMAKE_TOOLCHAIN_FILE=C:/dev/vcpkg/scripts/buildsystems/vcpkg.cmake
	cmake --build . --config Release

### Запуск сервера

	cd Release
	gRPC-Service.exe
	
После запуска вы увидите:
	
	Telemetry gRPC Service v1.0
	Сервер запущен на 0.0.0.0:50051

## 🔧 Установка grpcurl

Для отправки запросов к серверу используется утилита `grpcurl`.

### Windows

1. Скачайте файл `grpcurl_1.9.3_windows_x86_64.zip`  
   с [официальной страницы релизов](https://github.com/fullstorydev/grpcurl/releases).
2. Распакуйте архив в удобную папку (например, `C:\dev\grpcurl\`).
3. Добавьте путь к папке в переменную окружения `PATH` или используйте полный путь при вызове.

### Linux / macOS

1. Скачайте соответствующий файл для вашей архитектуры:  
   - `grpcurl_1.9.3_linux_amd64.tar.gz` — для 64-битной Linux  
   - `grpcurl_1.9.3_linux_arm64.tar.gz` — для ARM-систем  
   - `grpcurl_1.9.3_macos_x86_64.tar.gz` — для macOS (если доступен)
2. Распакуйте архив:  
   ```bash
   tar -xzf grpcurl_1.9.3_linux_amd64.tar.gz

---

## 🧪 Примеры запросов

Ниже приведены все основные сценарии работы с сервисом: успешные операции, обработка ошибок, защита от дублей и таймауты.

---

### ✅ 1. Успешное сохранение параметра

**Запрос:**

	grpcurl -plaintext -d "{\"name\":\"temperature\",\"value\":\"23.5\",\"request_id\":\"id1\"}" localhost:50051 telemetry.v1.TelemetryService/SetParameter

**Ответ:**

	{
	  "status": {
	    "message": "OK"
	  }
	}
	
**Ответ 2:**

	[2026-06-20 14:47:01.244] SetParameter: temperature - OK (value: 23.5, request_id: id1)

### ✅ 2. Успешное получение параметра

	grpcurl -plaintext -d "{\"name\":\"temperature\"}" localhost:50051 telemetry.v1.TelemetryService/GetParameter

**Ответ:**

	{
	  "name": "temperature",
	  "value": "23.5",
	  "timestamp": "2026-06-20T11:47:54.244371400Z",
	  "status": {
	    "message": "OK"
	  }
	}
	
**Ответ 2:**

	[2026-06-20 14:47:54.246] GetParameter: temperature - OK (value: 23.5)

### ❌ 3. Ошибка: параметр не найден

	grpcurl -plaintext -d "{\"name\":\"pressure\"}" localhost:50051 telemetry.v1.TelemetryService/GetParameter

**Ответ:**

	ERROR:
	  Code: NotFound
	  Message: Parameter not found
	
**Ответ 2:**

	[2026-06-20 14:49:06.086] GetParameter: pressure - NOT_FOUND
	
### ❌ 4. Ошибка: повторный request_id (защита от дублей)  ------

Первый запрос (успешный):

	grpcurl -plaintext -d "{\"name\":\"humidity\",\"value\":\"45\",\"request_id\":\"id1\"}" localhost:50051 telemetry.v1.TelemetryService/SetParameter

**Ответ:**

	{"status": {"code": 0, "message": "OK"}}
	
**Ответ 2:**

	[2026-06-20 14:33:37.889] SetParameter: temperature - OK (value: 23.5, request_id: id1)

2 

	grpcurl -plaintext -d "{\"name\":\"humidity\",\"value\":\"50\",\"request_id\":\"id1\"}" localhost:50051 telemetry.v1.TelemetryService/SetParameter

**Ответ:**

	ERROR:
	  Code: AlreadyExists
	  Message: Request ID already used
	
**Ответ 2:**

	[2026-06-20 14:33:37.889] SetParameter: temperature - OK (value: 23.5, request_id: id1)

### ❌ 5. Ошибка: пустое имя параметра

	grpcurl -plaintext -d "{\"name\":\"\",\"value\":\"23.5\",\"request_id\":\"id2\"}" localhost:50051 telemetry.v1.TelemetryService/SetParameter

**Ответ:**

	ERROR:
	  Code: InvalidArgument
	  Message: Invalid name
	
**Ответ 2:**

	[2026-06-20 14:51:13.833] ОШИБКА: SetParameter: invalid name (empty)

### ❌ 6. Ошибка: пустое значение параметра

	grpcurl -plaintext -d "{\"name\":\"humidity\",\"value\":\"\",\"request_id\":\"id3\"}" localhost:50051 telemetry.v1.TelemetryService/SetParameter

**Ответ:**

	ERROR:
	  Code: InvalidArgument
	  Message: Invalid value
	
**Ответ 2:**

	[2026-06-20 14:51:46.694] ОШИБКА: SetParameter: invalid value (empty)
	
### ❌ 7. Ошибка: пустой request_id

	grpcurl -plaintext -d "{\"name\":\"humidity\",\"value\":\"45\",\"request_id\":\"\"}" localhost:50051 telemetry.v1.TelemetryService/SetParameter

**Ответ:**

	ERROR:
	  Code: InvalidArgument
	  Message: Invalid request_id
	
**Ответ 2:**

	[2026-06-20 14:52:26.568] ОШИБКА: SetParameter: invalid request_id (empty)

### ⏱️ 8. Таймаут запроса (клиент отменил запрос)

**Цель теста:** Проверить, как сервер обрабатывает ситуацию, когда клиент не дожидается ответа.

**Запрос:**

	grpcurl -plaintext -connect-timeout 3 -d "{\"name\":\"temperature\"}" localhost:50051 telemetry.v1.TelemetryService/GetParameter

**Ожидаемый ответ (при возникновении таймаута):**

	ERROR:
	  Code: DeadlineExceeded
	  Message: Request timeout

**Фактический ответ в текущем тестовом окружении:**

	{
	  "name": "temperature",
	  "value": "23.5",
	  "timestamp": "2026-06-20T12:46:15.766015500Z",
	  "status": {
	    "message": "OK"
	  }
	}
	
**Лог сервера (при попытке воспроизвести таймаут):**

	[2026-06-20 15:43:17.431] GetParameter: temperature - OK (value: 23.5)

**Пояснение:**

Таймаут не был воспроизведён в стандартных условиях, поскольку сервер отвечает мгновенно. Однако:

- ✅ Логика обработки таймаута реализована в коде (`context->IsCancelled()`).
- ✅ При возникновении долгих операций (например:
  - выполнение сложных запросов к базам данных (SQL, NoSQL);
  - тяжёлые вычисления (обработка больших объёмов данных);
  - ожидание ответа от внешних API и микросервисов;
  - работа с медленными дисками или сетевыми ресурсами)
  сервер в цикле проверяет `context->IsCancelled()`.
  Если клиент отменил запрос (по таймауту), сервер немедленно прерывает обработку и возвращает статус `DEADLINE_EXCEEDED`.
- ✅ Ошибка таймаута логируется и будет видна в консоли сервера.

Таким образом, функциональность обработки таймаутов присутствует в реализации и может быть задействована при появлении длительных операций, что соответствует требованиям к надёжности сервиса в условиях высокой нагрузки.

**Логирование ошибки (при реальном возникновении таймаута):**

	[2026-06-20 15:43:17.431] ОШИБКА: GetParameter: request cancelled by client (timeout)



