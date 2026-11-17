# MASTER PROMPT — Glitch Photo (Android, Kotlin)

> Главный контекст для Claude Code. Положи файл в корень репозитория под именем `CLAUDE.md` —
> Claude Code загружает его автоматически в каждой сессии. Детальные задания по эффектам лежат в `prompts/`.

---

## 1. Цель

Нативное Android-приложение на Kotlin для создания glitch-фотографий:
редактор со стеком эффектов, живая камера с glitch-эффектами, экспорт в полном разрешении,
позже — видео и анимация. Нужен опыт в OpenGL ES, обработке изображений, CameraX/Camera2 и Media3.

Каталог содержит **61 фишку** в 11 группах (A–K). Каждая описана в `prompts/<группа>.md`
в формате: промпт → параметры → реализация → приёмка → подводные камни.

## 2. Как со мной работать (обязательно)

1. Перед началом задачи прочитай нужный файл из `prompts/`. Если его нет — скажи об этом и не додумывай задание.
2. **Сначала план, потом код.** Покажи план: какие файлы/классы создашь или изменишь, какие тесты добавишь,
   какие зависимости добавишь (списком: артефакт, версия, лицензия). Не пиши код, пока я не подтвержу план.
   Подтверждение плана = подтверждение перечисленных в нём зависимостей.
3. **Спрашивай отдельно** перед: добавлением зависимости, которой не было в плане; изменением контрактов
   из раздела 5; изменением minSdk/targetSdk; удалением файлов; изменением структуры модулей.
4. Одна итерация = один эффект (или одна задача фундамента): дескриптор + реализация + регистрация
   в `EffectRegistry` + тесты + строка в `docs/EFFECTS_STATUS.md`.
5. После итерации запусти проверки (раздел 10) и честно отчитайся о результате. Если не смог
   что-то запустить (нет устройства для instrumented-тестов) — так и скажи.
6. Не выдумывай API. Если не уверен в сигнатуре Android / CameraX / Media3 / ML Kit —
   проверь по документации (Context7 MCP, если доступен) или спроси меня.
7. Код и комментарии в коде — на английском. Сообщения мне — на русском.
8. Никогда не копируй код из проектов под GPL/неясной лицензией. Внешние проекты — только как визуальный референс.
   Все сторонние библиотеки фиксируй в `THIRD_PARTY.md` с лицензией.

## 3. Технический стек и ограничения

| Что | Решение |
|---|---|
| Язык | Kotlin 2.x, coroutines, kotlinx.serialization |
| UI | Jetpack Compose, Material 3; UDF: `ViewModel` + `StateFlow<UiState>` |
| DI | Hilt (в чистых JVM-модулях — только `javax.inject`) |
| minSdk / target | minSdk 26, targetSdk — последний стабильный на момент фундамента, фиксируется в version catalog |
| Сборка | Gradle version catalog (`gradle/libs.versions.toml`) + convention-плагины в `build-logic/`. Все версии зафиксированы; обновление — с моего согласия |
| Графика | **OpenGL ES 3.0 — базовый уровень** (GLSL ES 3.00). GLES 3.1 (compute) — опционально, по capability |
| Запрещено | RenderScript (deprecated). AGSL — только для мелкого UI, не для движка |
| Камера | CameraX (+ Camera2 interop для ручных параметров) |
| Видео | Media3 Transformer / Effect, MediaCodec |
| ML | ML Kit (Subject Segmentation, Selfie Segmentation, Face Detection); на устройствах без Google Play services — capability недоступна |
| Кодеки для glitch | **Свой JPEG/PNG-кодек на чистом Kotlin** (`engine/codec`), см. 6.3. libjpeg-turbo через NDK — с этапа 5 (B-13 и пр.) |
| Native | NDK + CMake, 16 KB page size alignment обязательно |
| Хранение | DataStore (настройки), Room (пресеты/история) |
| EXIF | `androidx.exifinterface` |

## 4. Структура модулей

