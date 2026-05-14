## Кратко

Я получил доступ к публичному репозиторию **`jqp4/AlgoLoad-flutter-3`**, клонировал его локально для анализа и просмотрел структуру, ключевые исходники, зависимости, маршруты, сценарии пользовательского взаимодействия и точки интеграции с backend. В текущем репозитории `/workspace/AlgoLoad` действительно уже зафиксировано, что Flutter-фронтенд планируется добавить как часть проекта: `frontend/` описан как планируемый Flutter-компонент. 【F:README.md†L5-L9】

Анализ ниже относится к внешнему репозиторию:

* GitHub: <https://github.com/jqp4/AlgoLoad-flutter-3>
* Проанализированный commit: `9143178e378cf0dfc9cf4ae121c70f88fc407467`

Изменений в кодовую базу `/workspace/AlgoLoad` я не вносил, поэтому commit и PR не создавал.

---

# 1. Назначение Flutter-репозитория

Репозиторий `AlgoLoad-flutter-3` содержит **Flutter Web-приложение**, которое выступает пользовательским интерфейсом к системе AlgoLoad. По коду видно, что приложение предназначено для:

1. авторизации пользователя;
2. получения текущей задачи с backend;
3. отображения визуализации графа через встроенный WebView;
4. просмотра и редактирования исходной конфигурации графа в форматах `XML`, `JSON`, `CPP`;
5. отправки изменённой конфигурации обратно на backend;
6. загрузки конфигурационного файла с компьютера;
7. переходов к связанным backend-страницам: скачивание задачи, отчёты, админ-панель.

README самого Flutter-репозитория минимален и содержит только заголовок `AlgoLoad Flutter (3)`, поэтому основная информация восстанавливается по исходному коду, `pubspec.yaml`, web-конфигурации, nginx-документации и структуре `lib/`.

---

# 2. Общая структура репозитория

На верхнем уровне репозитория находятся:

```text
AlgoLoad-flutter-3/
├── lib/
│   ├── main.dart
│   ├── main_development.dart
│   ├── main_production.dart
│   ├── main_staging.dart
│   └── src/
│       ├── core/
│       ├── features/
│       └── foundation/
├── web/
├── tmp/
│   ├── docs/
│   ├── nginx-local-deprecated/
│   └── nginx-virtual-server/
├── pubspec.yaml
├── analysis_options.yaml
├── Makefile
├── deploy.sh
└── README.md
```

Основная бизнес-логика и UI находятся в `lib/src`. Внутри `src` проект разделён на три крупных слоя:

* `core/` — инфраструктура приложения: bootstrap, DI, routing, network, storage, theme, UI-kit;
* `features/` — функциональные модули: `auth`, `algoview`, `menu`, `notes_deprecated`, `onboarding`, `settings`, `splash`;
* `foundation/` — общие расширения и утилиты.

Такое разбиение соответствует feature-oriented архитектуре с элементами Clean Architecture: в функциональных модулях есть разделение на `domain`, `infra`, `external`, `presentation`.

---

# 3. Технологический стек

Приложение реализовано на Flutter/Dart. В `pubspec.yaml` указаны:

* Dart SDK `^3.5.3`;
* `flutter_bloc` / `bloc` — управление состоянием;
* `get_it` и `injectable` — dependency injection;
* `auto_route` — маршрутизация;
* `dio` — HTTP-клиент;
* `dartz` — `Either` для обработки ошибок;
* `file_picker` — загрузка файлов пользователем;
* `flutter_inappwebview` — встраивание AlgoView-визуализации;
* `code_text_field` — редактор исходного кода/конфигурации;
* `url_launcher` — открытие внешних URL;
* `freezed`, `json_serializable`, `build_runner` — генерация кода.

