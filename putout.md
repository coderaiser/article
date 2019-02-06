# Практическое применение трансформации AST-деревьев на примере Putout

## Содержание

- Введение
- Что такое AST?
- Как строится AST?
- Примеры парсеров
- Кто использует AST?
    - `babel`
    - `eslint`
    - `jscodeshift`
    - `prettier`
    - `putout`
- Трансформация AST на примере плагинов `putout`

## Введение

Каждый день при работе над кодом, на пути к реализации полезного для пользователя функционала становятся вынужденные (неизбежные, либо же просто желательные) изменения кода. Это может быть рефакторинг, обновление библиотеки или фреймворка до новой мажорной версии, обновление синтаксиса JavaScript (что в последнее время совсем не редкость). Даже если библиотека является частью рабочего проекта - изменения неизбежны. Большинство таких изменений - это рутина. В них нет ничего интересного для разработчика с одной стороны, с другой это не приносит ничего бизнесу, а с третьей в процессе обновления нужно быть очень внимательным что бы не наломать дров и не поломать функционал. Таким образом мы приходим к тому, что такую рутину лучше переложить на плечи программ, что бы они все делали сами, а человек, в свою очередь, контролировал все ли правильно сделано. Вот об этом и пойдет речь в сегодняшней статье.

# Как строится AST

## ПАРСЕР

Рассмотрим работу парсера на примере кода:

```js
a + b
```

Обычно парсеры делятся на две части:

1. Лексический анализ

Разбивает код на токены, каждый из которых описывает часть кода:

```json
[{
    "type": "Identifier",
    "value": "a"
}, {
    "type": "Punctuator",
    "value": "+",
}, {
    "type": "Identifier",
    "value": "b"
}]
```

2. Синтаксический анализ.

Строит из токенов синтаксическое дерево:

```json
{
    "type": "BinaryExpression",
    "left": {
        "type": "Identifier",
        "name": "a"
    },
    "operator": "+",
    "right": {
        "type": "Identifier",
        "name": "b"
    }
}
```

## Putout

Putout - это трансформер кода с поддержкой плагинов. По сути это нечто среднее между `eslint` и `babel`,
объединяющее в себе достоинства обоих инструментов.

Как `eslint` `putout` показывает проблемные места в коде, но в отличие от `eslint` `putout` меняет поведение кода, то есть способен исправлять все ошибки которые сможет найти.

Как и `babel` `putout` преобразовывает код, но старается его минимально менять, таким образом его можно применять для работы с кодом, который хранится в репозитории.

### Как Putout устроен изнутри

Работу `putout` можно поделить на две части: движок и плагины. Такая архитектура позволяет при работе с движком не отвлекаться на трансформации, а при работе над плагинами максимально сосредоточится над их предназначением.

#### Встроенные плагины

Работа `putout` строится на системе плагинов. Каждый плагин представляет собой одно правило. С помощью встроенных правил можно сделать следующее:
1. Найти и удалить:
 - не используемые переменные
 - `debugger`
 - вызов `test.only`
 - вызов `test.skip`
 - вызов `console.log`
 - вызов `process.exit`
 - пустые блоки
 - пустые паттерны

 2. Найти и разбить объявление переменных
 
 ```js
 // было
 var one, two;

 // станет
 var one;
 var two;

 ```
 
 3. Конвертировать `esm` в `commonjs`:
 
 ```js
 // было
import one from 'one';

// станет
const one = require('one');
 ```

4. Применить деструктуризацию

```js
// было
const name = user.name;

// станет
const {name} = user;
```

5. Объединить свойства деструктуризации

```js
// было
const {name} = user;
const {password} = user;

// станет
const {
    name,
    password
} = user;
```

#### Пример использования

После того как мы ознакомились со встроенными правилами, мы можем рассмотреть пример использования `putout`.
Создадим файл `example.js` со следующим содержимым:

```js
const x = 1, y = 2;

const name = user.name;
const password = user.password;

console.log(name, password);
```

Теперь запустим `putout`, передав в качестве аргумента `example.js`:

```sh
coderaiser@cloudcmd:~/example$ putout example.js

/home/coderaiser/example/example.js
 1:6   error   "x" is defined but never used            remove-unused-variables
 1:13  error   "y" is defined but never used            remove-unused-variables
 6:0   error   Unexpected "console" call                remove-console
 1:0   error   variables should be declared separately  split-variable-declarations
 3:6   error   Object destructuring should be used      apply-destructuring
 4:6   error   Object destructuring should be used      apply-destructuring

✖ 6 errors in 1 files
  fixable with the `--fix` option
```

Мы получим информацию содержащую 6 ошибок, рассмотренных более детально выше, теперь исправим их, и посмотрим, что получилось:

```sh
coderaiser@cloudcmd:~/example$ putout example.js --fix
coderaiser@cloudcmd:~/example$ cat example.js
const {
  name,
  password
} = user;
```

В результате исправления неиспользуемые переменные и вызовы `console.log` были удалены, так же была применена деструктуризация.

#### Настройки

Настройки по умолчанию не всегда и не всем могут подойти, поэтому `putout` поддерживает конфигурационный файл `.putout.json`, он состоит из следующих разделов:

- Rules
- Ignore
- Match
- Plugins

##### Rules

Секция `rules` содержит систему правил. Правила, по умолчанию, выставлены следующим образом:

```json
{
    "rules": {
        "remove-unused-variables": true,
        "remove-debugger": true,
        "remove-only": true,
        "remove-skip": true,
        "remove-process-exit": false,
        "remove-console": true,
        "split-variable-declarations": true,
        "remove-empty": true,
        "remove-empty-pattern": true,
        "convert-esm-to-commonjs": false,
        "apply-destructuring": true,
        "merge-destructuring-properties": true
    }
}
```

Для того что бы включить `remove-process-exit` достаточно выставить его в `true` в файле `.putout.json`:

```json
{
    "rules": {
        "remove-process-exit": true
    }
}
```

Этого будет достаточно для того, что бы сообщать обо всех вызовах `process.exit` найденные в коде, и удалять их в случае использования параметра `--fix`.

##### Ignore

Если какие-то папки необходимо добавить в список исключений, достаточно добавить секцию `ignore`:

```json
{
    "ignore": [
        "test/fixture"
    ]
}
```

##### Match

В случае необходимости разветвленной системы правил, например, включить `process.exit` для каталога `bin`, достаточно воспользоваться секцией `match`:

```json
{
    "match": {
        "bin": {
            "remove-process-exit": true,
        }
    }
}
```

##### Plugins

В случае использования плагинов, которые не встроены и имеют префикс `putout-plugin-`, их необходимо включить в секцию `plugins`, прежде чем активировать в разделе `rules`. К примеру для подключения плагина `putout-plugin-add-hello-world` и включения правила `add-hello-world`, достаточно указать:

```json
{
    "rules": {
        "add-hello-world": true
    },
    "plugins": [
        "add-hello-world"
    ]
}
```

#### Движок Putout

Движок `putout` это инструмент командной строки, который читает настройки, парсит файлы, загружает и запускает на выполнение плагины, после чего записывает результат работы плагинов.