```
app/                    — Application, навигация, DI-граф
build-logic/            — convention-плагины Gradle
core/model/             — Recipe, EffectInstance, ParamDef, EffectDescriptor, RecipeMigrator (чистый Kotlin, без android.*)
core/common/            — SplitMix64 RNG, математика, цветовые пространства, детерминированный ресемплинг (чистый Kotlin)
engine/cpu/             — PixelBuffer + CPU-эффекты (чистый Kotlin/JVM → быстрые JUnit-тесты)
engine/codec/           — GlitchCodec: свой baseline-JPEG и PNG (чистый Kotlin), PNG/GIF writer;
                          PlatformCodec: Android Bitmap/BitmapFactory (импорт, экспорт, WebP)
engine/gl/              — EGL, ShaderProgram, FBO-пул, GL-эффекты, шейдеры в src/main/assets/shaders/
engine/native/          — JNI + CMake (libjpeg-turbo, FFT и т.п.), подключается с этапа 5
engine/pipeline/        — PipelineExecutor, Compositor (маски + blend), кэш шагов, тайлинг
feature/editor/         — экран редактора (undo/redo)
feature/camera/         — живая камера + временные эффекты
feature/export/         — экспорт, MediaStore, share
feature/video/          — видео-эффекты и кодеки (этап 5)
feature/ml/             — маски из ML Kit (этап 5)
docs/EFFECTS_STATUS.md  — таблица статусов эффектов
```

## 5. Ключевые контракты (менять только с моего согласия)

```kotlin
// core/model
@Serializable
data class Recipe(
    val version: Int = 1,                  // формат рецепта; миграции — RecipeMigrator
    val globalSeed: Long,
    val steps: List<EffectInstance>,
    val masks: Map<String, MaskSpec> = emptyMap(),   // ключ = maskId
)

@Serializable
data class EffectInstance(
    val uid: String,                       // UUID шага
    val effectId: String,                  // "rgb_split"
    val params: Map<String, ParamValue>,
    val seed: Long,
    val enabled: Boolean = true,
    val opacity: Float = 1f,
    val blend: BlendMode = BlendMode.Normal,
    val maskId: String? = null,            // ссылка на Recipe.masks
)

@Serializable sealed interface ParamValue {
    @Serializable @SerialName("f") data class F(val v: Float) : ParamValue
    @Serializable @SerialName("i") data class I(val v: Int) : ParamValue
    @Serializable @SerialName("b") data class B(val v: Boolean) : ParamValue
    @Serializable @SerialName("e") data class E(val v: String) : ParamValue
    @Serializable @SerialName("c") data class C(val argb: Int) : ParamValue
}

// Минимальный набор; расширение — в prompts/J_targeting.md (с моего согласия).
@Serializable enum class BlendMode { Normal, Add, Multiply, Screen, Overlay, Difference, Lighten, Darken }

// Варианты масок определяются в prompts/J_targeting.md. Здесь — только корень иерархии.
@Serializable sealed interface MaskSpec { val invert: Boolean; val feather: Float /* доля ширины */ }

sealed interface ParamDef {
    val key: String; val label: String; val group: String?
}
data class FloatParam(
    override val key: String, override val label: String,
    val min: Float, val max: Float, val default: Float,
    val unit: ParamUnit = ParamUnit.None,
    val randomRange: ClosedFloatingPointRange<Float>? = null,   // «разумный» диапазон для рандома
    override val group: String? = null,
) : ParamDef
// IntParam, BoolParam, EnumParam(options), ColorParam — аналогично

// PixelsAbsolute — величина в пикселях, которая по природе не масштабируется с разрешением (см. 6.2)
enum class ParamUnit { None, FractionOfWidth, FractionOfHeight, Degrees, Percent, Count, PixelsAbsolute }

enum class Engine { GL, CPU, CODEC, NATIVE, CAMERA, VIDEO }
sealed interface Locality { data class Local(val radiusFrac: Float) : Locality; data object Global : Locality }
sealed interface WorkResPolicy { data object Native : WorkResPolicy; data class Fixed(val longSide: Int) : WorkResPolicy }
enum class Capability { GLES31, FLOAT_RT, CAMERA_RAW, MANUAL_SENSOR, TEST_PATTERN, TONEMAP_CURVE, ML_SUBJECT, ML_FACE, NATIVE_JPEG }

// CrossDevice — результат побайтно одинаков на JVM и на любом устройстве (чистый Kotlin, без float-зависимостей от платформы).
// SameDevice — побайтно одинаков только на одном устройстве (GL, платформенные кодеки).
enum class Determinism { CrossDevice, SameDevice }

data class EffectDescriptor(
    val id: String,
    val group: EffectGroup,                // A..K
    val title: String,
    val params: List<ParamDef>,
    val engine: Engine,
    val locality: Locality,
    val determinism: Determinism = Determinism.SameDevice,
    val temporal: Boolean = false,         // нужен буфер кадров (камера/видео)
    val requires: Set<Capability> = emptySet(),
    val workRes: WorkResPolicy = WorkResPolicy.Native,
)
```

