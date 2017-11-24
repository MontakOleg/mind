## Layouts


### What we have?

UI можно определить как набор вьюх, которые отображают данные, реагируют на события и при этом каким-то определенным образом расположены на экране.

Как мы размещаем элементы на экране?
* Qt предлагает использовать контейнеры, умеющие внутри себя располагать элементы определенным образом. Вкладывая эти контейнеры друг в друга можно получить необходимую расстановку.

* Android предлагает аналогичные контейнеры.

* WPF/XAML предлагает аналогичные контейнеры.

* iOS предлагает AutoLayout. Нужно описать набор ограничений (уравнений), непротиворечиво и однозначно описывающих расположение элементов. Решив систему уравнений с этими ограничениями, движок получит координаты и размеры элементов.

* у Delphi есть anchors – прибивание краев к контейнеру. Очень похоже на resizing masks в iOS.

* В Web на сколько я понимаю – вкладывание друг в друга контейнеров, поведение которых описывается стилями.


### What problem with this?

_We need to use code for special cases_

Описанные инструменты заточены под типовые случаи, зачастую мы не можем (или можем, но это сильно неудобно) описать расположение какого-то элемента с помощью этих инструментов. Приходится лезть в код и рассчитывать координаты вручную. Логика описания layout'а размазывается по нескольким местам.
Должен быть способ лучше.


### Layout - what is it really?

_Layout is function_

Что такое layout? Это описание расположения элементов. Его можно выразить в виде функции.
На вход даем размер контейнера и массив элементов, на выходе получаем координаты этих элементов.
Теоретически layout может быть любой – тут уж как дизайнер нарисует. Поэтому чтобы описать такую функцию нам нужен тьюринг-полный язык. UI фреймворки предлагают нам что угодно, только не тьюринг-полный язык, отсюда и проблемы. Логично было бы взять язык, на котором пишется остальная часть программы: для iOS objc/swift, для android java/kotlin и т.д.


### What does it mean?

_We can easy describe layouts in code_

Мы можем легко описывать layout в коде! Достаточно написать одну функцию. Часто мы будем располагать элементы относительно существующих (правее, ниже и т.д.). Можно спрятать такие вычисления в функции с читабельными именами. Под iOS это сделано в библиотеках [Facade (objc)](https://github.com/mamaral/Facade), [Neon (swift)](https://github.com/mamaral/Neon). Можно легко написать свою на любом языке. У многих есть негативный опыт программирования UI в коде, предполагаю что причиной этого было использование антипаттернов.


### Anti-patterns

* Смешивание создания контролов, настройки их стилей, настройки на данные и лайаут. Все в одной функции вперемешку. Предлагаю в каждой вью создавать отдельные методы `createSubviews()`, `configureForModel()`, `layoutSubviews()`
* Вычисление фреймов руками на низком уровне, без удобных функций-оберток.
* xib способствуют созданию больших сложных вью с глубокой иерархией. Используйте декомпозицию.
* Следствие из пред. пункта – управление вьюшками глубже первого уровня. Надо управлять только непосредственными детьми. Можно нарушать в тривиальных случаях.
* Следствие из пред. пункта – setNeedsLayout/LayoutIfNeeded везде и всюду, в попытках добиться необходимого поведения


### Example, libraries

В iOS если вы хотите определить кастомный layout для view, нужно переопределить метод `layoutSubviews()` (или `viewDidLayoutSubviews()` у контроллера) и задать в нем frame дочерним элементам. Этот метод вызывается при изменении размеров view, то есть повороты, split-view и пр. зафорсят вызов этого метода. 

Вот пример с официальной страницы Neon:

```swift
let isLandscape : Bool = UIDevice.currentDevice().orientation.isLandscape.boolValue
let bannerHeight : CGFloat = view.height() * 0.43
let avatarHeightMultipler : CGFloat = isLandscape ? 0.75 : 0.43
let avatarSize = bannerHeight * avatarHeightMultipler

searchBar.fillSuperview()
bannerImageView.anchorAndFillEdge(.Top, xPad: 0, yPad: 0, otherSize: bannerHeight)
bannerMaskView.fillSuperview()
avatarImageView.anchorInCorner(.BottomLeft, xPad: 15, yPad: 15, width: avatarSize, height: avatarSize)
nameLabel.alignAndFillWidth(align: .ToTheRightCentered, relativeTo: avatarImageView, padding: 15, height: 120)
cameraButton.anchorInCorner(.BottomRight, xPad: 10, yPad: 7, width: 28, height: 28)
buttonContainerView.alignAndFillWidth(align: .UnderCentered, relativeTo: bannerImageView, padding: 0, height: 62)
buttonContainerView.groupAndFill(group: .Horizontal, views: [postButton, updateInfoButton, activityLogButton, moreButton], padding: 10)
buttonContainerView2.alignAndFillWidth(align: .UnderCentered, relativeTo: buttonContainerView, padding: 0, height: 128)
buttonContainerView2.groupAndFill(group: .Horizontal, views: [aboutView, photosView, friendsView], padding: 10)
```

![portrait](https://github.com/mamaral/Neon/raw/master/Screenshots/portrait.png)
![landscape](https://github.com/mamaral/Neon/raw/master/Screenshots/landscape.png)

Неплохо для 10 строк кода, правда?


### Pros

* Вычислять фреймы – это очень быстро. Если верить [LayoutFrameworkBenchmark](https://github.com/lucdion/LayoutFrameworkBenchmark) ручной расчет фреймов в 10-15 раз быстрее, чем AutoLayout.
* Код layout'а в одном месте – проще поддерживать.
* Легкая отладка.
* Полный контроль, можно описать какой угодно layout.
* Написание view в коде естественным образом ведет к декомпозиции и переиспользованию view.
* Написание view в коде упрощает использование общих ресурсов (шрифты, цвета, etc.)


### Cons

Часто UI описывается декларативно в виде разметки (xml, xaml, xib), которая потом превращается в код (qt, delphi, c# winforms) или из которой создается готовый граф объектов (xib, xaml). При этом IDE предоставляют средства предпросмотра, что весьма удобно.
Описывая layout в коде мы отказываемся от инструментов предпросмотра.

Обычно layout идет сверху вниз: у нас есть размер экрана, зная этот размер мы layout'им первый слой вьюх, получив их размеры заходим глубже и т.д. Бывает нужно идти с другой стороны: растягивать контейнер по размеру содержимого. Описывать такое не всегда удобно.


### What about RTL, split view, iPhone X ?

* RTL – можно обрабатывать вручную; для Neon висит [PR](https://github.com/mamaral/Neon/pull/56) для использования leading, trailing как в autolayout.
* Split view – можно ловить изменение trait collection и использовать разные функции layout.
* iPhone X – safeAreaGuide доступен внутри view, можно учитывать соответствующие отступы.


### Conclusion

На мой взгляд это хороший подход, если вы уже пишете UI в коде. Кода при этом получается примерно столько же как при использовании AutoLayout, но работает он быстрее и предсказуемее.
Главный минус – теряется наглядность в виде инструментов предпросмотра.