Он использует библиотеку [recast](https://github.com/benjamn/recast), которая помогает осуществить очень важную задачу: после парсинга и трансформации собрать код в состояние, максимально похожее на прежнее.

#### Парсинг

Для парсинга используется [estree](https://github.com/estree/estree)-совместимый парсер, в данный момент это [espree](https://github.com/eslint/espree), от команды `eslint`.
В будущем возможно будет использоваться [cherow](https://github.com/cherow/cherow), когда выйдет вторая версия, в первой нет поддержки комментариев.
#### Перевод в формат Babel

Да, именно так 🙂. Берется `estree` формат и с помощью модуля [estree-to-babel](https://github.com/coderaiser/estree-to-babel) переводится в формат `babel`, который немного отличается.
Делается это по причине [слишком явного и частого бага](https://github.com/benjamn/recast/issues/558), что бы его можно было просто проигнорировать.

По этой причине код изначально парсится в формате `estree`, по этому любой совместимый движок подойдет (включая `babel` с плагином `estree`), что оставляет некоторую возможность для маневра
(К примеру `cherow` заявляет что значительно превосходит в скорости остальные `estree` парсеры, а конкуренция это всегда хорошо).

#### Инициализация плагинов

#### Трансформация с помощью плагинов

На этом шаге я могу объяснить почему нужен был `babel`. Дело в том, что это очень популярный продукт, значительно популярнее чем остальные похожие инструменты, развивается он гораздо стремительнее.
И [каждое новое предложение в стандарт EcmaScript не обходится без babel-плагина](https://github.com/babel/proposals). Еще у `babel` есть книга [Babel Handbook](https://github.com/jamiebuilds/babel-handbook/blob/master/translations/en/plugin-handbook.md) в которой очень неплохо описаны все возможности и инструменты для обхода и трансформации AST-дерева

#### Структура плагина

`Putout` плагин состоит из 3-ех функций:
- `report` - возвращает сообщение;
- `find` - ищет места с ошибками и возвращает их;
- `fix` - исправляет эти места;

Основной момент который стоит помнить при создании плагина для `putout` это его название, оно должно начинаться с `putout-plugin-`. Дальше может идти название операции которую плагин осуществляет, например плагин `remove-wrong` должен называться так: `putout-plugin-remove-wrong`.

Так же следует добавить в `package.json`, в секцию `keywords` добавить слова: `putout` и `putout-plugin`, а в `peerDependencies` указать `"putout": ">3.10"`, или той версии которая будет последней на момент написания плагина.

#### Пример плагина для Putout

Давайте для примера напишем плагин который будет удалять слово `debugger` из кода. Такой плагин уже есть, это [@putout/plugin-remove-debugger](https://github.com/coderaiser/putout/tree/master/packages/plugin-remove-debugger) и он достаточно прост, что бы его сейчас рассмотреть.

Выглядит он таким образом:

```js

// возвращаем ошибку соответствующую каждому из найденых узлов
module.exports.report = () => 'Unexpected "debugger" statement';

// в этой функции ищем узлы, содержащией debugger с помощью паттерна Visitor
module.exports.find = (ast, {traverse}) => {
    const places = [];
    
    traverse(ast, {
        DebuggerStatement(path) {
            places.push(path);
        }
    });
    
    return places;
};

// исправление код выглядиит таким образом
module.exports.fix = (path) => {
    path.remove();
};
```

Если правило `remove-debugger` включено в `.putout.json`, плагин `@putout/plugin-remove-debugger` будет загружен. Сперва вызовется функция `find` которая с помощью функции `traverse` обойдет узлы AST-дерева и сохранит все нужные места.

Следующим шагом `putout` обратится к `report` для получения нужного сообщения.

В случае использования флага `--fix` будет вызвана функция `fix` у плагина, и выполнится трансформация, в данном случае - удаление узла.

#### Пример теста плагина

Для того, что бы упростить тестирование плагинов был написан инструмент [@putout/test](https://github.com/coderaiser/putout/tree/master/packages/test). По своей сути это ни что иное, как [tape](https://github.com/substack/tape), в который добавлено несколько методов для удобства и упрощения тестирования.

Тест для плагина `remove-debugger` может выглядит таким образом:

```js
const removeDebugger = require('..');
const test = require('@putout/test')(__dirname, {
    'remove-debugger': removeDebugger,
});

// проверяем что бы сообщение было именно таким
test('remove debugger: report', (t) => {
    t.reportCode('debugger', 'Unexpected "debugger" statement');
    t.end();
});

// проверяем результат трансформации
test('remove debugger: transformCode', (t) => {
    t.transformCode('debugger', '');
    t.end();
});
```

#### Codemods

Не любую трансформацию нужно использовать каждый день, для разовых трансформаций достаточно сделать все тоже самое, только вместо публикации в `npm` разместить в папке `~/.putout`. При запуске `putout` посмотрит в эту папку, подхватит и запустит трансформации.

Вот пример трансформации, который заменяет подключение `tape` и [try-to-tape](https://github.com/coderaiser/try-to-tape) вызовом [supertape](https://github.com/coderaiser/supertape): [convert-tape-to-supertape](https://github.com/coderaiser/putout/tree/master/codemods/convert-tape-to-supertape).

