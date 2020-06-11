Недавно наткнулся на [статью](https://habr.com/en/post/481696/) про [статический анализатор Ruleguard](https://github.com/quasilyte/go-ruleguard/) и хотел написать к ней комментарий, но получилась статья.

Интересно, что похожие идеи могут в одно время прийти разным людям, пишущим на разных языках. Я работаю над [статическим анализатором](https://habr.com/en/post/504594/) [Putout](https://github.com/coderaiser/putout), и обратил внимание, что синтаксис, используемый в этих двух инструментах очень похож. В статье будет много кода на `JavaScript` и `Go`, языке который я совсем не знаю, тем не менее схожесть используемых подходов, не могла не вызвать у меня интерес.

А в конце вас ждет опрос :).
 <cut/>

## Время жизни

Рассмотрим хронологию:

- 30.11.2018 - [первый коммит Putout](https://github.com/coderaiser/putout/commit/12d8fe164f2c991391596579c5a24d400a7d69f7) 
- 12.12.2019 - [первый коммит Ruleguard](https://github.com/quasilyte/go-ruleguard/commit/28262423d199e704e3b345e2f535e65b693a3b7b)

Как не сложно заметить, разработка Putout началась на год раньше Ruleguard, поэтому дальнейшие сравнения, нацелены на то, что бы показать схожести и отличия, но никак не качество и обилие функционала.

## Правила

Сравним примеры правила, которое упрощает выражение `!!x` до `x`.
Для `ruleguard`, [код](https://github.com/quasilyte/go-ruleguard/blob/master/docs/gorules.md) будет выглядеть таким образом:

```go
package gorules

import "github.com/quasilyte/go-ruleguard/dsl/fluent"

func simplify(m fluent.Matcher) {
    m.Match(`!!$x`)
     .Suggest(`$x`)
     .Report(`can simplify !!$x to $x`)
}
```
А вот (упрощенный и адаптированный) пример правила putout [remove-double-negations](https://github.com/coderaiser/putout/tree/master/packages/plugin-remove-double-negations):

```javascript
module.exports.report = () => 'can simplify !!x to x';
module.exports.replace = () => ({
    '!!__x': '__x'
});
```

Обратите внимание, что `ruleguard`, использует `$x` для обозначения переменных (*интересно разворачивает ли функция Report переменную `$x`*), а putout, для той же цели, использует двойное нижнее подчеркивание __x. Почему в putout не был выбран символ `$`? Одна из причин это наличие валидных, (хоть и не часто) используемых идентификаторов в [популярных библиотеках](https://jquery.com/), а еще похожесть на один очень практичный язык программирования, где все переменные начинаются с этого символа :), а так же выразительность двойного подчеркивания. [Тут](https://github.com/coderaiser/putout/tree/master/packages/compare#supported-template-variables) список всех поддерживаемых переменных.

Судя по всему, автор ruleguard тоже изначально использовал  '__', но [в какой-то момент отдал предпочтение знаку доллара](https://github.com/quasilyte/go-ruleguard/blob/a29d6cc795181132f4d775e4396187ce5f93751b/ruleguard/typematch/typematch.go#L74), который лаконичнее.

## Работа с AST

[Вначале](https://habr.com/en/post/439564/) правила `putout`, писались таким образом, что `AST` парсилось отдельно каждым плагином, таким образом:

```javascript
// возвращаем ошибку соответствующую каждому из найденых узлов
module.exports.report = () => 'Unexpected "debugger" statement';

// ищем узлы, содержащией debugger с помощью паттерна Visitor
module.exports.find = (ast, {traverse}) => {
    const places = [];

    traverse(ast, {
        DebuggerStatement(path) {
            places.push(path);
        }
    });

    return places;
};

// удаляем код, найденный в предыдущем шаге
module.exports.fix = (path) => {
    path.remove();
};
```

[Как верно заметил](https://habr.com/en/post/439564/) @Alternator:

> Каждый плагин выполняет свой `traverse`, что медленнее чем один комбинированный `traverse`, выполненный ядром.

Поэтому, обход всего AST (по возможности) [выполняется единожды](https://github.com/coderaiser/putout/tree/v8.17.0/packages/engine-runner), поочередно заходя в нужные плагины (что позволяет [60-ти плагинам](https://github.com/coderaiser/putout#built-in-transforms) отрабатывать за вменяемое время), которые в самом простом виде могут выглядеть так:

```javascript
module.exports.report = () => 'Unexpected "debugger" statement';
module.exports.replace = () => ({
    'debugger': '',
});
```

Однако в `ruleguard`, судя по всему, каждый раз при вызове `Match` выполняется обход всего дерева, в поисках нужного места кода, @quasilyte, это так, или я не до конца понимаю механизм?

Ели это так, то:
- да, это медленно
- это не дает контекста

Дело в том, что при разработке плагинов гораздо чаще появляется необходимость детально исследовать найденные узлы, как (и сколько раз) переменные используются, где они объявлены (и объявлены ли), в каком окружении используются. К примеру в плагине [convert-for-in-to-for-of](https://github.com/coderaiser/putout/blob/v8.17.0/packages/plugin-convert-for-in-to-for-of), который делает следующее:

```diff
-for (const name in object) {
-   if (object.hasOwnProperty(name)) {
+for (const name of Object.keys(object)) {
    console.log(a);
-   }
}
```

Работает таким образом:

```javascript
const {
    generate,
    operator,
} = require('putout');

const {
    contains,
    getTemplateValues,
} = operator;

module.exports.report = () => `for-of should be used instead of for-in`;

// отфильтровуем необходимые данные
// важно что бы for-in имел `hasOwnProperty`
// иначе реализация for-of будет работать иначе
module.exports.match = () => ({
    'for (__a in __b) __body': ({__a, __b, __body}) => {
        const declaration = getTemplateValues(__a, 'var __a');
        const {name} = declaration.__a;
        
        return contains(__body, [
            `if (${__b.name}.hasOwnProperty(${name})) __body`,
        ]);
    },
});

// выполняем замену, если функция прошла фильтр
module.exports.replace = () => ({
    'for (__a in __b) __body': ({__b, __body}) => {
        const [first] = __body.body;
        const condition = getTemplateValues(first, 'if (__b.hasOwnProperty(__a)) __body');
        const {code} = generate(condition.__body);
        
        return `for (const ${condition.__a.name} of Object.keys(${__b.name})) ${code}`;
    },
});
```

Конечно, в `Go` нет таких конструкций (там другие не менее полезные :)), однако необходимость в том, что бы знать больше об узлах, которые обрабатываются остается прежней, и мне пока не понятно  как это можно сделать с помощью функции `Match` из `ruleguard`. На помощь могут прийти [фильтры](https://github.com/quasilyte/go-ruleguard/blob/v0.1.1/docs/gorules.md#filters), но они в основном связаны с типами. Возможно я просто пока не понял, как эту функциональность можно расширить, либо этот функционал (на что я надеюсь) появится в новых версиях Ruleguard.

## Плагины

`Putout` поддерживает 5 видов подгрузки плагинов:
- @putout/plugin-name - core-плагины из папки `node_modules`
- `putout-plugin-name`- `userland`-плагины из `node_modules`
- `~/.putout` - `codemods`, которые нужны для текущих проектов
- с помощью флага `--rulesdir`, к примеру, из папки текущего проекта
- с помощью параметра командной строки `-t, --transform` 

По поводу `ruleguard` @quasilyte говорит следующее:

> Правила ruleguard подгружаются на старте, из [специального gorules файла](https://github.com/quasilyte/go-ruleguard/blob/v0.1.1/rules.go), декларативно описывающего паттерны кода, на которые стоит выдавать предупреждения. Этот файл может свободно редактироваться пользователями ruleguard.

И в этом есть плюс и минус:
- плюс в том, что можно правила обработки проекта записать в специальный файл, писать туда правила и переиспользовать их для нового кода;
- минус - в невозможности переиспользования написанного кода для других проектов (файл необходимо копировать, к файлу не понятно как писать тесты, и не понятно как выбрать нужные правила для использования в других проектах)

## Командная строка

[Gogrep](https://habr.com/en/post/505652/) (на котором основан Ruleguard) вдохновил меня на то, что бы добавить возможность обрабатывать данные из командной строки в Putout, таким образом:

```sh
putout --transform 'module.exports = __a -> const cheers = __a' index.js
```
Ищем конструкцию `module.exports = __a`, где `__a` - любая валидная конструкция JavaScript. Для автоматического исправления достаточно передать флаг `fix`, и тогда код:

```javascript
module.exports = {
    hello: 'world',
};
```

Превратится в:

```javascript
const cheers = {
    hello: 'world',
};
```

## Послесловие

В целом я считаю, что это правильное направление: работа с AST самым простым из возможных путей, и популяризация идеи в массы :). На `Go` никогда не писал, но смотрю, что для работы с `AST`, есть все необходимое при чем, как я понимаю, из коробки.

Хочу поблагодарить читателя, очутившегося на этих строках, @quasilyte, за создание такого замечательного инструмента, как Ruleguard, надеюсь на дальнейшее развитие и популяризацию автоматизации рефакторинга. А еще на то, что эта статья добавит пищу для размышлений, и возможно, вдохновит на развитие новых идей, инструментов и функциональных возможностей в области статического анализа.

Буду рад любым комментариям с предложениями по обработке синтаксиса на JavaScript (и Go, если @quasilyte подключится к беседе).

И, на последок, опрос.
