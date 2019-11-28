## Change detection в Angular

Change detection Change detection ( презентация от Pascal Precht) - это процесс отобразить изменение модели на представление (DOM).

Поставим вопрос: что может породить изменения в модели приложения? Ответ: События (действия юзера), таймеры, http. Как Angular узнает о том, что данные действия произошли?

**Концепция Zone**

Angular основывается на концепции Zone (zone.js - отдельная библиотека с небольшим кол-вом кода). Суть: для того чтобы знать, например, когда закончится выполнение функции по setTimeout мы можем переопределить setTimeout. То есть когда мы оборачиваем код в zone и это означает, что setTimeout уже пропатчен и мы можем знать, когда закончится выполнение setTimeout:

```js
// обертка:
const oldSetTimeout = setTimeout;

setTimeout = (cb, time) => {
    console.log('setTimeout start');
    oldSetTimeout(() => {
        cb();
        console.log('setTimeout finish');
    }, time);
}
```

Когда мы заворачиваем код в zone, то это означает, что setTimeout пропатчен и раз он пропатчен мы можем узнать, что он выполнился в конкретной zone.....

```js
zone.run(() => {
    foo();
    setTimeout(doSth, 0);
    bar();
})
```

**Как использует zone Angular?**

У Angular есть сущность ```NgZone```- ответвление от zone, но основанное на ```Observable```.
Основной метод NgZone - ```onMicrotaskEmpty```, на который подписан zone и выполняет метод ```tick```. Метод tick запускает весь механизм синхронизации - **change detection**.

```js
this.zone.onMicrotaskEmpty
        .subscribe(() => {
            this.zone.run(() => this.tick());
    })
```

```zone```и ```change detection``` это **две разные сущности**: например, вы можете выключить zone и работать с change detection самостоятельно (zone помогают запускать change detection).

change detection: у каждого компонента свой **Change Detector**. Когда происходит событие в одном из компонентов zone об этом узнает и zone запускает change detection (CD) и CD идет вниз по всем Change Detector (дочерних компонентов). У Angular одна зона и куча Change Detector.

Инструкция, что в данном методе не нужно запускать Change Detector:

```js
constructor(zone: NgZone) {
    zone.outsideAngular(() => {
        //....
    })
}
```

Вручную дергаем change detection:

```js
constructor(app: ApplicationRed) {
    app.tick();
}
```

**mutable && immutable objects**

Стратегия **ChangeDetectionStrategy.OnPush** на компоненте позволяет избежать лишних проверок на изменение - в результате чего Angular начинает работать гораздо быстрее.

На чем основана стратегия ChangeDetectionStrategy.OnPush:

**immutable** - мы не можем изменить свойства в уже существующем объекте, то есть чтобы работать со свойствами этого объекта мы должны склонировать (user2 = clone(user).name = 'Иван') этот объект.
когда объект mutable, то если меняется свойство объекта Angular нужно запускать сложное сравнение по вложенным свойствам и в конце концов татже запускать change detection на соотв-м компоненте.

Но что если мы используем *immutable objects*, то есть в результате два объекты становятся не равными (и Angular достаточно просто сравнить два объекта **по ссылке**):

```js
@Component({
    selector: 'app-root',
    // стратегия ChangeDetectionStrategy.OnPush говорит Angular о том,
    // что используем (передаем на @Input) только immutable объекты (то есть если данные не изменятся на компоненте не будет
    // запущен change detection)
    changeDetection: ChangeDetectionStrategy.OnPush
})
```

То есть если ссылка на объект не изменилась, то при стратегии ChangeDetectionStrategy.OnPush на компоненте не будет запускаться change detection.

Может пригодится - библиотека immutable.js, чтобы работать с immutable объектами. Или при помощи натива ```Object.assign```.

**Как работать с Observable при стратегии OnPush?**

Что делать с ```Observable```, который может передаваться как параметр? Мы можем:


```js
// cd - ChangeDetector конкретного компонента
constructor(private cd: ChangeDetectorRef) {

}
ngOnInit() {
    this.opStream.subscribe(() => {
        // markForCheck - помечает всю ветку, начиная с компонента и вверх по дереву до корневого, специальным маркером
        // далее, когда выполняется change detection, change detection выполняется только по этой ветке
        // после этого маркер убирается
        this.cd.markForCheck();
    })
}

ngOnInit() {
    // this.cd.detach() - выкинет компонент из дерева change detection
    // (то есть этим мы можем выключить  change detection на время выполнения тяжелой логики)

    // this.cd.reattach() - включит change detection
}



update() {
    // вручную вызываем change detection на конкретном компоненте
    this.cd.detectChanges();
}
```