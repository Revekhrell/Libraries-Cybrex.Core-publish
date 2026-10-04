# Cybrex.Core

> Фундаментальный слой фреймворка Cybrex: примитивы
  представления, контракты, алгоритмы и инфраструктурные
  сервисы, на которых строятся все остальные модули.

---

## Обзор

Cybrex.Core – это не сборник утилит и не библиотека для решения
прикладных задач. Это набор строительных блоков, из которых
собираются все остальные ветви фреймворка.

Принципы ядра:

- Ядро не решает конкретных прикладных задач. Оно предоставляет
  базовые инструменты для решения этих задач.
- Ядро не зависит ни от одного другого модуля. Только BCL.

---

## Основные понятия

### Примитивы представления

- **`StreamCounter`** – поток без данных. Держит только
  Position и Length. Предназначен для сценариев, где точный
  размер должен быть вычислен без записи, но может быть вычислен
  только в процессе кодирования. Позволяет "писать" и
  "перемещаться", но не позволяет читать.

- **`BitArray` / `BitSpan` / `ReadOnlyBitSpan`** –
  массовая работа с битами. Интринсики, логические операции,
  zero-copy обёртки. Отдельные биты неэффективны, поэтому API
  ориентирован на пакетные операции. Реализует LSB-first
  индексацию блоков данных, это важно учитывать, поскольку
  иначе, оборачивание, например, int[2] массива, может дать
  неожиданный порядок бит.

### Контракты и проверки

- **`ICloneable<T>`** – типобезопасная альтернатива
  System.ICloneable.

- **`ExceptionHelper`** – статический помощник по условному
  вызову исключений. Обобщённые проверки аргументов, например:
  `ThrowIfArgumentOutOfRange<T>(T argument, T low, T high, string? argumentName) where T : INumber<T>`.

### Алгоритмы обработки последовательностей

- **`SequenceCatcher<T>`** – перехватчик последовательности.
  Инициализируется последовательностью, предоставляет:
  - `bool Catch(T value)` – проверяет, была ли встречена
    искомая последовательность при переборе;
  - `int Match(ReadOnlySpan<T> values)` – ищет
    последовательность в уже загруженном в память буфере.

### Потокобезопасность

- **`ReadWriteLocker`** – обёртка над
  `ReaderWriterLockSlim`. Методы `ReadLock()` и
  `WriteLock()` возвращают IDisposable-структуры.

- **`ReadLocker` / `WriteLocker`** –
  IDisposable-структуры для оборачивания кода в потокобезопасный
  блок в контексте `ReadWriteLocker`.

- **`LockableObject`** – абстрактный класс для
  потокобезопасных объектов. Содержит `Locker` и
  предоставляет `GetField<T>(ref T field)` и
  `SetField<T>(ref T field, T value)`.

### Инфраструктурные сервисы

- **`Localizer`** – статический локализатор для сборок
  `Cybrex`.
  - `Initialize()` сканирует домен на наличие сборок
  - `"Cybrex*"` и подгружает их словари локализации.
  - `RegisterLocales(Assembly assembly)` позволяет загрузить
    словари любой сборки при условии, что они являются
    внедрёнными ресурсами по пути
    `"<имя_сборки>.Resources.Locale.<имя_культуры>.json"`.
  - Для инвариантной локализации, разместите ресурс по пути
    `"<имя_сборки>.Resources.Locale..json"`.
  - Позволяет регистрировать, устанавливать и удалять записи
    локализаций.
  - Потокобезопасен.

---

## Примеры

<details>
<summary>Потокобезопасность</summary>

```csharp
public class CustomConcurrentList<T> : LockableObject, IList<T>
{
    private int _Count;
    public int Count
    {
        get => GetField(ref _Count);
    }

    public void Add(T item)
    {
        using WriteLocker locker = Locker.WriteLock();
        // Добавление
    }

    // Остальной код
}
```
</details>

<details>
<summary>Локализация</summary>

```csharp
internal class Program
{
    static void Main(string[] args)
    {
        // Загрузка словарей фреймворка
        Localizer.Initialize();

        // Загрузка словарей текущей сборки
        Localizer.RegisterLocales(typeof(Program).Assembly);
    }
}
```
</details>

<details>
<summary>Логическое умножение массивов</summary>

```csharp
// Осуществляет логическое умножение двух массивов
private static void And<T>(T[] src, T[] dst) where T : unmanaged
{
    // Проверка аргументов
    ExceptionHelper.ThrowIfArgumentIsNull(src, nameof(src));
    ExceptionHelper.ThrowIfArgumentIsNull(dst, nameof(dst));
    ExceptionHelper.ThrowIfContainersNonEquals(src.Length, dst.Length, nameof(src), nameof(dst));

    // Выбор стратегии
    int bytes = Unsafe.SizeOf<T>() * src.Length;
    if (bytes % sizeof(long) == 0)
    {
        ReadOnlyBitSpan srcBits = ReadOnlyBitSpan.FromAligned(src);
        BitSpan dstBits = BitSpan.FromAligned(dst);
        dstBits.And(srcBits);
    }
    else
    {
        ReadOnlySpan<byte> srcBuffer = MemoryMarshal.Cast<T, byte>(src);
        ReadOnlySpan<ulong> srcBlocks = MemoryMarshal.Cast<T, ulong>(src);
        ReadOnlyBitSpan srcMainBits = ReadOnlyBitSpan.FromAligned(srcBlocks);
        ReadOnlyBitSpan srcTailBits = BitArray.FromBuffer(srcBuffer.Slice(srcBlocks.Length * sizeof(long)));

        Span<byte> dstBuffer = MemoryMarshal.Cast<T, byte>(dst.AsSpan());
        Span<ulong> dstBlocks = MemoryMarshal.Cast<T, ulong>(dst.AsSpan());
        BitSpan dstMainBits = BitSpan.FromAligned(dstBlocks);
        BitSpan dstTailBits = BitArray.FromBuffer(dstBuffer.Slice(dstBlocks.Length * sizeof(long)));

        dstMainBits.And(srcMainBits);
        dstTailBits.And(srcTailBits);
        dstTailBits.CopyTo(dstBuffer.Slice(dstBlocks.Length * sizeof(long)));
    }
}
```
</details>