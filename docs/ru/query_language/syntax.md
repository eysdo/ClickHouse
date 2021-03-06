# Синтаксис

В системе есть два вида парсеров: полноценный парсер SQL (recursive descent parser) и парсер форматов данных (быстрый потоковый парсер).
Во всех случаях кроме запроса INSERT, используется только полноценный парсер SQL.
В запросе INSERT используется оба парсера:

``` sql
INSERT INTO t VALUES (1, 'Hello, world'), (2, 'abc'), (3, 'def')
```

Фрагмент `INSERT INTO t VALUES` парсится полноценным парсером, а данные `(1, 'Hello, world'), (2, 'abc'), (3, 'def')` - быстрым потоковым парсером.
Данные могут иметь любой формат. При получении запроса, сервер заранее считывает в оперативку не более `max_query_size` байт запроса (по умолчанию, 1МБ), а всё остальное обрабатывается потоково.
Таким образом, в системе нет проблем с большими INSERT запросами, как в MySQL.

При использовании формата Values в INSERT запросе может сложиться иллюзия, что данные парсятся также, как выражения в запросе SELECT, но это не так - формат Values гораздо более ограничен.

Далее пойдёт речь о полноценном парсере. О парсерах форматов, смотри раздел "Форматы".

## Пробелы

Между синтаксическими конструкциями (в том числе, в начале и конце запроса) может быть расположено произвольное количество пробельных символов. К пробельным символам относятся пробел, таб, перевод строки, CR, form feed.

## Комментарии

Поддерживаются комментарии в SQL-стиле и C-стиле.
Комментарии в SQL-стиле: от `--` до конца строки. Пробел после `--` может не ставиться.
Комментарии в C-стиле: от `/*` до `*/`. Такие комментарии могут быть многострочными. Пробелы тоже не обязательны.

## Ключевые слова

Ключевые слова (например, `SELECT`) регистронезависимы. Всё остальное (имена столбцов, функций и т. п.), в отличие от стандарта SQL, регистрозависимо. Ключевые слова не зарезервированы (а всего лишь парсятся как ключевые слова в соответствующем контексте).

## Идентификаторы

Идентификаторы (имена столбцов, функций, типов данных) могут быть квотированными или не квотированными.
Не квотированные идентификаторы начинаются на букву латинского алфавита или подчёркивание; продолжаются на букву латинского алфавита или подчёркивание или цифру. Короче говоря, должны соответствовать регулярному выражению `^[a-zA-Z_][0-9a-zA-Z_]*$`. Примеры: `x, _1, X_y__Z123_.`

Квотированные идентификаторы расположены в обратных кавычках ` `id` ` (также, как в MySQL), и могут обозначать произвольный (непустой) набор байт. При этом, внутри записи такого идентификатора, символы (например, символ обратной кавычки) могут экранироваться с помощью обратного слеша. Правила экранирования такие же, как в строковых литералах (см. ниже).
Рекомендуется использовать идентификаторы, которые не нужно квотировать.

## Литералы

### Числовые

Числовой литерал пытается распарситься:
- сначала как 64-битное число без знака, с помощью функции `strtoull`;
- если не получилось - то как 64-битное число со знаком, с помощью функции `strtoll`;
- если не получилось - то как число с плавающей запятой, с помощью функции `strtod`;
- иначе - ошибка.