```kotlin
// engine/cpu — чистый Kotlin
// Непрозрачный sRGB ARGB_8888, НЕ premultiplied, alpha всегда 0xFF внутри пайплайна.
class PixelBuffer(val width: Int, val height: Int, val argb: IntArray)

// Через CpuEffect реализуются эффекты с Engine.CPU, Engine.CODEC и Engine.NATIVE.
// Engine.CAMERA — контракт в prompts/G_time_camera.md и H_hardware.md; Engine.VIDEO — в prompts/I_video.md.
interface CpuEffect {
    val descriptor: EffectDescriptor
    suspend fun apply(input: PixelBuffer, p: ResolvedParams, ctx: CpuContext): PixelBuffer
}

// engine/gl
interface GlEffect {
    val descriptor: EffectDescriptor
    fun onCreate(gl: GlContext)
    fun render(gl: GlContext, input: GlTexture, output: GlTarget, p: ResolvedParams, frame: FrameInfo)
    fun onRelease(gl: GlContext)
}

// engine/codec — детерминированный кодек для glitch-эффектов (Determinism.CrossDevice).
// Одна и та же реализация работает в JUnit на JVM и на устройстве.
interface GlitchCodec {
    fun encodeJpeg(src: PixelBuffer, quality: Int, opts: JpegOptions = JpegOptions()): ByteArray
    fun encodePng(src: PixelBuffer, opts: PngOptions = PngOptions()): ByteArray
    fun decode(bytes: ByteArray): DecodeResult
}
sealed interface DecodeResult {
    // partial = данные кончились/сломались посреди скана; недекодированная часть заполнена
    // детерминированно (правило заполнения — в JpegOptions/PngOptions)
    data class Ok(val image: PixelBuffer, val partial: Boolean) : DecodeResult
    data object Failed : DecodeResult           // сломан заголовок/таблицы — декодировать нечего
}

// engine/codec — платформенный кодек (Android Bitmap). Импорт, финальный экспорт, WebP-эффекты.
// Эффекты на нём — Determinism.SameDevice. JVM-реализация для тестов (javax.imageio) — только JPEG/PNG.
interface PlatformCodec {
    fun encodeJpeg(src: PixelBuffer, quality: Int): ByteArray
    fun encodeWebp(src: PixelBuffer, quality: Int, lossless: Boolean): ByteArray
    fun decode(bytes: ByteArray): PixelBuffer?   // null = декодер не справился
}

// Значения уже провалидированы и дополнены default'ами из ParamDef.
interface ResolvedParams {
    val values: Map<String, ParamValue>
    val seed: Long                         // уже смешан с globalSeed (6.1)
    val outWidth: Int
    val outHeight: Int
    fun f(k: String): Float
    fun i(k: String): Int
    fun b(k: String): Boolean
    fun e(k: String): String
    fun c(k: String): Int                  // ARGB
}

data class FrameInfo(val index: Long, val timeSec: Float, val history: FrameHistory? = null)
```

