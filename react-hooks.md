# Автоматизируем переход на react-hooks

## Введение

[React 6.18](https://reactjs.org/blog/2019/02/06/react-v16.8.0.html) - первый стабильный релиз с поддержкой [react hooks](https://reactjs.org/docs/hooks-intro.html). Теперь их можно использовать не опасаясь, что в их API что-то изменится кардинальным образом. И хотя команда разработчиков `react` советует использовать хуки лишь для новых компонентов, многим, в том числе и мне, хотелось бы их использовать и для старых компонентов использующих классы. Но поскольку ручной рефакторинг - это не очень весело, мы попробуем автоматизировать этот процесс.

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

Далее мы разберем в мельчайших подробностях все различия, и разберемся с процессом программного преобразования кода.

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