Соответствующее значение будет иметь тип минимального размера, который вмещает значение.
Например, 1 парсится как `UInt8`, а 256 - как `UInt16`. Подробнее смотрите раздел [Типы данных](../data_types/index.md#data_types).

Примеры: `1`, `18446744073709551615`, `0xDEADBEEF`, `01`, `0.1`, `1e100`, `-1e-100`, `inf`, `nan`.

### Строковые

Поддерживаются только строковые литералы в одинарных кавычках. Символы внутри могут быть экранированы с помощью обратного слеша. Следующие escape-последовательности имеют соответствующее специальное значение: `\b`, `\f`, `\r`, `\n`, `\t`, `\0`, `\a`, `\v`, `\xHH`. Во всех остальных случаях, последовательности вида `\c`, где c - любой символ, преобразуется в c. Таким образом, могут быть использованы последовательности `\'` и `\\`. Значение будет иметь тип [String](../data_types/string.md#data_types-string).

Минимальный набор символов, которых вам необходимо экранировать в строковых литералах: `'` и `\`.

### Составные

Поддерживаются конструкции для массивов: `[1, 2, 3]` и кортежей: `(1, 'Hello, world!', 2)`.
На самом деле, это вовсе не литералы, а выражение с оператором создания массива и оператором создания кортежа, соответственно.
Подробнее смотри в разделе "Операторы".
Массив должен состоять хотя бы из одного элемента, а кортеж - хотя бы из двух.
Кортежи носят служебное значение для использования в секции IN запроса SELECT. Кортежи могут быть получены в качестве результата запроса, но не могут быть сохранены в базу (за исключением таблиц типа Memory).

<a name="null-literal"></a>

### NULL

Обозначает, что значение отсутствует.

Чтобы в поле таблицы можно было хранить `NULL`, оно должно быть типа [Nullable](../data_types/nullable.md#data_type-nullable).

В зависимости от формата данных (входных или выходных) `NULL` может иметь различное представление. Подробнее смотрите в документации для [форматов данных](../interfaces/formats.md#formats).

При обработке `NULL` есть множество особенностей. Например, если хотя бы один из аргументов операции сравнения — `NULL`, то результатом такой операции тоже будет `NULL`. Этим же свойством обладают операции умножения, сложения и пр. Подробнее читайте в документации на каждую операцию.

В запросах можно проверить `NULL` с помощью операторов [IS NULL](operators.md#operator-is-null) и [IS NOT NULL](operators.md#operator-is-not-null), а также соответствующих функций `isNull` и `isNotNull`.

## Функции

Функции записываются как идентификатор со списком аргументов (возможно, пустым) в скобках. В отличие от стандартного SQL, даже в случае пустого списка аргументов, скобки обязательны. Пример: `now()`.
Бывают обычные и агрегатные функции (смотрите раздел "Агрегатные функции"). Некоторые агрегатные функции могут содержать два списка аргументов в круглых скобках. Пример: `quantile(0.9)(x)`. Такие агрегатные функции называются "параметрическими", а первый список аргументов называется "параметрами". Синтаксис агрегатных функций без параметров ничем не отличается от обычных функций.

## Операторы

Операторы преобразуются в соответствующие им функции во время парсинга запроса, с учётом их приоритета и ассоциативности.
Например, выражение `1 + 2 * 3 + 4` преобразуется в `plus(plus(1, multiply(2, 3)), 4)`.
Подробнее смотрите раздел "Операторы" ниже.

## Типы данных и движки таблиц

Типы данных и движки таблиц в запросе `CREATE` записываются также, как идентификаторы или также как функции. То есть, могут содержать или не содержать список аргументов в круглых скобках. Подробнее смотрите разделы "Типы данных", "Движки таблиц", "CREATE".

## Синонимы

В запросе SELECT, в выражениях могут быть указаны синонимы с помощью ключевого слова AS. Слева от AS стоит любое выражение. Справа от AS стоит идентификатор - имя для синонима. В отличие от стандартного SQL, синонимы могут объявляться не только на верхнем уровне выражений:

``` sql
SELECT (1 AS n) + 2, n
```

В отличие от стандартного SQL, синонимы могут использоваться во всех секциях запроса, а не только `SELECT`.

## Звёздочка

В запросе `SELECT`, вместо выражения может стоять звёздочка. Подробнее смотрите раздел "SELECT".

## Выражения

Выражение представляет собой функцию, идентификатор, литерал, применение оператора, выражение в скобках, подзапрос, звёздочку; и может содержать синоним.
Список выражений - одно выражение или несколько выражений через запятую.
Функции и операторы, в свою очередь, в качестве аргументов, могут иметь произвольные выражения.

[Оригинальная статья](https://clickhouse.yandex/docs/ru/query_language/syntax/) <!--hide-->