## 6. Сквозные правила движка

**6.1 Детерминизм.** Любая случайность берётся только из seed.
Seed шага = `splitMix64(globalSeed xor splitMix64(effectInstance.seed))` (реализация SplitMix64 — в `core/common`,
покрыта тестом на эталонные значения). В шейдер seed передаётся как `uniform uint uSeed`
(`GLES30.glUniform1ui`) после свёртки 64 → 32 бит:

```kotlin
fun Long.toShaderSeed(): Int = (this xor (this ushr 32)).toInt()
```

Гарантии:
- Один и тот же рецепт + то же фото на **том же устройстве** = побайтно одинаковый результат.
- Эффекты `Determinism.CrossDevice` дают побайтно одинаковый результат на JVM и на любом устройстве.
- Превью и экспорт совпадают «по смыслу» (6.2); для шагов с `WorkResPolicy.Fixed` — побайтно на workRes (6.3).

**6.2 Независимость от разрешения.** Все пространственные параметры задаются в долях ширины/высоты
(`ParamUnit.FractionOfWidth`), а не в пикселях. Превью рендерится на канонический прокси (длинная сторона 1080, см. 6.3),
экспорт — в полном разрешении; картинка должна совпадать по смыслу.
- Случайные ячейки (блоки, полосы, строки сдвига) вычисляются **от `uv` и количества ячеек**, а не от
  `gl_FragCoord`/индекса пикселя: `ivec2 c = ivec2(floor(uv * cellCount))`. Иначе на 1080 и 4000 px
  шум будет разным.
- Исключение — эффекты, где величина в пикселях по природе не зависит от разрешения
  (напр., ошибка stride, см. C-19), — для них `ParamUnit.PixelsAbsolute`.

**6.3 Кодек-эффекты (группы B и C).**
- JPEG и PNG для glitch-эффектов — **только через `GlitchCodec`** (свой baseline-кодек на чистом Kotlin).
  Платформенные `Bitmap.compress`/`BitmapFactory` дают разные байты на разных устройствах и по-разному
  ведут себя на битых данных (часто возвращают частичное изображение вместо `null`), поэтому для
  databending не годятся. Декодер `GlitchCodec` обязан быть толерантным к порче: декодировать всё, что
  можно, и детерминированно заполнять остальное.
- WebP-эффекты используют `PlatformCodec` и помечаются `Determinism.SameDevice`.
- Эффекты работают в фиксированном рабочем разрешении (`WorkResPolicy.Fixed(1024)` по умолчанию;
  параметр шага `workRes`: 1024 / 2048).
- **Вход шага на workRes считается одинаково на превью и экспорте:** префикс рецепта (шаги 0..k−1)
  выполняется на каноническом прокси (длинная сторона 1080, получен из оригинала детерминированным
  CPU-даунскейлом из `core/common`), затем тем же CPU-даунскейлом — до workRes. Результат шага на workRes
  кэшируется и на экспорте берётся из кэша (или пересчитывается тем же путём). Так превью и экспорт
  совпадают побайтно на workRes.
- **Апскейл к целевому размеру** — параметр шага `upscale`:
  - `DetailPreserve` (по умолчанию): `out = fullInput + nearestUp(fxWork − inputWork)` с клампом —
    блоки 8×8 сохраняют видимый размер, а там, где эффект ничего не изменил, остаётся полная детализация;
  - `Nearest`: `out = nearestUp(fxWork)` — честный «пиксельный» вид, вся картинка становится workRes.
  Комбинирование на экспорте делается в GL (6.7), а не в куче.

**6.4 Маски и смешивание — забота композитора, не эффекта.** Эффект рендерит «полностью
обработанное» изображение; `Compositor` делает `out = mix(base, blend(base, fx), opacity * mask)`.