Источник: [`pubspec.yaml`](https://github.com/jqp4/AlgoLoad-flutter-3/blob/9143178e378cf0dfc9cf4ae121c70f88fc407467/pubspec.yaml#L6-L49).

---

# 4. Точка входа и bootstrap

В `lib/main.dart` основной запуск сейчас направлен на development-конфигурацию:

```dart
void main() => d.main();
```

Источник: [`lib/main.dart`](https://github.com/jqp4/AlgoLoad-flutter-3/blob/9143178e378cf0dfc9cf4ae121c70f88fc407467/lib/main.dart#L1-L7).

При этом в проекте есть отдельные entrypoint-файлы:

* `main_development.dart`;
* `main_staging.dart`;
* `main_production.dart`.

Все они вызывают общий `bootstrap`, но передают разный `AppBuildType`. Источники:

* [`main_development.dart`](https://github.com/jqp4/AlgoLoad-flutter-3/blob/9143178e378cf0dfc9cf4ae121c70f88fc407467/lib/main_development.dart#L3-L8)
* [`main_staging.dart`](https://github.com/jqp4/AlgoLoad-flutter-3/blob/9143178e378cf0dfc9cf4ae121c70f88fc407467/lib/main_staging.dart#L3-L8)
* [`main_production.dart`](https://github.com/jqp4/AlgoLoad-flutter-3/blob/9143178e378cf0dfc9cf4ae121c70f88fc407467/lib/main_production.dart#L3-L8)

Bootstrap выполняет:

1. `WidgetsFlutterBinding.ensureInitialized()`;
2. настройку DI через `configureDependencies()`;
3. запуск `BootstrapService`;
4. `runApp`.

Источник: [`bootstrap.dart`](https://github.com/jqp4/AlgoLoad-flutter-3/blob/9143178e378cf0dfc9cf4ae121c70f88fc407467/lib/src/core/bootstrap/bootstrap.dart#L12-L30).

`BootstrapService` разбивает инициализацию на стадии:

* `core`;
* `storage`;
* `connection`;
* `features`;
* `completed`.

На практике сейчас важная стадия — `connection`, где инициализируется сетевой драйвер. Источник: [`bootstrap_service.dart`](https://github.com/jqp4/AlgoLoad-flutter-3/blob/9143178e378cf0dfc9cf4ae121c70f88fc407467/lib/src/core/bootstrap/bootstrap_service.dart#L34-L100).

---

# 5. Основное приложение и маршрутизация

Корневой виджет — `MyApp`. Он создаёт `AppRouter`, настраивает тему и запускает `MaterialApp.router`.

Источник: [`app.dart`](https://github.com/jqp4/AlgoLoad-flutter-3/blob/9143178e378cf0dfc9cf4ae121c70f88fc407467/lib/src/core/app/app.dart#L4-L27).

В маршрутизаторе реально активны два маршрута:

| Маршрут | Страница | Назначение |
|---|---|---|
| `/login/` | `LoginPage` | страница авторизации, initial route |
| `/algoview/` | `AlgoViewMainPage` | основная страница работы с задачей и визуализацией |

Источник: [`app_router.dart`](https://github.com/jqp4/AlgoLoad-flutter-3/blob/9143178e378cf0dfc9cf4ae121c70f88fc407467/lib/src/core/router/app_router.dart#L7-L27).

В коде также видны закомментированные маршруты для:

* `SettingsRoute`;
* `OnboardingRoute`;
* `NotesListRoute`;
* `SingleNoteRoute`.

То есть проект содержит заделы/остатки других страниц, но в текущем routing-е они отключены. Источник: [`app_router.dart`](https://github.com/jqp4/AlgoLoad-flutter-3/blob/9143178e378cf0dfc9cf4ae121c70f88fc407467/lib/src/core/router/app_router.dart#L10-L16).

---

# 6. Dependency Injection

DI реализован через `get_it` и `injectable`.

Файл `di.dart` объявляет глобальный контейнер `getIt`, функцию `configureDependencies()` и helper `inject<T>()`. Источник: [`di.dart`](https://github.com/jqp4/AlgoLoad-flutter-3/blob/9143178e378cf0dfc9cf4ae121c70f88fc407467/lib/src/core/di/di.dart#L1-L24).

Сгенерированный `di.config.dart` регистрирует:

* `BootstrapService`;
* `NetworkDriver`;
* `AppVersionService`;
* `LoadingOverlay`;
* `SecureStorageService`;
* `IAlgoViewRemoteDataSource`;
* `IAuthRemoteDataSource`;
* репозитории и use case-ы для `auth`, `algoview`, `notes_deprecated`.

Источник: [`di.config.dart`](https://github.com/jqp4/AlgoLoad-flutter-3/blob/9143178e378cf0dfc9cf4ae121c70f88fc407467/lib/src/core/di/di.config.dart#L48-L80).

---

# 7. Сетевой слой и связь с backend

## 7.1. Базовый URL backend

Все основные API-запросы идут через `NetworkDriver`. В нём жёстко задан production/backend root:

```dart
static const String rootUrl = 'https://algoload.parallel.ru/';
```

Источник: [`network_driver.dart`](https://github.com/jqp4/AlgoLoad-flutter-3/blob/9143178e378cf0dfc9cf4ae121c70f88fc407467/lib/src/core/driver/network_driver.dart#L16-L19).

То есть Flutter-приложение сейчас завязано на backend:

```text
https://algoload.parallel.ru/
```

## 7.2. HTTP-клиент

`NetworkDriver` использует `Dio`, настраивает:

* `baseUrl`;
* redirects;
* таймауты;
* `validateStatus`;
* `withCredentials`;
* content-type `application/x-www-form-urlencoded`;
* для Flutter Web дополнительно включает `BrowserHttpClientAdapter.withCredentials = true`.

Источник: [`network_driver.dart`](https://github.com/jqp4/AlgoLoad-flutter-3/blob/9143178e378cf0dfc9cf4ae121c70f88fc407467/lib/src/core/driver/network_driver.dart#L20-L44).

Это важно для связи с backend, потому что авторизация, судя по коду, опирается на cookie/session-механизм.

## 7.3. Методы NetworkDriver

В сетевом драйвере есть методы:

* `get(String url)`;
* `post(String url, {Map<String, dynamic> body})`;
* `uploadFileFromString(...)`;
* `uploadFile(...)`.

Источник: [`network_driver.dart`](https://github.com/jqp4/AlgoLoad-flutter-3/blob/9143178e378cf0dfc9cf4ae121c70f88fc407467/lib/src/core/driver/network_driver.dart#L46-L110).

Для Flutter Web особенно важен `uploadFileFromString`, потому что он формирует `multipart/form-data` из строки и отправляет её как файл. Именно этот механизм используется для отправки конфигурации графа.

---

# 8. Авторизация

## 8.1. Backend endpoints авторизации

Модуль `auth` взаимодействует с backend через следующие endpoint-ы:

| Endpoint | Метод | Назначение |
|---|---|---|
| `/api/login` | POST | вход пользователя |
| `/api/logout` | GET | выход |
| `/api/username` | GET | получение имени текущего пользователя |

Источник: [`auth/external/remote_data_source.dart`](https://github.com/jqp4/AlgoLoad-flutter-3/blob/9143178e378cf0dfc9cf4ae121c70f88fc407467/lib/src/features/auth/external/remote_data_source.dart#L11-L210).

## 8.2. Login flow

Форма авторизации содержит поля:

* username;
* password;
* кнопка `Sign in`.

Источник: [`login_form_page.dart`](https://github.com/jqp4/AlgoLoad-flutter-3/blob/9143178e378cf0dfc9cf4ae121c70f88fc407467/lib/src/features/auth/presentation/ui/pages/login_form/login_form_page.dart#L74-L108).

При изменении полей UI отправляет события в `AuthBloc`:

* `updateUserNameForm`;
* `updatePasswordForm`;
* `submitLoginForm`.

Источник: [`login_form_page.dart`](https://github.com/jqp4/AlgoLoad-flutter-3/blob/9143178e378cf0dfc9cf4ae121c70f88fc407467/lib/src/features/auth/presentation/ui/pages/login_form/login_form_page.dart#L82-L107).

`AuthBloc` хранит `LoginForm`, вызывает use case `LoginUser`, а при успешном результате переводит состояние в `completed`. Источник: [`bloc.dart`](https://github.com/jqp4/AlgoLoad-flutter-3/blob/9143178e378cf0dfc9cf4ae121c70f88fc407467/lib/src/features/auth/presentation/controllers/bloc/bloc.dart#L53-L70).

`LoginPage` слушает состояние `completed` и перенаправляет пользователя на `AlgoViewMainRoute`. Источник: [`page_wrapper.dart`](https://github.com/jqp4/AlgoLoad-flutter-3/blob/9143178e378cf0dfc9cf4ae121c70f88fc407467/lib/src/features/auth/presentation/ui/pages/login_form/page_wrapper.dart#L15-L31).

## 8.3. Автовход

При создании `AuthBloc` сразу отправляется событие `tryAutoLogin`. Источник: [`page_wrapper.dart`](https://github.com/jqp4/AlgoLoad-flutter-3/blob/9143178e378cf0dfc9cf4ae121c70f88fc407467/lib/src/features/auth/presentation/ui/pages/login_form/page_wrapper.dart#L15-L17).

Внутри `AuthBloc` этот сценарий вызывает тот же `LoginUser`, но с текущей формой `_form`, которая изначально создаётся как `LoginForm.empty()`. Источник: [`bloc.dart`](https://github.com/jqp4/AlgoLoad-flutter-3/blob/9143178e378cf0dfc9cf4ae121c70f88fc407467/lib/src/features/auth/presentation/controllers/bloc/bloc.dart#L24-L25) и [`bloc.dart`](https://github.com/jqp4/AlgoLoad-flutter-3/blob/9143178e378cf0dfc9cf4ae121c70f88fc407467/lib/src/features/auth/presentation/controllers/bloc/bloc.dart#L72-L89).

Для диссертации это можно описать осторожно: **в текущей реализации предусмотрена попытка автоматической авторизации при открытии страницы входа, но механизм выглядит незавершённым/экспериментальным, поскольку использует тот же login use case и пустую форму**.

## 8.4. Cookie/session

В `AuthRemoteDataSourceImpl.loginUser` после успешного ответа возвращается строка `'null'`, а старый код извлечения session cookie закомментирован. Источник: [`auth/external/remote_data_source.dart`](https://github.com/jqp4/AlgoLoad-flutter-3/blob/9143178e378cf0dfc9cf4ae121c70f88fc407467/lib/src/features/auth/external/remote_data_source.dart#L36-L87).

При этом в `NetworkDriver` включён `withCredentials`, что указывает на использование browser-managed cookies. Источник: [`network_driver.dart`](https://github.com/jqp4/AlgoLoad-flutter-3/blob/9143178e378cf0dfc9cf4ae121c70f88fc407467/lib/src/core/driver/network_driver.dart#L29-L38).

Также есть `AuthInterceptor`, который умеет читать cookie из `document.cookie`, но в `NetworkDriver` его подключение сейчас закомментировано. Источник: [`network_driver.dart`](https://github.com/jqp4/AlgoLoad-flutter-3/blob/9143178e378cf0dfc9cf4ae121c70f88fc407467/lib/src/core/driver/network_driver.dart#L40-L43) и [`interceptors.dart`](https://github.com/jqp4/AlgoLoad-flutter-3/blob/9143178e378cf0dfc9cf4ae121c70f88fc407467/lib/src/core/driver/interceptors.dart#L9-L111).

Вывод: **текущая версия авторизации, вероятнее всего, полагается не на ручное хранение токена в приложении, а на cookie-сессию backend-а и `withCredentials` в браузере**.

---

# 9. Основной модуль AlgoView

Главная функциональность находится в `features/algoview`.

## 9.1. Назначение

`AlgoViewMainPage` — основная страница приложения. При открытии она:

1. запрашивает у backend текущую задачу;
2. получает исходную конфигурацию графа;
3. получает ссылки на статическую AlgoView-страницу и JSON-данные графа;
4. строит полный URL визуализации;
5. показывает WebView с визуализацией;
6. показывает редактор исходной конфигурации;
7. позволяет отправить изменённую конфигурацию обратно на backend.

Источник: [`algoview_main_page/page_wrapper.dart`](https://github.com/jqp4/AlgoLoad-flutter-3/blob/9143178e378cf0dfc9cf4ae121c70f88fc407467/lib/src/features/algoview/presentation/ui/pages/algoview_main_page/page_wrapper.dart#L23-L66).

## 9.2. Backend endpoints AlgoView

Модуль AlgoView использует два endpoint-а:

| Endpoint | Метод | Назначение |
|---|---|---|
| `/api/upload_task` | POST multipart | отправка конфигурации задачи |
| `/api/receive_task` | GET | получение результата/текущей задачи |

Источник: [`algoview/external/remote_data_source.dart`](https://github.com/jqp4/AlgoLoad-flutter-3/blob/9143178e378cf0dfc9cf4ae121c70f88fc407467/lib/src/features/algoview/external/remote_data_source.dart#L12-L79).

## 9.3. Отправка задачи

При отправке конфигурации вызывается `uploadTask`. Он отправляет файл в поле `file_data`, а также передаёт:

```text
task_code = flutter_app_submition
submit = Сгенерировать
```

Источник: [`algoview/external/remote_data_source.dart`](https://github.com/jqp4/AlgoLoad-flutter-3/blob/9143178e378cf0dfc9cf4ae121c70f88fc407467/lib/src/features/algoview/external/remote_data_source.dart#L18-L27).

В UI имя файла формируется как:

```text
flutter_app_upload.<тип>
```

где тип — `xml`, `json` или `cpp`. Источник: [`algoview_main_page/page_wrapper.dart`](https://github.com/jqp4/AlgoLoad-flutter-3/blob/9143178e378cf0dfc9cf4ae121c70f88fc407467/lib/src/features/algoview/presentation/ui/pages/algoview_main_page/page_wrapper.dart#L37-L47).

## 9.4. Получение задачи

При получении задачи `receiveTask` ожидает от backend данные, содержащие:

* `user_comment`;
* `graph_source_config`;
* `graph_source_type`;
* `algoview_static_link`;
* `json_graph_data_link`.

На их основе строится `ComplitedTask`. Источник: [`algoview/external/remote_data_source.dart`](https://github.com/jqp4/AlgoLoad-flutter-3/blob/9143178e378cf0dfc9cf4ae121c70f88fc407467/lib/src/features/algoview/external/remote_data_source.dart#L66-L78).

Полный URL для визуализации строится так:

```text
<rootUrl><algoview_static_link>?jsonGraphDataUrl=<json_graph_data_link>
```

Источник: [`algoview/external/remote_data_source.dart`](https://github.com/jqp4/AlgoLoad-flutter-3/blob/9143178e378cf0dfc9cf4ae121c70f88fc407467/lib/src/features/algoview/external/remote_data_source.dart#L66-L69).

Это важный архитектурный момент: **Flutter-приложение не рисует граф само, а встраивает готовую AlgoView-визуализацию, передавая ей URL JSON-данных графа**.

---

# 10. Отображение визуализации

Визуализация открывается через `AlgoViewWebViewContainer`, который использует `InAppWebView`.

Источник: [`algoview_webview_container.dart`](https://github.com/jqp4/AlgoLoad-flutter-3/blob/9143178e378cf0dfc9cf4ae121c70f88fc407467/lib/src/features/algoview/presentation/ui/widgets/algoview_webview_container.dart#L5-L52).

Внутри контейнера WebView получает `initialUrlRequest` с `algoViewFullUrl`. Источник: [`algoview_webview_container.dart`](https://github.com/jqp4/AlgoLoad-flutter-3/blob/9143178e378cf0dfc9cf4ae121c70f88fc407467/lib/src/features/algoview/presentation/ui/widgets/algoview_webview_container.dart#L30-L33).

Для описания в диссертации можно сформулировать так:

> Визуальный компонент Flutter-приложения выполняет роль оболочки над AlgoView: backend формирует статическую страницу визуализации и JSON-файл с данными графа, а frontend встраивает эту страницу в пользовательский интерфейс через WebView.

---

# 11. Редактор конфигурации графа

На основной странице пользователь видит:

* комментарий пользователя;
* WebView с визуализацией;
* заголовок `Graph source code`;
* переключатель типа конфигурации: `XML`, `JSON`, `CPP`;
* редактор кода с подсветкой;
* кнопки `Submit` и `Upload file`.

Источник: [`algoview_main_page/page_wrapper.dart`](https://github.com/jqp4/AlgoLoad-flutter-3/blob/9143178e378cf0dfc9cf4ae121c70f88fc407467/lib/src/features/algoview/presentation/ui/pages/algoview_main_page/page_wrapper.dart#L153-L258).

Поддерживаемые типы конфигурации описаны enum-ом:

```dart
enum GraphSourceConfigType { xml, json, cpp, unknown }
```

Источник: [`graph_source_config_type.dart`](https://github.com/jqp4/AlgoLoad-flutter-3/blob/9143178e378cf0dfc9cf4ae121c70f88fc407467/lib/src/features/algoview/domain/entities/graph_source_config_type.dart#L1).

Для подсветки используются языки `xml`, `json`, `cpp`. Источник: [`algoview_main_page/page_wrapper.dart`](https://github.com/jqp4/AlgoLoad-flutter-3/blob/9143178e378cf0dfc9cf4ae121c70f88fc407467/lib/src/features/algoview/presentation/ui/pages/algoview_main_page/page_wrapper.dart#L116-L122).

---

# 12. Загрузка файла пользователем

Сценарий `Upload file` реализован через `file_picker`.

Пользователь может выбрать файл с расширением:

* `xml`;
* `json`;
* `cpp`.

Источник: [`algoview_main_page/page_wrapper.dart`](https://github.com/jqp4/AlgoLoad-flutter-3/blob/9143178e378cf0dfc9cf4ae121c70f88fc407467/lib/src/features/algoview/presentation/ui/pages/algoview_main_page/page_wrapper.dart#L68-L83).

После выбора файл читается:

* из `bytes` для web;
* из `path` для desktop/mobile.

Источник: [`algoview_main_page/page_wrapper.dart`](https://github.com/jqp4/AlgoLoad-flutter-3/blob/9143178e378cf0dfc9cf4ae121c70f88fc407467/lib/src/features/algoview/presentation/ui/pages/algoview_main_page/page_wrapper.dart#L85-L101).

Затем содержимое файла помещается в редактор, обновляется тип конфигурации, задача отправляется на backend, после чего результат снова запрашивается. Источник: [`algoview_main_page/page_wrapper.dart`](https://github.com/jqp4/AlgoLoad-flutter-3/blob/9143178e378cf0dfc9cf4ae121c70f88fc407467/lib/src/features/algoview/presentation/ui/pages/algoview_main_page/page_wrapper.dart#L327-L347).

---

# 13. Боковое меню

В приложении есть `SideMenuScaffold`, который используется как оболочка основной страницы AlgoView. Источник: [`menu.dart`](https://github.com/jqp4/AlgoLoad-flutter-3/blob/9143178e378cf0dfc9cf4ae121c70f88fc407467/lib/src/features/menu/presentation/ui/widgets/menu.dart#L11-L35).

Меню:

1. запрашивает username через use case `GetUserName`;
2. отображает username в AppBar;
3. содержит ссылки/кнопки:
   * `AlgoLoad`;
   * `Download task`;
   * `Reports`;
   * `Admin panel`;
   * `Logout`.

Источник: [`menu.dart`](https://github.com/jqp4/AlgoLoad-flutter-3/blob/9143178e378cf0dfc9cf4ae121c70f88fc407467/lib/src/features/menu/presentation/ui/widgets/menu.dart#L73-L87) и [`menu.dart`](https://github.com/jqp4/AlgoLoad-flutter-3/blob/9143178e378cf0dfc9cf4ae121c70f88fc407467/lib/src/features/menu/presentation/ui/widgets/menu.dart#L156-L236).

Внешние URL в меню:

| Кнопка | URL |
|---|---|
| `Download task` | `https://algoload.parallel.ru/user/<username>/task` |
| `Reports` | `https://algoload.parallel.ru/upload_report` |
| `Admin panel` | `https://algoload.parallel.ru/admin` |

Источник: [`menu.dart`](https://github.com/jqp4/AlgoLoad-flutter-3/blob/9143178e378cf0dfc9cf4ae121c70f88fc407467/lib/src/features/menu/presentation/ui/widgets/menu.dart#L34-L35) и [`menu.dart`](https://github.com/jqp4/AlgoLoad-flutter-3/blob/9143178e378cf0dfc9cf4ae121c70f88fc407467/lib/src/features/menu/presentation/ui/widgets/menu.dart#L179-L216).

Открытие URL реализовано через `url_launcher`, а при ошибке используется fallback через JavaScript `window.open`. Источник: [`menu.dart`](https://github.com/jqp4/AlgoLoad-flutter-3/blob/9143178e378cf0dfc9cf4ae121c70f88fc407467/lib/src/features/menu/presentation/ui/widgets/menu.dart#L89-L118).

---

# 14. Сценарии использования

## Сценарий 1. Авторизация пользователя

1. Пользователь открывает приложение.
2. Открывается `/login/`.
3. Приложение пробует выполнить auto-login.
4. Если auto-login не удался, пользователь вводит username/password.
5. Форма отправляется на `/api/login`.
6. При успешном ответе пользователь перенаправляется на `/algoview/`.

Связанные исходники:

* [`app_router.dart`](https://github.com/jqp4/AlgoLoad-flutter-3/blob/9143178e378cf0dfc9cf4ae121c70f88fc407467/lib/src/core/router/app_router.dart#L17-L26)
* [`login_form_page.dart`](https://github.com/jqp4/AlgoLoad-flutter-3/blob/9143178e378cf0dfc9cf4ae121c70f88fc407467/lib/src/features/auth/presentation/ui/pages/login_form/login_form_page.dart#L74-L108)
* [`auth/external/remote_data_source.dart`](https://github.com/jqp4/AlgoLoad-flutter-3/blob/9143178e378cf0dfc9cf4ae121c70f88fc407467/lib/src/features/auth/external/remote_data_source.dart#L11-L39)

---

## Сценарий 2. Просмотр текущей задачи

1. Пользователь попадает на `/algoview/`.
2. В `initState` вызывается `_receiveTask()`.
3. Приложение отправляет GET `/api/receive_task`.
4. Backend возвращает конфигурацию графа, тип конфигурации, комментарий и ссылки для визуализации.
5. Flutter показывает комментарий, WebView и редактор исходной конфигурации.

Связанные исходники:

* [`algoview_main_page/page_wrapper.dart`](https://github.com/jqp4/AlgoLoad-flutter-3/blob/9143178e378cf0dfc9cf4ae121c70f88fc407467/lib/src/features/algoview/presentation/ui/pages/algoview_main_page/page_wrapper.dart#L31-L66)
* [`algoview/external/remote_data_source.dart`](https://github.com/jqp4/AlgoLoad-flutter-3/blob/9143178e378cf0dfc9cf4ae121c70f88fc407467/lib/src/features/algoview/external/remote_data_source.dart#L43-L79)

---

## Сценарий 3. Редактирование и отправка конфигурации

1. Пользователь редактирует исходный код/конфигурацию графа в CodeField.
2. При изменении текста обновляется `_newTask.graphSourceConfig`.
3. Пользователь выбирает тип `XML`, `JSON` или `CPP`.
4. Нажимает `Submit`.
5. Flutter отправляет содержимое как файл на `/api/upload_task`.
6. После отправки снова вызывает `/api/receive_task`.
7. Обновлённая визуализация отображается в WebView.

Связанные исходники:

* [`algoview_main_page/page_wrapper.dart`](https://github.com/jqp4/AlgoLoad-flutter-3/blob/9143178e378cf0dfc9cf4ae121c70f88fc407467/lib/src/features/algoview/presentation/ui/pages/algoview_main_page/page_wrapper.dart#L170-L258)
* [`algoview_main_page/page_wrapper.dart`](https://github.com/jqp4/AlgoLoad-flutter-3/blob/9143178e378cf0dfc9cf4ae121c70f88fc407467/lib/src/features/algoview/presentation/ui/pages/algoview_main_page/page_wrapper.dart#L344-L347)
* [`algoview/external/remote_data_source.dart`](https://github.com/jqp4/AlgoLoad-flutter-3/blob/9143178e378cf0dfc9cf4ae121c70f88fc407467/lib/src/features/algoview/external/remote_data_source.dart#L12-L41)

---

## Сценарий 4. Загрузка конфигурации из файла

1. Пользователь нажимает `Upload file`.
2. Выбирает `.xml`, `.json` или `.cpp`.
3. Приложение читает содержимое файла.
4. Вставляет содержимое в редактор.
5. Определяет тип конфигурации по расширению.
6. Отправляет задачу на backend.
7. Запрашивает обновлённый результат.

Связанные исходники:

* [`algoview_main_page/page_wrapper.dart`](https://github.com/jqp4/AlgoLoad-flutter-3/blob/9143178e378cf0dfc9cf4ae121c70f88fc407467/lib/src/features/algoview/presentation/ui/pages/algoview_main_page/page_wrapper.dart#L68-L113)
* [`algoview_main_page/page_wrapper.dart`](https://github.com/jqp4/AlgoLoad-flutter-3/blob/9143178e378cf0dfc9cf4ae121c70f88fc407467/lib/src/features/algoview/presentation/ui/pages/algoview_main_page/page_wrapper.dart#L327-L342)

---

## Сценарий 5. Работа с дополнительными backend-страницами

Через боковое меню пользователь может открыть:

* страницу скачивания задачи;
* страницу отчётов;
* админ-панель.

Они открываются в новой вкладке, а не как внутренние Flutter-страницы. Источник: [`menu.dart`](https://github.com/jqp4/AlgoLoad-flutter-3/blob/9143178e378cf0dfc9cf4ae121c70f88fc407467/lib/src/features/menu/presentation/ui/widgets/menu.dart#L179-L216).

---

## Сценарий 6. Выход из системы

1. Пользователь нажимает `Logout`.
2. Приложение отправляет событие logout.
3. После задержки возвращает пользователя на `LoginRoute`.

Источник: [`menu.dart`](https://github.com/jqp4/AlgoLoad-flutter-3/blob/9143178e378cf0dfc9cf4ae121c70f88fc407467/lib/src/features/menu/presentation/ui/widgets/menu.dart#L219-L230).

При этом стоит отметить техническую особенность: в меню создаётся новый `AuthBloc()` напрямую, а не используется уже существующий bloc из контекста. Это можно описать как реализационную особенность/потенциальную зону доработки.

---

# 15. Дополнительные и устаревшие модули

В репозитории есть модуль `notes_deprecated`. По названию видно, что он устаревший. Он содержит функциональность, не связанную напрямую с текущим AlgoView-сценарием:

* отправка аудиофайла;
* отправка YouTube URL;
* получение результата задачи;
* работа с заметками.

Backend endpoint-ы этого модуля:

* `/audio_task`;
* `/youtube_task`;
* `/result_task`.

Источник: [`notes_deprecated/external/remote_data_source.dart`](https://github.com/jqp4/AlgoLoad-flutter-3/blob/9143178e378cf0dfc9cf4ae121c70f88fc407467/lib/src/features/notes_deprecated/external/remote_data_source.dart#L10-L104).

Так как маршруты notes в основном router-е закомментированы, для диссертации этот модуль лучше описывать как **исторический/экспериментальный задел, не являющийся частью основного пользовательского сценария AlgoLoad**. Источник: [`app_router.dart`](https://github.com/jqp4/AlgoLoad-flutter-3/blob/9143178e378cf0dfc9cf4ae121c70f88fc407467/lib/src/core/router/app_router.dart#L10-L16).

---

# 16. Web-конфигурация и deployment

## 16.1. Base href

В `web/index.html` установлен:

```html
<base href="/app/" />
```

Источник: [`web/index.html`](https://github.com/jqp4/AlgoLoad-flutter-3/blob/9143178e378cf0dfc9cf4ae121c70f88fc407467/web/index.html#L17-L19).

Это означает, что Flutter Web-приложение предполагается размещать не в корне домена, а под путём `/app/`.

## 16.2. Web-скрипты

В `index.html` подключаются:

* `dart_print_override.js`;
* `url_launcher_web.js`.

Также задаётся `window.flutterWebRenderer = "html"` и helper `window.openUrl`. Источник: [`web/index.html`](https://github.com/jqp4/AlgoLoad-flutter-3/blob/9143178e378cf0dfc9cf4ae121c70f88fc407467/web/index.html#L41-L61).

## 16.3. Build/deploy

`deploy.sh` выполняет:

```bash
flutter build web
```

и затем копирует build на сервер по `scp`. Источник: [`deploy.sh`](https://github.com/jqp4/AlgoLoad-flutter-3/blob/9143178e378cf0dfc9cf4ae121c70f88fc407467/deploy.sh#L1-L14).

В `tmp/docs/nginx-setup.md` есть инструкция по установке nginx, клонированию репозитория и запуску nginx. Источник: [`tmp/docs/nginx-setup.md`](https://github.com/jqp4/AlgoLoad-flutter-3/blob/9143178e378cf0dfc9cf4ae121c70f88fc407467/tmp/docs/nginx-setup.md#L9-L63).

В `tmp/nginx-virtual-server/nginx.conf` описан nginx server на порту `8080`, root `/root/flutter-build-web`, fallback на `index.html`, CORS-заголовки и правила кэширования статических файлов. Источник: [`tmp/nginx-virtual-server/nginx.conf`](https://github.com/jqp4/AlgoLoad-flutter-3/blob/9143178e378cf0dfc9cf4ae121c70f88fc407467/tmp/nginx-virtual-server/nginx.conf#L41-L99).

---

# 17. Архитектурная схема для диссертации

Можно описать систему так:

```text
Пользователь
   │
   ▼
Flutter Web App
   │
   ├── Auth module
   │      ├── /api/login
   │      ├── /api/logout
   │      └── /api/username
   │
   ├── AlgoView module
   │      ├── /api/receive_task
   │      └── /api/upload_task
   │
   ├── Embedded WebView
   │      └── AlgoView static page + jsonGraphDataUrl
   │
   └── Side menu
          ├── /user/<username>/task
          ├── /upload_report
          └── /admin
```

Backend в этой архитектуре выполняет несколько ролей:

1. аутентифицирует пользователя;
2. хранит/выдаёт текущую задачу пользователя;
3. принимает изменённую конфигурацию;
4. генерирует/отдаёт данные для визуализации;
5. предоставляет отдельные web-страницы для администрирования, отчётов и скачивания задач.

Flutter frontend выполняет роль **интерактивной оболочки**:

* маршрутизация и авторизация;
* UI для редактирования конфигурации;
* загрузка файла;
* отправка задачи;
* встраивание визуализации;
* навигация к дополнительным backend-страницам.

---

# 18. Важные замечания и ограничения текущей реализации

Для диссертации полезно отделить **проектную архитектуру** от **текущего состояния кода**.

## 18.1. README Flutter-репозитория почти пустой

Документация в самом Flutter-репозитории минимальна, поэтому архитектура восстановлена по исходникам.

## 18.2. Есть незавершённые элементы

В коде много `todo`, а некоторые части выглядят как промежуточные:

* `AuthInterceptor` есть, но не подключён;
* ручное хранение session token закомментировано;
* `AlgoViewRepositoryImpl.reciveTask` не реализован;
* UI напрямую использует `IAlgoViewRemoteDataSource`, обходя use case/repository слой;
* есть опечатки в названиях: `ComplitedTask`, `ReciveTaskForm`, `reciveTask`, `submition`;
* root backend URL жёстко задан в коде;
* `development`, `staging`, `production` entrypoints есть, но полноценной env-конфигурации не видно.

Источники:

* [`network_driver.dart`](https://github.com/jqp4/AlgoLoad-flutter-3/blob/9143178e378cf0dfc9cf4ae121c70f88fc407467/lib/src/core/driver/network_driver.dart#L16-L43)
* [`algoview/infra/repository/repository.dart`](https://github.com/jqp4/AlgoLoad-flutter-3/blob/9143178e378cf0dfc9cf4ae121c70f88fc407467/lib/src/features/algoview/infra/repository/repository.dart#L24-L28)
* [`algoview_main_page/page_wrapper.dart`](https://github.com/jqp4/AlgoLoad-flutter-3/blob/9143178e378cf0dfc9cf4ae121c70f88fc407467/lib/src/features/algoview/presentation/ui/pages/algoview_main_page/page_wrapper.dart#L26-L27)
* [`algoview/domain/entities/complited_task.dart`](https://github.com/jqp4/AlgoLoad-flutter-3/blob/9143178e378cf0dfc9cf4ae121c70f88fc407467/lib/src/features/algoview/domain/entities/complited_task.dart#L3-L18)
* [`algoview/domain/entities/recive_task_form.dart`](https://github.com/jqp4/AlgoLoad-flutter-3/blob/9143178e378cf0dfc9cf4ae121c70f88fc407467/lib/src/features/algoview/domain/entities/recive_task_form.dart#L1-L2)

## 18.3. Фронтенд уже фактически интегрирован с production backend

Несмотря на наличие development/staging/prod entrypoints, сетевой слой использует `https://algoload.parallel.ru/`, то есть приложение ориентировано на конкретный backend-домен. Источник: [`network_driver.dart`](https://github.com/jqp4/AlgoLoad-flutter-3/blob/9143178e378cf0dfc9cf4ae121c70f88fc407467/lib/src/core/driver/network_driver.dart#L16-L19).

---

# 19. Готовая формулировка для главы диссертации

Можно использовать примерно такой текст:

> Клиентская часть системы AlgoLoad реализована как Flutter Web-приложение. Приложение выполняет роль интерактивной пользовательской оболочки над backend-сервисом и подсистемой визуализации AlgoView. Основной пользовательский сценарий включает авторизацию, получение текущей задачи, просмотр визуализации информационного графа алгоритма, редактирование исходной конфигурации графа и отправку изменённой конфигурации на сервер.
>
> Архитектурно frontend разделён на инфраструктурный слой и функциональные модули. Инфраструктурный слой включает bootstrap-приложения, dependency injection, маршрутизацию, сетевой драйвер, тему и набор переиспользуемых UI-компонентов. Функциональные модули включают авторизацию, основной модуль AlgoView и боковое меню. Внутри модулей используется разделение на domain, infra, external и presentation, что соответствует feature-oriented подходу с элементами Clean Architecture.
>
> Взаимодействие с backend выполняется через HTTP API. Авторизация использует endpoint-ы `/api/login`, `/api/logout` и `/api/username`. Основной модуль AlgoView использует `/api/receive_task` для получения текущей задачи и `/api/upload_task` для отправки пользовательской конфигурации. После получения задачи frontend формирует URL визуализации на основе ссылки на статическую AlgoView-страницу и ссылки на JSON-данные графа. Эта визуализация встраивается в интерфейс Flutter-приложения через WebView.
>
> Пользователь может редактировать конфигурацию графа непосредственно в браузере. Поддерживаются форматы XML, JSON и C++. Также предусмотрена загрузка конфигурационного файла с компьютера. После отправки изменённых данных backend обрабатывает задачу, а frontend повторно получает результат и обновляет визуализацию.
>
> Дополнительно приложение содержит боковое меню, через которое доступны переходы к скачиванию задачи, отчётам и административной панели backend-системы. Таким образом, Flutter-приложение объединяет сценарии авторизации, редактирования, визуализации и навигации по связанным сервисам AlgoLoad.

---

# 20. Команды, которые я использовал

**Проверки / команды**

* ✅ `rm -rf /tmp/AlgoLoad-flutter-3 && git clone --depth 1 https://github.com/jqp4/AlgoLoad-flutter-3.git /tmp/AlgoLoad-flutter-3`
* ✅ `cd /tmp/AlgoLoad-flutter-3 && git rev-parse HEAD && git remote -v`
* ✅ `cd /tmp/AlgoLoad-flutter-3 && find lib -maxdepth 6 -type f | sort`
* ✅ `cd /tmp/AlgoLoad-flutter-3 && rg -n "class |@RoutePage|AutoRoute|path:|Dio|BaseOptions|baseUrl|http|https|api|endpoint|@injectable|@lazySingleton|getIt|Bloc|Cubit|Future<|upload|login|logout|token|storage|InAppWebView|launchUrl" lib web README.md Makefile deploy.sh tmp/docs -S`
* ✅ `cd /tmp/AlgoLoad-flutter-3 && nl -ba pubspec.yaml`
* ✅ `cd /tmp/AlgoLoad-flutter-3 && nl -ba lib/src/core/router/app_router.dart`
* ✅ `cd /tmp/AlgoLoad-flutter-3 && nl -ba lib/src/core/driver/network_driver.dart`
* ✅ `cd /tmp/AlgoLoad-flutter-3 && nl -ba lib/src/features/auth/external/remote_data_source.dart`
* ✅ `cd /tmp/AlgoLoad-flutter-3 && nl -ba lib/src/features/algoview/external/remote_data_source.dart`
* ✅ `cd /tmp/AlgoLoad-flutter-3 && nl -ba lib/src/features/algoview/presentation/ui/pages/algoview_main_page/page_wrapper.dart`
* ✅ `cd /tmp/AlgoLoad-flutter-3 && nl -ba lib/src/features/menu/presentation/ui/widgets/menu.dart`
* ✅ `git status --short`
* ✅ `rg -n "AlgoLoad-flutter|flutter|frontend|front" README.md . -g 'README*' -g '!node_modules' -g '!build'`
