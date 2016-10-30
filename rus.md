Одна из хороших штук в SVG, это то, что вы можете использовать media-запросы, чтобы добавить отзывчивости для изображений:

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
    

**Но когда кружок должен стать синим?** В спеке говорят, что `min-width` 
[должен соответствовать ширине вьюпорта][1], но…

## Какой вьюпорт?

    <img src="circle.svg" width="50" height="50">
    <img src="circle.svg" width="100" height="100">
    <iframe src="circle.svg" width="50" height="50"></iframe>
    <svg width="50" height="50">
      …как указано выше…
    </svg>

Что из перечисленного выше нарисует (потенциально обрезанный) *синий* круг в HTML-документе? Как и чей вьюпорт следует использовать? Должно быть:


* Размер родительского документа в CSS
* `<svg>` aтрибуты ширины, высоты, viewBox 
* `<img>` атрибуты ширины, высоты 
* `<img>` атрибуты размеров указанные в CSS  

Вот демо того что расписанно выше: 

http://codepen.io/anon/pen/qaQBQd

### Большинство браузеров говорят…

Для `<img>`, SVG масштабируется до размеров изображения, а также вьюпорт для SVG в CSS до размеров `<img>`. Таким образом, первый `<img>` имеет ширину вьюпорта 50 пикселей, а второй имеет ширину 100. Это означает, что второй `<img>` подхватывает «синий» медиа-запрос, но первый этого не делает.

Для `<iframe>`, размером вьюпорта SVG является вьюпорт фрейм-документа . Таким образом, в приведенном выше примере, ширина вьюпорта в 50 пикселей CSS, потому что это ширина фрейма.

Что касается встроенного `<svg>`, то SVG не имеет своего собственного значения вьюпорт, он становится частью родительского документа. Это означает, что `<style>` находится в собственности родительского документа - он не ограничен областью видимости в SVG. Это привлекло мое внимание, когда я впервые использовал встроенный SVG, но это имеет смысл и [хорошо определено в спецификации][3].

### Что же говорит лиса?

Firefox has other ideas. It behaves as above, except:

For `<img>`, the viewport is the rendered size in device pixels, meaning
the viewport changes depending on display density. The first image in the demo 
will appear green on 1x screens, but blue on 2x screens and above. This is a 
problem as some laptops and most phones have a pixel density greater than 1.

This feels like a bug, especially as Firefox doesn't apply the same logic to
the iframe, but we have to cut Firefox some slack here as the spec doesn't 
really cover how SVG-in
-`<img>` should be scaled, let alone how media queries should be handled
.

I've [filed an issue with the spec][4], hopefully this can be cleared up.

But things get a lot more complicated when you start…

## Drawing SVG to a canvas {#drawing-svg-to-a-canvas}

You can also draw `<img>`s to `<canvas>`es:

    canvas2dContext.drawImage(img, x, y, width, height);
    

**But when should the circle be blue?** There are a few more viewport choices
this time. Should it be:

*   The CSS size of the host window
*   The width/height/viewBox attributes on the `<svg>` 
*   The width/height attributes on the `<img>` 
*   The CSS layout dimensions of the `<img>` 
*   The pixel-data dimensions of the `<canvas>` 
*   The CSS layout dimensions of the `<canvas>` 
*   The width/height specified in `drawImage` 
*   The width/height specified in `drawImage`, multiplied by whatever transform
    the 2d context has
   

Which would you expect? Again, the spec is unclear, and this time every browser
has gone in a different direction. Give it a try:

<legend>SVG size</legend> 
 50x50

 100x100

 Use viewBox

<legend> `<img>` size</legend> 
 Unset

 50x50

 100x100

<legend> `<img>` CSS size</legend> 
 Unset

 50x50

 100x100

 Add to <body>

<legend> `<canvas>` size</legend> 
 50x50

 100x100

<legend> `drawImage` size</legend> 
 50x50

 100x100

<legend>Context transform</legend> 
 0.5x

As far as I can tell, here's what the browsers are doing:

### Chrome {#chrome}

Chrome goes for the width/height attributes specified in the SVG document. This
means if the SVG document says`width="50"`, you'll get the media queries for a
50px wide viewport. If you wanted to draw it using the media queries for a 100px
wide viewport, tough luck. No matter what size you draw it to the canvas, it'll 
draw using the media queries for a 50px width.

However, if the SVG specifies a `viewBox` rather than a fixed width, Chrome
uses the pixel-data width of the`<canvas>` as the viewport width. You
could argue this is similar to how things work with inline SVG, where the 
viewport is the whole window, but switching behaviours based on`viewBox` is
really odd.

Chrome wins the bizarro-pants award for "wonkiest behaviour".

### Safari {#safari}

Like Chrome, Safari uses the size specified in the SVG document, with the same
downsides. But if the SVG uses a`viewBox` rather than a fixed width, it
calculates the width from the`viewBox`, so an SVG with 
`viewBox="50 50 200 200"` would have a width of 150.

So, less bizarre than Chrome, but still really restrictive.

### Firefox {#firefox}

Firefox uses the width/height specified in the `drawImage` call, multiplied by
any context transforms. This means if you draw your SVG so it's 300 canvas 
pixels wide, it'll have a viewport width of 300px.

This kinda reflects their weird `<img>` behaviour - it's based on pixels
drawn. This means you'll get the same density inconsistencies if you multiply 
your canvas width and height by`devicePixelRatio` (and scale back down with CSS
), which you should do to avoid blurriness on high-density screens:<figure
class="full-figure
">

![][5]<canvas width="150" height="60" class="text-canvas"></canvas><canvas
width="150" height="60" class="text-canvas-sharp
"></canvas> <figcaption><img>, <canvas>, multiplied <canvas>. Without
multiplication, the canvas will appear blurry on high-density screens</
figcaption
></figure>

There's logic to what Firefox is doing, but it means your media queries are
tied to pixels drawn.

### Microsoft Edge {#microsoft-edge}

Edge uses the layout size of the `<img>` to determine the viewport. If
the`<img>` doesn't have layout (`display:none` or not in the document
tree) then it falls back to the width/height attributes, if it doesn't have 
those it falls to the intrinsic dimensions of the`<img>`.

This means you can draw the SVG at 1000x1000, but if the image is 
`<img width="100">`, it'll have a viewport width of 100px.

In my opinion this is ideal. It means you can activate media queries for widths
independent of the drawn width. It also feels consistent with responsive images.
When you draw an`<img srcset="…" sizes="…">` to a canvas, all browsers
agree that the drawn image should be the resource currently selected by the
`<img>`.

## Phew! {#phew}

I've [filed an issue with the spec to adopt Edge's behaviour][6], and 
[proposed an addition to `createImageBitmap` so the viewport can be specified in script][7]

For completeness, [here's how I gathered the data][8], and 
[here are the full results][9].

 [1]: https://drafts.csswg.org/mediaqueries-3/#width
 [2]: img/fixed100.4b1cb7cb9384.svg
 [3]: https://svgwg.org/svg2-draft/styling.html#StyleSheetsInHTMLDocuments
 [4]: https://github.com/w3c/svgwg/issues/289
 [5]: img/text.941f43fc7ea8.svg
 [6]: https://github.com/whatwg/html/issues/1880
 [7]: https://github.com/whatwg/html/issues/1881
 [8]: http://jsbin.com/gefaju/2/edit?js,output

 [9]: https://docs.google.com/spreadsheets/d/15IkG42KrEWgv_FbrgfGBSM_PYRi22Vj_uGrcp4LxyMU/edit#gid=0