**6.5 Пайплайн.** Подряд идущие GL-шаги выполняются без readback (ping-pong FBO).
Переход GL→CPU — `glReadPixels` (по возможности через PBO), CPU→GL — upload текстуры.
Кэш шагов — только для превью (прокси-разрешение) и результатов на workRes (6.3); ключ
`hash(inputHash, step)`; при изменении шага k пересчитываются только шаги k..n. Кэш — LRU в RAM
с лимитом `min(64 МБ, maxMemory / 8)`. На экспорте полноразмерные промежуточные результаты не кэшируются.

**6.6 Потоки.** Один GL-поток (HandlerThread) владеет EGL-контекстом редактора. Камера — отдельный
EGL-контекст с shared-ресурсами. CPU-эффекты — `Dispatchers.Default`, обработка строк чанками
с `ensureActive()` (отмена при движении слайдера).

**6.7 Память и большие изображения.**
- Максимальное разрешение экспорта — настройка, по умолчанию длинная сторона ≤ 4096 (≈12 Мп);
  максимум — `min(GL_MAX_TEXTURE_SIZE, 8192)`. Источник больше лимита даунскейлится на импорте.
- В Java-куче одновременно не больше **двух** полноразмерных ARGB-копий (12 Мп = 48 МБ каждая).
  Композитинг (`base`, `fx`, `out`) и апскейл кодек-шагов на экспорте — в GL-текстурах с переиспользованием
  FBO из пула, не в куче.
- Если выход превышает `GL_MAX_TEXTURE_SIZE` или лимиты памяти — Local-эффекты рендерятся тайлами
  с перекрытием `radiusFrac`. Global-эффекты (pixel sort, feedback, FFT) — целиком, при превышении лимитов —
  документированный кап (4096 по длинной стороне) и уведомление пользователю.

**6.8 Capabilities.** При старте собрать `DeviceCaps`: версия GLES, расширения
(`EXT_color_buffer_half_float`, `EXT_color_buffer_float`), `GL_MAX_TEXTURE_SIZE`,
`GL_MAX_ARRAY_TEXTURE_LAYERS`, возможности камеры (RAW, MANUAL_SENSOR, test patterns, tonemap curve),
список кодеков, доступность ML-модулей и Google Play services. Эффект с невыполненными `requires`
в UI показывается как «недоступно на этом устройстве» с причиной.

**6.9 Ошибки кодеков.**
- `GlitchCodec.decode` → `Ok(partial = true)` — это валидный glitch-результат, не ошибка.
- `Failed` (или `null` от `PlatformCodec`) — повторить порчу с производным seed
  `splitMix64(stepSeed + attempt)`, до 8 попыток. Номер попытки в рецепт **не сохраняется**: повторы
  детерминированы, поэтому пересчёт даёт тот же результат. Номер попытки пишется в метаданные кэша
  (для отладки).
- Если все 8 попыток неудачны — вернуть вход без изменений и показать мягкое сообщение.

**6.10 Цвет, импорт, экспорт.**
- Пайплайн работает в **sRGB, гамма-кодированные 8-битные значения**. Линейное пространство — только
  внутри эффекта, если это явно указано в его задании.
- Импорт: Photo Picker (`PickVisualMedia`) → декодирование → применение EXIF-ориентации к пикселям →
  конвертация в sRGB (`inPreferredColorSpace`, в т.ч. из Display P3) → `PixelBuffer`.
  Ultra HDR: берётся SDR-база, gainmap отбрасывается (MVP).
- Экспорт: sRGB, пиксели уже повёрнуты (EXIF Orientation = normal). GPS-теги удаляются по умолчанию,
  `DateTimeOriginal` сохраняется. Сохранение через MediaStore; на API 26–28 —
  `WRITE_EXTERNAL_STORAGE` с `maxSdkVersion="28"`, на API 29+ разрешение не нужно.
- Разрешения: `CAMERA` — runtime-запрос только при входе в камеру. Доступ к галерее — без разрешений, через Photo Picker.

