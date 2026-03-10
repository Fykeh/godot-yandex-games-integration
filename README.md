# Интеграция Яндекс.Игр с Godot 4.6

Данное руководство описывает полный процесс интеграции Яндекс.Игр с Godot 4.6, включая настройку SDK, обработку событий паузы/возобновления и управление звуком при сворачивании браузера.

## 📋 Содержание
1. [Настройка экспорта Godot](#1-настройка-экспорта-godot)
2. [Установка плагина Mist1351/yandex-games-sdk](#2-установка-плагина-mist1351yandex-games-sdk)
3. [Создание файла инициализации yandex-init.js](#3-создание-файла-инициализации-yandex-initjs)
4. [Настройка index.html](#4-настройка-indexhtml)
5. [Глобальный менеджер звука (альтернативный метод)](#5-глобальный-менеджер-звука-альтернативный-метод)
6. [Проверка работы](#6-проверка-работы)
7. [Часто задаваемые вопросы](#7-часто-задаваемые-вопросы)

---

## 1. Настройка экспорта Godot

Первым делом необходимо настроить проект для корректной работы в вебе:

1. Откройте **Project → Export**
2. Создайте или выберите шаблон **Web**
3. Перейдите на вкладку **"Options"** (или **"Возможности"**)
4. В разделе **"Custom Features"** добавьте строку: `yandex`
5. Сохраните настройки

> **Зачем это нужно:** Флаг `yandex` активирует специфичные для платформы функции в плагине.

---

## 2. Установка плагина Mist1351/yandex-games-sdk

### 2.1. Скачивание плагина
Скачайте плагин с официального репозитория:  
👉 [Mist1351/yandex-games-sdk](https://github.com/Mist1351/yandex-games-sdk)

### 2.2. Установка
Распакуйте архив и скопируйте папку `addons` в корневую папку вашего проекта.

### 2.3. Настройка автозагрузки
**Важно:** Плагин НЕ нужно включать в настройках проекта через **Project Settings → Plugins**. Вместо этого:

1. Откройте **Project → Project Settings → Autoload**
2. Нажмите на папку и выберите файл:  
   `addons/godot-yandex-games-sdk/yandex_games_sdk.gd`
3. В поле **"Node Name"** оставьте `YandexSDK`
4. Нажмите **"Add"**

![Autoload настройка](https://example.com/path-to-image.jpg) *(добавьте скриншот позже)*

Порядок автозагрузки должен быть таким:
- YandexSDK
- (другие ваши скрипты)

---

## 3. Создание файла инициализации `yandex-init.js`

Создайте файл `yandex-init.js` в корневой папке вашего веб-экспорта (рядом с `index.html`).

### Полный код `yandex-init.js`:

```javascript
// yandex-init.js
console.log('🚀 yandex-init.js загружен');

// Конфигурация
const INIT_CONFIG = {
    maxRetries: 5,
    retryInterval: 1000,
    fullscreenElement: document.getElementById("canvas")
};

let sdkInitialized = false;
let retryCount = 0;

/**
 * Функция для повторной попытки инициализации с задержкой
 */
function retryInit() {
    if (retryCount < INIT_CONFIG.maxRetries) {
        retryCount++;
        console.log(`⏳ Повторная попытка инициализации (${retryCount}/${INIT_CONFIG.maxRetries})...`);
        setTimeout(initYandexSDK, INIT_CONFIG.retryInterval);
    } else {
        console.error('❌ Превышено максимальное количество попыток инициализации SDK');
    }
}

/**
 * Функция для отправки событий в Godot
 */
function notifyGodot(eventName) {
    console.log(`📢 Отправляем событие в Godot: ${eventName}`);

    // Вызываем колбэки Godot
    if (window.YandexSDK) {
        if (eventName === 'game_api_pause' && window.YandexSDK._pause_callback) {
            window.YandexSDK._pause_callback();
            console.log('✅ Вызван YandexSDK._pause_callback()');
            return true;
        }
        if (eventName === 'game_api_resume' && window.YandexSDK._resume_callback) {
            window.YandexSDK._resume_callback();
            console.log('✅ Вызван YandexSDK._resume_callback()');
            return true;
        }
    }

    // Fallback на старые методы
    if (window.godotPauseCallback && eventName === 'game_api_pause') {
        window.godotPauseCallback();
        console.log('✅ Вызван godotPauseCallback');
        return true;
    }
    if (window.godotResumeCallback && eventName === 'game_api_resume') {
        window.godotResumeCallback();
        console.log('✅ Вызван godotResumeCallback');
        return true;
    }

    console.log(`⚠️ Не удалось найти функцию для вызова Godot (событие: ${eventName})`);
    return false;
}

/**
 * Основная функция инициализации SDK
 */
async function initYandexSDK() {
    if (sdkInitialized) {
        console.log('ℹ️ SDK уже инициализирован, пропускаем');
        return;
    }

    console.log('📡 Инициализация Yandex SDK...');

    try {
        // Ждём полной загрузки DOM
        await new Promise(resolve => {
            if (document.readyState === 'complete') {
                resolve();
            } else {
                window.addEventListener('load', resolve, { once: true });
            }
        });

        // Проверяем доступность YaGames
        if (typeof YaGames === 'undefined') {
            console.warn('⏳ YaGames не найден, повтор через 1с...');
            retryInit();
            return;
        }

        // Проверяем наличие canvas элемента
        if (!INIT_CONFIG.fullscreenElement) {
            console.warn('⚠️ Элемент canvas не найден, fullscreen может не работать');
        }

        // Инициализируем SDK
        const initParams = {
            screen: {
                fullscreen: {
                    enabled: true,
                    fullscreenElement: INIT_CONFIG.fullscreenElement
                }
            }
        };

        const ysdk = await YaGames.init(initParams);

        // Сохраняем глобально
        window.ysdk = ysdk;
        sdkInitialized = true;
        retryCount = 0;

        console.log('✅ Yandex SDK инициализирован успешно!');

        // Сообщаем платформе, что игра готова
        if (ysdk.features && ysdk.features.LoadingAPI) {
            ysdk.features.LoadingAPI.ready();
            console.log('✅ LoadingAPI.ready() отправлен');
        }

        // Отмечаем начало игрового процесса
        if (ysdk.features && ysdk.features.GameplayAPI) {
            ysdk.features.GameplayAPI.start();
            console.log('🎮 GameplayAPI.start() вызван');
        }

        // Подписываемся на события платформы
        ysdk.on('game_api_pause', () => {
            console.log('⏸️ Получено событие game_api_pause от платформы');
            notifyGodot('game_api_pause');
        });

        ysdk.on('game_api_resume', () => {
            console.log('▶️ Получено событие game_api_resume от платформы');
            notifyGodot('game_api_resume');
        });

        console.log('✅ Подписка на события выполнена успешно');
        console.log('📊 Версия SDK:', ysdk.environment?.app?.version || 'не определена');

    } catch (error) {
        console.error('❌ Ошибка инициализации Yandex SDK:', error);
        if (!sdkInitialized) {
            retryInit();
        }
    }
}

// Запускаем инициализацию
if (document.readyState === 'loading') {
    document.addEventListener('DOMContentLoaded', initYandexSDK);
} else {
    initYandexSDK();
}

// Экспортируем функцию на случай ручного вызова
window.initYandexSDK = initYandexSDK;
```

###4. Настройка index.html
Файл index.html должен подключать скрипты в правильном порядке:

```<!DOCTYPE html>
<!DOCTYPE html>
<html lang="en">
	<head>
		<meta charset="utf-8">
		<meta name="viewport" content="width=device-width, user-scalable=no, initial-scale=1.0">
		<!-- Яндекс SDK -->
		<script src="/sdk.js"></script>
		
		<!-- Наш скрипт инициализации -->
		<script src="yandex-init.js"></script>
		<title>Space Mayhem</title>
		<style>
html, body, #canvas {
	margin: 0;
	padding: 0;
	border: 0;
}

body {
	color: white;
	background-color: black;
	overflow: hidden;
	touch-action: none;
}

#canvas {
	display: block;
}

#canvas:focus {
	outline: none;
}

#status, #status-splash, #status-progress {
	position: absolute;
	left: 0;
	right: 0;
}

#status, #status-splash {
	top: 0;
	bottom: 0;
}

#status {
	background-color: #242424;
	display: flex;
	flex-direction: column;
	justify-content: center;
	align-items: center;
	visibility: hidden;
}

#status-splash {
	max-height: 100%;
	max-width: 100%;
	margin: auto;
}

#status-splash.show-image--false {
	display: none;
}

#status-splash.fullsize--true {
	height: 100%;
	width: 100%;
	object-fit: contain;
}

#status-splash.use-filter--false {
	image-rendering: pixelated;
}

#status-progress, #status-notice {
	display: none;
}

#status-progress {
	bottom: 10%;
	width: 50%;
	margin: 0 auto;
}

#status-notice {
	background-color: #5b3943;
	border-radius: 0.5rem;
	border: 1px solid #9b3943;
	color: #e0e0e0;
	font-family: 'Noto Sans', 'Droid Sans', Arial, sans-serif;
	line-height: 1.3;
	margin: 0 2rem;
	overflow: hidden;
	padding: 1rem;
	text-align: center;
	z-index: 1;
}
		</style>
		<link id="-gd-engine-icon" rel="icon" type="image/png" href="index.icon.png" />
<link rel="apple-touch-icon" href="index.apple-touch-icon.png"/>

	</head>
	<body>
		<canvas id="canvas">
			Your browser does not support the canvas tag.
		</canvas>

		<noscript>
			Your browser does not support JavaScript.
		</noscript>

		<div id="status">
			<img id="status-splash" class="show-image--true fullsize--true use-filter--true" src="index.png" alt="">
			<progress id="status-progress"></progress>
			<div id="status-notice"></div>
		</div>

		<script src="index.js"></script>
		<script>
const GODOT_CONFIG = {"args":[],"canvasResizePolicy":2,"emscriptenPoolSize":8,"ensureCrossOriginIsolationHeaders":true,"executable":"index","experimentalVK":false,"fileSizes":{"index.pck":56942884,"index.wasm":35739700},"focusCanvas":true,"gdextensionLibs":[],"godotPoolSize":4};
const GODOT_THREADS_ENABLED = false;
const engine = new Engine(GODOT_CONFIG);

(function () {
	const statusOverlay = document.getElementById('status');
	const statusProgress = document.getElementById('status-progress');
	const statusNotice = document.getElementById('status-notice');

	let initializing = true;
	let statusMode = '';

	function setStatusMode(mode) {
		if (statusMode === mode || !initializing) {
			return;
		}
		if (mode === 'hidden') {
			statusOverlay.remove();
			initializing = false;
			return;
		}
		statusOverlay.style.visibility = 'visible';
		statusProgress.style.display = mode === 'progress' ? 'block' : 'none';
		statusNotice.style.display = mode === 'notice' ? 'block' : 'none';
		statusMode = mode;
	}

	function setStatusNotice(text) {
		while (statusNotice.lastChild) {
			statusNotice.removeChild(statusNotice.lastChild);
		}
		const lines = text.split('\n');
		lines.forEach((line) => {
			statusNotice.appendChild(document.createTextNode(line));
			statusNotice.appendChild(document.createElement('br'));
		});
	}

	function displayFailureNotice(err) {
		console.error(err);
		if (err instanceof Error) {
			setStatusNotice(err.message);
		} else if (typeof err === 'string') {
			setStatusNotice(err);
		} else {
			setStatusNotice('An unknown error occurred.');
		}
		setStatusMode('notice');
		initializing = false;
	}

	const missing = Engine.getMissingFeatures({
		threads: GODOT_THREADS_ENABLED,
	});

	if (missing.length !== 0) {
		if (GODOT_CONFIG['serviceWorker'] && GODOT_CONFIG['ensureCrossOriginIsolationHeaders'] && 'serviceWorker' in navigator) {
			let serviceWorkerRegistrationPromise;
			try {
				serviceWorkerRegistrationPromise = navigator.serviceWorker.getRegistration();
			} catch (err) {
				serviceWorkerRegistrationPromise = Promise.reject(new Error('Service worker registration failed.'));
			}
			// There's a chance that installing the service worker would fix the issue
			Promise.race([
				serviceWorkerRegistrationPromise.then((registration) => {
					if (registration != null) {
						return Promise.reject(new Error('Service worker already exists.'));
					}
					return registration;
				}).then(() => engine.installServiceWorker()),
				// For some reason, `getRegistration()` can stall
				new Promise((resolve) => {
					setTimeout(() => resolve(), 2000);
				}),
			]).then(() => {
				// Reload if there was no error.
				window.location.reload();
			}).catch((err) => {
				console.error('Error while registering service worker:', err);
			});
		} else {
			// Display the message as usual
			const missingMsg = 'Error\nThe following features required to run Godot projects on the Web are missing:\n';
			displayFailureNotice(missingMsg + missing.join('\n'));
		}
	} else {
		setStatusMode('progress');
		engine.startGame({
			'onProgress': function (current, total) {
				if (current > 0 && total > 0) {
					statusProgress.value = current;
					statusProgress.max = total;
				} else {
					statusProgress.removeAttribute('value');
					statusProgress.removeAttribute('max');
				}
			},
		}).then(() => {
			setStatusMode('hidden');
		}, displayFailureNotice);
	}
}());
		</script>
	</body>
</html>
```
Важно: Порядок скриптов критичен!

###5. Глобальный менеджер звука (альтернативный метод)
Вместо использования сигналов SDK для управления звуком при сворачивании браузера, мы используем более надежный метод — отслеживание события visibilitychange напрямую через JavaScriptBridge.

##5.1. Создайте файл GlobalSoundManager.gd:
```extends Node

# Для хранения колбэков
var _visibility_callback: JavaScriptObject

func _ready():
    print("🌍 Инициализация отслеживания видимости страницы")
    
    # Проверяем, что мы в вебе
    if not OS.has_feature("web"):
        print("❌ Не веб-платформа")
        return
    
    # Создаем колбэк для события видимости
    _visibility_callback = JavaScriptBridge.create_callback(_on_visibility_change)
    
    # Добавляем обработчик visibilitychange через JavaScript
    JavaScriptBridge.eval("""
        function setupVisibilityListener() {
            document.addEventListener('visibilitychange', function() {
                console.log('👁️ visibilitychange: ' + document.visibilityState);
                if (window._godot_visibility_callback) {
                    window._godot_visibility_callback(document.visibilityState);
                }
            });
            console.log('✅ Слушатель visibilitychange установлен');
        }
        setupVisibilityListener();
    """, true)
    
    # Сохраняем колбэк в глобальной переменной JavaScript
    JavaScriptBridge.get_interface("window")._godot_visibility_callback = _visibility_callback
    
    print("🌍 Отслеживание видимости настроено")

func _on_visibility_change(args):
    var state = args[0] as String
    print("🌍 Изменение видимости: ", state)
    
    var master_index = AudioServer.get_bus_index("Master")
    if master_index == -1:
        print("❌ Шина Master не найдена")
        return
    
    if state == "hidden":
        print("🔇 Страница скрыта - выключаем звук")
        AudioServer.set_bus_mute(master_index, true)
    elif state == "visible":
        print("🔊 Страница видима - включаем звук")
        AudioServer.set_bus_mute(master_index, false)
```
##5.2. Добавьте в автозагрузку:
    1. Откройте Project → Project Settings → Autoload
    2. Добавьте GlobalSoundManager.gd после YandexSDK

Правильный порядок автозагрузки:
1. YandexSDK
2. GlobalSoundManager
3. (остальные ваши скрипты)

###6. Проверка работы
##6.1. Что должно быть в консоли при запуске:
```🚀 yandex-init.js загружен
📡 Инициализация Yandex SDK...
✅ Yandex SDK инициализирован успешно!
✅ LoadingAPI.ready() отправлен
🎮 GameplayAPI.start() вызван
✅ Подписка на события выполнена успешно
🌍 Инициализация отслеживания видимости страницы
🌍 Отслеживание видимости настроено
```
##6.2. При сворачивании браузера:
```👁️ visibilitychange: hidden
🌍 Изменение видимости: hidden
🔇 Страница скрыта - выключаем звук
```
##6.3. При разворачивании браузера:
```
👁️ visibilitychange: visible
🌍 Изменение видимости: visible
🔊 Страница видима - включаем звук
```

###7. Часто задаваемые вопросы
##❓ Почему не используются сигналы SDK для управления звуком?
Сигналы SDK (game_api_paused/resumed) работают нестабильно в разных версиях Godot. Метод с visibilitychange напрямую отслеживает событие браузера и работает надежно.

##❓ Нужно ли включать плагин в настройках проекта?
Нет, плагин Mist1351/yandex-games-sdk работает через автозагрузку, а не через стандартную систему плагинов Godot.

##❓ Как проверить, что SDK инициализирован?
В debug-панели Яндекс.Игр должен гореть зеленый круг и надпись "Is loader: true".

###📝 Лицензия
Данное руководство распространяется под лицензией MIT. Вы можете свободно использовать, копировать и модифицировать его для своих проектов.

Авторы: [Fyketh]
Дата: Март 2026
Версия Godot: 4.6
