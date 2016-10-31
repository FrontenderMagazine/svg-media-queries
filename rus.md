# SVG и медиа-запросы

Одно из достоинств SVG, это то, что можно использовать медиа-запросы для реализации медиавыражений:

    <svg width="100" height="100" xmlns="http://www.w3.org/2000/svg">
      <style>
        circle {
          fill: green;
        }
        @media (min-width: 100px) {
          circle {
            fill: blue;
          }
        }
      </style>
      <circle cx="50" cy="50" r="50"/>
    </svg>

**Но когда именно кружок должен стать синим?** Согласно спецификации `min-width` [должен соответствовать ширине вьюпорта][1], но…


## Какого вьюпорта?

    <img src="circle.svg" width="50" height="50">
    <img src="circle.svg" width="100" height="100">
    <iframe src="circle.svg" width="50" height="50"></iframe>
    <svg width="50" height="50">
      …как указано выше…
    </svg>

Что из перечисленного выше нарисует (потенциально обрезанный) *синий* круг в HTML-документе? Как и чей вьюпорт следует использовать? Что нужно брать в расчет:

* Размер родительского документа в CSS
* `<svg>` aтрибуты `width`, `height`, `viewBox` 
* `<img>` атрибуты `width`, `height`
* размеры `<img>` указанные в CSS  

Вот демо того что расписанно выше: 

<p data-height="265" data-theme-id="dark" data-slug-hash="pEXrpN" data-default-tab="result" data-user="FMRobot" data-embed-version="2" data-pen-title="pEXrpN" class="codepen"><a href="http://codepen.io/FMRobot/pen/pEXrpN/">Посмотрите на CodePen</a>.</p>


### В большинстве браузеров…

Для `<img>`, SVG масштабируется до размеров изображения, а также вьюпорт для SVG в CSS до размеров `<img>`. Таким образом, первый `<img>` имеет ширину вьюпорта 50 пикселей, а второй имеет ширину 100. Это означает, что второй `<img>` подхватывает «синий» медиа-запрос, но первый этого не делает.

Для `<iframe>`, размером вьюпорта SVG является вьюпорт фрейм-документа . Таким образом, в приведенном выше примере, ширина вьюпорта в 50 пикселей CSS, потому что это ширина фрейма.

Что касается встроенного `<svg>`, то SVG не имеет своего собственного вьюпорта, он становится частью родительского документа. Это означает, что `<style>` находится в собственности родительского документа - он не ограничен областью видимости в SVG. Это привлекло мое внимание, когда я впервые использовал встроенный SVG, но это имеет смысл и [хорошо определено в спецификации][3].


### Как говорит лисичка?

У Firefox своя точка зрения. Он ведет себя так, как описано выше, за одним исключением:

Для `<img>`, вьюпорт это отрендеренный размер в пикселях устройства, а это означает изменение отображения в зависимости от плотности пикселей экрана. Первое изображение в демке будет зеленым на 1x экранах, но станет синего цвета на экранах 2x и выше. Это проблема, так как некоторые ноутбуки и большинство телефонов имеют плотность пикселей больше чем 1.

Выглядит ошибкой, тем более, что Firefox не применяет ту же логику к `iframe`, но мы должны признать в Firefox некоторую слабину их спецификации, она на самом деле не покрывает случаи когда SVG используется в `<img>` - должны масштабироваться, не говоря уже, о медиа запросах, которые должны обрабатываться.

Я [создал ишью для спецификации][4], будем надеяться, что это устронят.

Но все становится гораздо сложнее, когда вы начинаете…


## Рисовать SVG на canvas

Можно рисовать `<img>` в `<canvas>`, вот так:

    canvas2dContext.drawImage(img, x, y, width, height);

**Но когда круг должен стать синим?** На этот раз больше вьюпортов. Что из этого играет роль:

* Размер родительского окна в CSS
* Атрибуты `width`, `height` и `viewbox` у `<svg>`
* Атрибуты `width` и `height` тега `<img>` 
* Заданные в CSS размеры `<img>` 
* Плотность пикселей `<canvas>` 
* Заданные в CSS размеры `<canvas>` 
* Ширина, высота указаные в `drawImage` 
* Ширина, высота указаные в `drawImage`, учитывая примененные к двухмерному контексту трансформации

Как вы думаете? Опять же, спецификация неясна, и на этот раз каждый браузер пошел по своему собственному пути. Давайте ознакомимся:

<p data-height="265" data-theme-id="dark" data-slug-hash="VKJMMY" data-default-tab="result" data-user="FMRobot" data-embed-version="2" data-pen-title="VKJMMY" class="codepen"><a href="http://codepen.io/FMRobot/pen/VKJMMY/">Посмотрите на CodePen</a>.</p>

Насколько я могу судить, вот что браузеры делают:


### Chrome

Chrome отталкивается от атрибутов width/height, указанных в SVG документе. Это означает, что если в SVG документе указана `width="50"`, сработает медиа-запрос для вьюпорта шириной 50px. Если вы хотите отобразить его так, что бы сработал медиа-запрос для вьюпорта шириной 100px, незадача. Независимо от того, какой размер вы используйте в `canvas`, он будет рисовать, используя медиа-запрос для 50px ширины.

Однако, если в SVG задан атрибут `viewBox`, а не фиксированная ширина, Chrome использует плотность пикселей `<canvas>` в качестве ширины вьюпорта. Можно утверждать, что это подобно тому, будто все работает с инлайн SVG, где окно вьюпорта это все окно браузера, но переключение поведения, основанного на `viewBox` действительно странно.

Chrome выигрывает награду странные-штанишки за «шаткое поведение».


### Safari

Как и Chrome, Safari использует размер, указанный в документе SVG, с теми же недостатками. Но если SVG использует viewBox, а не фиксированую ширину, он вычисляет ширину на основе viewBox, так что SVG с viewBox = "50 50 200 200" будет иметь ширину 150.

Не так странно как в Chrome, но всё равно ограничивает.


### Firefox

Firefox использует ширину и высоту, указанную в вызове `drawImage`, с учетом любых трасформаций. Это означает, что если вы рисуете вашу SVG, так, что размер холста 300 пикселей в ширину, он будет иметь ширину вьюпорта в 300px.

Это своего рода отражает их странное поведение `<img>` — это основано на  отрисовке пикселей. Это означает, что вы получите те же несоответствия плотности, если умножите ширину вашего холста и высоту `devicePixelRatio` (и масштаб упадет обратно с помощью CSS), вы должны это сделать, что бы избежать размытости на экранах с высокой плотностью:

<p data-height="265" data-theme-id="dark" data-slug-hash="vXqWBg" data-default-tab="result" data-user="FMRobot" data-embed-version="2" data-pen-title="vXqWBg" class="codepen"><a href="http://codepen.io/FMRobot/pen/vXqWBg/">Посмотрите на CodePen</a>.</p>

Есть смысл в том, что делает Firefox, но это означает, что медиа-запросы привязаны к пикселям.


### Microsoft Edge

Egde использует размер элемента `<img>` для определения размеров вьюпорта. Если `<img>` не имеет размеров (`display: none` или элемент вне документа), то он возвращается к атрибутам `width` и `height`, если их нет, тогда он использует внутренние размеры `<img>`.

Это означает, что вы можете сделать SVG размером 1000x1000, но изображение `<img width= "100">`, вьюпорт будет иметь ширину 100px.

На мой взгляд, это идеальный вариант. Это означает, что вы можете активировать медиа-запросы и не зависить от ширина русинка. Это, кроме того, соответствует поведению адаптивных изображений. Когда вы рисуете `<img srcset= "…" width="…">` на холсте, все браузеры соглашаются, что изображение должно содержать ресурс из `<img>`.


## Фух!

Я [создал ишью с предложением принять поведение Edge][6], и [предложил дополнение к `createImageBitmap` которое позволит задать вьюпорт из скрипта][7]. Надеюсь, мы сможем добиться большей кроссбраузерности!

Для полноты картины, [вот как я собрал данные][8], а [вот полные результаты][9].


 [1]: https://drafts.csswg.org/mediaqueries-3/#width
 [2]: img/fixed100.4b1cb7cb9384.svg
 [3]: https://svgwg.org/svg2-draft/styling.html#StyleSheetsInHTMLDocuments
 [4]: https://github.com/w3c/svgwg/issues/289
 [5]: img/text.941f43fc7ea8.svg
 [6]: https://github.com/whatwg/html/issues/1880
 [7]: https://github.com/whatwg/html/issues/1881
 [8]: http://jsbin.com/gefaju/2/edit?js,output

 [9]: https://docs.google.com/spreadsheets/d/15IkG42KrEWgv_FbrgfGBSM_PYRi22Vj_uGrcp4LxyMU/edit#gid=0