**6.11 Рецепты и история.**
- `Recipe` неизменяем; undo/redo — стек снимков `Recipe` (лимит 100).
- `RecipeMigrator` переводит рецепт любой прошлой `version` в текущую; на каждую версию — тест с JSON-фикстурой.
- Неизвестный `effectId` при загрузке (рецепт из более новой версии) не удаляется: шаг показывается как
  «неизвестный эффект», пропускается при рендере и сохраняется обратно без изменений.

## 7. Конвенции шейдеров

Все GL-эффекты пишут только функцию `vec4 effect(vec2 uv)`; движок склеивает прелюдию + тело.

```glsl
#version 300 es
precision highp float;
precision highp int;

uniform sampler2D uTex;       // вход шага; размер входа — textureSize(uTex, 0)
uniform vec2  uResolution;    // размер выхода в px
uniform uint  uSeed;          // свёрнутый seed шага (6.1)
uniform float uTime;          // сек, для анимации/камеры; 0 для стоп-кадра
in vec2 vUv;
out vec4 fragColor;

float luma(vec3 c) { return dot(c, vec3(0.2126, 0.7152, 0.0722)); }

uint pcg(uint v) {
    uint s = v * 747796405u + 2891336453u;
    uint w = ((s >> ((s >> 28u) + 4u)) ^ s) * 277803737u;
    return (w >> 22u) ^ w;
}
// uint → [0,1). Берём старшие 24 бита: они точно представимы во float32,
// поэтому результат строго < 1.0 (float(h) / 2^32 мог округлиться до 1.0).
float u01(uint h)         { return float(h >> 8u) * (1.0 / 16777216.0); }
// детерминированный хэш целочисленной ячейки → [0,1)
float hash1(int x)        { return u01(pcg(uint(x) ^ uSeed)); }
float hash2(ivec2 p)      { return u01(pcg(pcg(uint(p.x) ^ uSeed) ^ uint(p.y))); }
float hash3(ivec3 p)      { return u01(pcg(pcg(pcg(uint(p.x) ^ uSeed) ^ uint(p.y)) ^ uint(p.z))); }

vec4 tex(vec2 uv) { return texture(uTex, uv); }
vec2 px() { return 1.0 / uResolution; }
// Количество ячеек по осям для размера ячейки, заданного в доле ширины (квадратные ячейки, 6.2)
vec2 cellsFromWidthFrac(float f) { return vec2(1.0 / f, uResolution.y / (uResolution.x * f)); }

vec4 effect(vec2 uv);
void main() { fragColor = effect(vUv); }
```

- Шейдеры лежат в `engine/gl/src/main/assets/shaders/<effectId>.frag` (+ `<effectId>_pass2.frag`).
- Uniform-имена: `u` + PascalCase ключа параметра (`amount` → `uAmount`).
- Дополнительные текстуры эффект объявляет в теле шейдера по соглашению:
  `uPass1` — выход предыдущего прохода, `uHistory0..uHistoryN` — прошлые кадры (0 = предыдущий),
  `uLut` — таблица. Привязку делает `GlEffect.render`.
- Текстуры: `GL_CLAMP_TO_EDGE` по умолчанию; режим края — параметр, где важно.
- Float render targets (RGBA16F) — только при `Capability.FLOAT_RT`, иначе fallback на RGBA8
  с явной упаковкой/масштабированием.

## 8. Производительность (бюджеты)

| Сценарий | Бюджет |
|---|---|
| Превью: цепочка из 5 GL-эффектов, 1080 px | ≤ 16 мс на средне-верхнем устройстве, ≤ 33 мс на среднем |
| CPU/кодек-эффект в workRes 1024 | ≤ 300 мс |
| Экспорт 12 Мп, типичный рецепт | ≤ 5 с, с прогрессом и отменой |
| Камера | 30 fps на превью 1080p для GL-цепочки до 3 эффектов |

## 9. Эталонная среда тестов

