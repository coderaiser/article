# Автоматизируем переход на react-hooks

## Введение

[React 6.18](https://reactjs.org/blog/2019/02/06/react-v16.8.0.html) - первый стабильный релиз с поддержкой [react hooks](https://reactjs.org/docs/hooks-intro.html). Теперь хуки можно использовать не опасаясь, что API изменится кардинальным образом. И хотя команда разработчиков `react` советует использовать новую технологию лишь для новых компонентов, многим, в том числе и мне, хотелось бы их использовать и для старых компонентов использующих классы. Но поскольку ручной рефакторинг - это не очень весело, мы попробуем автоматизировать этот процесс.

## Особенности react-hooks

В статье [Введение в React Hooks](https://habr.com/en/post/429712/) очень подробно рассказано, что это за хуки, и с чем их едят. В двух словах, это новая безумная технология создания компонентов, имеющих `state`, без использования классов.

Рассмотрим файл `button.js`:

```javascript
import React, {Component} from 'react';
export default Button;

class Button extends Component {
    constructor() {
        super();
        
        this.state = {
            enabled: true
        };
        
        this.toogle = this._toggle.bind(this);
    }
    
    _toggle() {
        this.setState({
            enabled: false,
        });
    }
    
    render() {
        const {enabled} = this.state;
        
        return (
            <button
                enabled={enabled}
                onClick={this.setEnabled}
            />
        );
    }
}
```

С хуками он будет выглядеть таким образом:

```javascript
import React, {useState} from 'react';
export default Button;

function Button(props) {
    const [enabled, setEnabled] = useState(true);
    
    function toggle() {
        setEnabled(false);
    }
    
    return (
        <button
            enabled={enabled}
            onClick={setEnabled}
        />
    );
}
```

Можно долго спорить насколько такой вид записи более очевиден для людей, незнакомых с технологией, но одно понятно сразу: код более лаконичен, и его проще переиспользовать. Интересные наборы пользовательских хуков можно найти на [usehooks.com](https://usehooks.com/) и [streamich.github.io](https://streamich.github.io/).

Далее мы разберем в мельчайших подробностях все различия, и разберемся с процессом программного преобразования кода, но перед этим мне хотелось бы рассказать о том, что привело к такой форме записи, с точки зрения синтаксиса.

## Лирическое отступление: нестандартное использование синтаксиса деструктуризации

`ES2015` подарил миру такую замечательную вещь как [деструктуризация массивов](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Destructuring_assignment). И теперь вместо того, что бы доставать каждый элемент по отдельности:

```javascript
const letters = ['a', 'b'];
const first = letters[0];
const second = letters[1];
```

Мы можем достать сразу все нужные элементы:

```javascript
const letters = ['a', 'b'];
const [first, second] = letters;
```

Такая записать не только более лаконичная, но и менее подвержена ошибкам, поскольку убирает необходимость в том, что бы помнить об индексах элементов, и позволяет сосредоточится на том, что действительно важно: инициализации переменных.

Таким образом мы приходим к тому, что если бы не `es2015` команда реакта не придумала такой необычный способ работы со стейтом.

Далее я хотел бы рассмотреть несколько библиотек которые используют похожий подход.

#### Try Catch

За полгода до объявления о хуках в реакте, мне пришла в голову идея, что деструктуризацию можно использовать не только для того, что бы
достать из массива однородные данные, но и для того, что бы доставать информацию об ошибке или результат выполнения функции, по аналогии с коллбэками в `node.js`. К примеру, вместо того, что бы использовать синтаксическую конструкцию `try-catch`:

```javascript
let data;
let error;

try {
    data = JSON.parse('xxxx');
} catch (e) {
    error = e;
}
```

Что выглядит очень громоздко, при этом несет достаточно мало информации, и заставляет нас использовать `let`, хотя менять значения переменных мы не планировали. Вместо этого, можно вызвать функцию [try-catch](https://github.com/coderaiser/try-catch), которая сделает все что нужно, избавив нас от перечисленных выше проблем:

```javascript
const [error, data] = tryCatch(JSON.parse, 'xxxx');
```

Таким вот интересным способом мы избавились от всех не нужных данных, оставив лишь необходимое. Такой способ обладает следующими преимуществами:
- возможно задавать любые удобные нам имена переменных (при использовании деструктуризации объектов, у нас такой привилегии не было бы, вернее у нее была бы своя громоздкая цена);
- возможность использовать константы для данных которые не меняются;
- более лаконичный синтаксис, отсутствует все что можно было убрать;

И опять же все то благодаря синтаксису деструктуризации массивов. Без этого синтаксиса, использование библиотеки выглядело бы нелепо:

```javascript
const result = tryCatch(JSON.parse, 'xxxx');
const error = result[0];
const data = result[1];
```

Это все еще допустимый код, но он значительно теряет по сравнению с деструктуризацией.
Хочу еще добавить пример работы библиотеки [try-to-catch](https://github.com/coderaiser/try-to-catch) с приходом `async-await` конструкция `try-catch` все еще актуальна, и может быть записана таким образом:

```javascript
const [error, data] = await tryToCatch(readFile, path, 'utf8');
```

Если идея такого использования деструктуризации пришла мне, то почему ей не прийти и создателям реакта, ведь по сути, мы имеешь что-то типо функции которая имеет 2 возвращаемых значения: [кортеж](https://ru.wikibooks.org/wiki/Haskell/ListsAndTuples#%D0%9A%D0%BE%D1%80%D1%82%D0%B5%D0%B6%D0%B8) из хаскеля.

На этом лирическое отступление можно закончить и перейти к вопросу преобразования.

## Преобразование класа в react-hooks

Для преобразования мы будем использовать AST-трансформатор [putout](https://habr.com/en/post/439564/), позволяющий менять только то, что необходимо и [@putout/plugin-react-hooks](https://github.com/coderaiser/putout/tree/master/packages/plugin-react-hooks).

Для того, что бы преобразовать класс наследуемый от `Component` в функцию, использующую `react-hooks` необходимо проделать следующие шаги:

- удалить `bind`
- переименовать приватные методы в публичные (убрать "_");
- поменять `this.state` на использование хуков
- поменять `this.setState` на использование хуков
- убрать `this` отовсюду
- конвертировать `class` в функцию
- в импортах использовать `useState` вместо `Component`

### Подключение

Установим `putout` вместе с плагином `@putout/plugin-react-hooks`:

```sh
npm i putout @putout/plugin-react-hooks -D
```

После чего попробуем `putout` в действии.

```sh
coderaiser@cloudcmd:~/example$ putout button.js
/home/coderaiser/putout/packages/plugin-react-hooks/button.js
 11:8   error   bind should not be used                                     react-hooks/remove-bind
 14:4   error   name of method "_toggle" should not start from under score  react-hooks/rename-method-under-score
 7:8    error   hooks should be used instead of this.state                  react-hooks/convert-state-to-hooks
 15:8   error   hooks should be used instead of this.setState               react-hooks/convert-state-to-hooks
 21:14  error   hooks should be used instead of this.state                  react-hooks/convert-state-to-hooks
 7:8    error   should be used "state" instead of "this.state"              react-hooks/remove-this
 11:8   error   should be used "toogle" instead of "this.toogle"            react-hooks/remove-this
 11:22  error   should be used "_toggle" instead of "this._toggle"          react-hooks/remove-this
 15:8   error   should be used "setState" instead of "this.setState"        react-hooks/remove-this
 21:26  error   should be used "state" instead of "this.state"              react-hooks/remove-this
 26:25  error   should be used "setEnabled" instead of "this.setEnabled"    react-hooks/remove-this
 3:0    error   class Button should be a function                           react-hooks/convert-class-to-function

✖ 12 errors in 1 files
  fixable with the `--fix` option
```

`putout` нашел 12 мест которые можно поправить, попробуем:

```sh
putout --fix button.js
```

Теперь `button.js` выглядит таким образом:

```javascript
import React, {useState} from 'react';
export default Button;

function Button(props) {
    const [enabled, setEnabled] = useState(true);
    
    function toggle() {
        setEnabled(false);
    }
    
    return (
        <button
            enabled={enabled}
            onClick={setEnabled}
        />
    );
}
```

### Настройка

## Заключение

