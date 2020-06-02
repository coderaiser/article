В 2015 году Николас Заказ опубликовал статью с [похожим названием](https://www.smashingmagazine.com/2015/09/eslint-the-next-generation-javascript-linter/), только вместо [Putout](https://github.com/coderaiser/putout) было ESLint. В те времена это было действительно так, ESLint безусловно стандарт дефакто в мире JavaScript линтеров. Однако совершенству нет предела, и любому успешному инструменту приходит на смену еще более успешный, либо дополняет его устраняя недостатки. Об одном из таких инструментов мы сегодня поговорим, однако начать хотелось бы с истории.
<cut />
**История линтеров**

Начнем с истоков, чтобы было понятно откуда ноги растут, после чего перейдем к самым популярным решениям на JavaScript.

*   [1972 - Стивен Джонсон создает первый линтер в Bell Labs](https://en.wikipedia.org/wiki/Lint_(software)), он проверяет наличие ошибок в коде на C. Слово “Lint”  означает что-то вроде “комков волокон в хлопковой ткани”. 

*   2002 - Дуглас Кроуфорд создает первый JavaScript линтер [JSLint](http://jslint.com/) с [несвободной лицензией The Software shall be used for Good, not Evil](https://en.wikipedia.org/wiki/JSLint). И сообщество с радостью (бы) приняло такой полезный инструмент, но были недостатки: правила (почти) нельзя конфигурировать, в нем минимум настроек.

*   2011 - Антон Ковалев создает форк [JSHint](https://jshint.com/) лишенный недостатка предшественника [имеет возможность обильной конфигурации](https://web.archive.org/web/20110224022052/http://anton.kovalyov.net/2011/02/20/why-i-forked-jslint-to-jshint/), однако монолитен и не расширяем.

*   2013 - Николас Закас не выдерживает отсутствия плагинов в JSHint и [пишет свой линтер](https://www.smashingmagazine.com/2015/09/eslint-the-next-generation-javascript-linter/) о котором знает каждый js-разработчик: [ESLint](https://eslint.org/).
*   2018 - Coderaiser пишет инструмент, [который удаляя неиспользуемые переменные](https://github.com/coderaiser/putout/commit/12d8fe164f2c991391596579c5a24d400a7d69f7), делает то, что ESLint, делать отказывается, а именно: [использует правила, которые могут изменить поведение кода](https://eslint.org/docs/developer-guide/working-with-rules#applying-fixes).

**Что не так с существующими линтерами**

В JavaScript Community все более-менее утряслось, и остался один линтер, выживший либо [вмердживший](https://eslint.org/blog/2016/04/welcoming-jscs-to-eslint) в себя [конкурентов](https://eslint.org/blog/2019/01/future-typescript-eslint). ESLint и правда делает очень многие вещи достаточно качественно. Но все не так просто, и есть набор моментов, который, скажем так, можно было бы улучшить:

1. Нет состояния прогресса. В 2020-ом году, мы каждый день запускаем инструмент, который неизвестно когда завершит работу. Да, на маленьких проектах это не заметно и не существенно. Но фронтенд сейчас очень сильно разросся, и плагинов, и правил существует довольно-таки много, и проверка может вполне выполняться несколько минут. Как минимум, это не удобно.
2. Переусложненная система плагинов. Да она быстрая, эффективная, но очень уж многословная: 
    - [классическая версия ESLint плагина](https://github.com/coderaiser/putout/blob/v6.12.0/packages/eslint-plugin-putout/rules/new-line-function-call-arguments.js)
    - [Putout-версия ESLint плагина](https://github.com/coderaiser/putout/blob/v8.11.0/packages/eslint-plugin-putout/lib/new-line-function-call-arguments/index.js)
    - а вот простейший плагин Putout [вполне может быть и таким](https://github.com/coderaiser/putout/blob/v8.11.0/packages/plugin-remove-useless-spread/lib/object/index.js)
3. Наличие такой концепции как *warning*: зачем? Если нашел проблему - возьми и исправь, зачем о ней говорить, помечая желтым цветом. Постоянные напоминания о чем-то, что не сильно важно не имеют смысла.
4. Отсутствие опции **--fix** для огромного количества правил. Это сейчас ESLint двигается в сторону **autofix**, но делает он это крайне плавно, и само наличие правил, которые не исправляются сами, лично меня напрягает.

**В чем ключевая идея Putout**

Владея основной информацией о том, что не так с существующими решениями, было решено сделать продукт в соответствии со следующими принципами:

1. Возможность отображения статуса прогресса.
2. Каждое правило имеет возможность исправления.
3. Правила могут минимально менять поведение кода, если это делает его лучше.
4. Правила имеют полные и конкретные названия ([convert-equal-to-strict-equal](https://github.com/coderaiser/putout/tree/master/packages/plugin-convert-equal-to-strict-equal) в противовес [eqeqeq](https://eslint.org/docs/rules/eqeqeq)).
5. Нет ворнингов, есть только ошибки.
6. Максимально дополнять ESLint, не дублируя его работу, косметические правки имеет смысл оставить ему.
7. 100% покрытие тестами (сейчас их более [1400](https://travis-ci.org/github/coderaiser/putout/jobs/692841908#L3173))
8. Писать и тестировать плагины должно быть [просто и легко](https://putout.cloudcmd.io/).
9. Маленькое ядро, вся система состоит из плагинов.
10. Интеграция с существующими решениями.

**Архитектура**

![image](https://github.com/coderaiser/putout/blob/master/images/putout.png?raw=true)

Основной механизм работы Putout состоит из работы трех модулей:

*   [parser](https://github.com/coderaiser/putout/tree/master/packages/engine-parser) - парсит код в AST, а затем AST обратно в код
*   [loader](https://github.com/coderaiser/putout/tree/master/packages/engine-loader) - загружает плагины
*   [runner](https://github.com/coderaiser/putout/tree/master/packages/engine-runner) - запускает на выполнение плагины

**Плагины**

Встроенные плагины поддерживают [4 вида интерфейсов](https://github.com/coderaiser/putout/tree/master/packages/engine-runner), начиная от [самых простых](https://github.com/coderaiser/putout/blob/v8.11.0/packages/plugin-remove-useless-spread/lib/array/index.js), и заканчивая [сложными](https://github.com/coderaiser/putout/blob/v8.11.0/packages/plugin-merge-duplicate-imports/lib/merge-duplicate-imports.js). Благодаря этому putout содержит в себе больше [50 плагинов](https://github.com/coderaiser/putout#plugins-1), каждый из которых является отдельным npm-пакетом, то есть при обновлении плагина, update придет пользователям более старых версий Putout наравне с пользователями новых версий.

**Интеграция с существующими решениями**

Putout [поддерживает](https://github.com/coderaiser/putout/tree/v8.11.0/packages/engine-parser#parsecode) все самые популярные парсеры JavaScript кода, а так же системы написания кодмодов, такие как [jscodeshift](https://github.com/facebook/jscodeshift). Кроме этого может использоваться как [плагин для ESLint](https://github.com/coderaiser/putout/tree/master/packages/eslint-plugin-putout) (благодаря чему попадает во все IDE), плагин для babel, а также имеет возможность запуска ESLint для форматирования файла, после внесения изменений, а так же встроенную поддержку работы со staged файлами в git и улучшенную работу кэша (кэшируются результаты проверки всех правил Putout, и [только встроенных правил](https://github.com/eslint/eslint/issues/10712#issuecomment-409748549) ESLint).

**Послесловие**

Я не рассказал как установить, использовать и настраивать Putout поскольку уже говорил об этом в одной из [предыдущих статей](https://habr.com/en/post/439564/), хочу лишь добавить, что все замечания написанные в комментариях были учтены и исправлены. Всем спасибо за внимание :).