- JVM-тесты: JUnit на `engine/cpu`, `engine/codec`, `core/*`.
- Instrumented GL-тесты: эталоны (golden) снимаются на **эмуляторе API 34 x86_64 с `-gpu swiftshader_indirect`**
  и хранятся в `engine/gl/src/androidTest/assets/golden/`. На физических устройствах сравнение с теми же
  эталонами по SSIM ≥ 0.98. Перегенерация эталонов — только с моего согласия, с объяснением причины.

## 10. Проверки и Definition of Done

- `./gradlew ktlintCheck detekt` — без ошибок.
- `./gradlew :engine:cpu:test :engine:codec:test :core:model:test :core:common:test` — зелёные.
- `./gradlew assembleDebug` — собирается.
- GL-эффекты: instrumented-тест на offscreen EGL pbuffer — шейдер компилируется, golden-сравнение по SSIM ≥ 0.98.
- Тест независимости от разрешения для Local-эффектов: рендер 1080 vs рендер 3000 → даунскейл до 1080 → SSIM ≥ 0.9.
- Тест детерминизма: два рендера одного рецепта на одном устройстве побайтно равны.
- Эффекты `Determinism.CrossDevice`: хэш результата в JVM-тесте совпадает с хэшем в instrumented-тесте.
- Кодек-эффекты: результат на workRes при превью и при экспорте побайтно равен (6.3).
- Эффект зарегистрирован в `EffectRegistry`, параметры отображаются в редакторе автоматически.
- Строка в `docs/EFFECTS_STATUS.md` обновлена.

## 11. Этапы (порядок реализации)

| Этап | Содержимое |
|---|---|
| 0 | Фундамент — `prompts/00_foundation.md`. Обязательно включает: `GlitchCodec` (baseline JPEG + PNG, толерантный декодер), детерминированный CPU-ресемплинг, импорт с EXIF/цветом (6.10), `RecipeMigrator` |
| 1 — MVP | A-01, A-03, A-07, A-08, B-10, B-11, C-19, D-26, D-29, E-34, F-37, F-38, J-56, J-57, K-58, K-61 |
| 2 — полный редактор | A-02, A-04, A-05, A-09, B-12, B-14, B-15, B-16, B-17, B-18, C-20…C-24, D-27, D-28, D-30…D-33, E-35, E-36, F-39, F-40 |
| 3 — камера | G-41…G-46 + живая камера для GL-эффектов |
| 4 — «железо» | H-47…H-50 |
| 5 — native, видео, ML | A-06, B-13, C-25, I-51…I-53, J-54, J-55 |
| 6 — вывод | K-59, K-60 |

## 12. Индекс файлов с заданиями

| Файл | Группа | Эффекты |
|---|---|---|
| `prompts/00_foundation.md` | Фундамент | F0.1–F0.10 |
| `prompts/A_signal.md` | A. Сигнальные/аналоговые сбои | A-01…A-09 |
| `prompts/B_compression.md` | B. Компрессия и форматы | B-10…B-18 |
| `prompts/C_databending.md` | C. Databending сырых данных | C-19…C-25 |
| `prompts/D_pixels.md` | D. Перестановка пикселей | D-26…D-33 |
| `prompts/E_geometry.md` | E. Геометрия и смещение | E-34…E-36 |
| `prompts/F_color.md` | F. Цвет и квантование | F-37…F-40 |
| `prompts/G_time_camera.md` | G. Время и движение | G-41…G-46 |
| `prompts/H_hardware.md` | H. Камера и «железо» | H-47…H-50 |
| `prompts/I_video.md` | I. Видеокодеки | I-51…I-53 |
| `prompts/J_targeting.md` | J. Маски и смешивание | J-54…J-57 |
| `prompts/K_output.md` | K. Вывод и метафункции | K-58…K-61 |

## 13. Шаблон запроса к Claude Code

```
Прочитай prompts/<файл>.md.
Задача: реализовать <ID> (<effectId>).
Покажи план (файлы, классы, тесты, новые зависимости) и жди подтверждения.
```
