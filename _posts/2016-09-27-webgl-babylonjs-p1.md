---
layout: post
title:  "Знакомимся с WebGL и BabylonJS (часть 1)"
date:   2016-09-27 17:30:00 +0300
permalink: /webgl-babylonjs-p1/
tags: [webgl, babylonjs, js]
keywords: "webgl, babylonjs, js"
author: zivaaa
---

WebGL привлекает к&nbsp;себе все больше внимания. Веб-разработчики пробуют себя в&nbsp;мире 3D&nbsp;графики, разработчики игр хотят освоить новую платформу.
Основные браузеры уже поддерживают WebGL на&nbsp;хорошем уровне и&nbsp;в&nbsp;вебе начинают появляться серьезные 3D&nbsp;проекты.
В&nbsp;этом уроке мы&nbsp;рассмотрим основные концепции 3D&nbsp;графики на&nbsp;примере создания космической сцены.
Узнаем что такое шейдеры, как моделировать объекты, накладывать текстуры, создавать окружение и&nbsp;ставить свет.
В&nbsp;конце анимируем статичную сцену и&nbsp;добавим эффекты пост-обработки.

<img style="width:50%;" src="/assets/posts/webgl-babylonjs-p1/result.png"/>


Рис.&nbsp;1. Результат, который мы&nbsp;получим в&nbsp;конце урока.

Работать будем с&nbsp;фреймворком BabylonJS, который облегчит нашу задачу и&nbsp;поможет быстрее добиться результата.

<!--more-->

<br/>

## WebGL

Разберем алгоритм работы с WebGL.

![Image](/assets/posts/webgl-babylonjs-p1/schema1.png)

Рис.&nbsp;2. Основные части WebGl приложения.

Чтобы создать любую WebGL сцену, необходимо проделать следующие операции:

* Создаем на&nbsp;странице элемент Canvas
* Получаем контекст WebGL из&nbsp;Canvas для отрисовки графики
* Инициализируем и&nbsp;компилируем шейдеры
* Создаем необходимые буферы&nbsp;/ текстуры&nbsp;/ переменные&nbsp;/ матрицы
* Делаем необходимые расчеты
* Рисуем результат

Если нам нужна анимация, то&nbsp;последние 2&nbsp;пункта будут повторяться снова и&nbsp;снова.
Дальше мы&nbsp;реализуем алгоритм на&nbsp;практике, но&nbsp;сначала разберемся с&nbsp;тем, что такое шейдеры.

<br/>

## Шейдеры (Shaders)

Шейдеры это программы, написанные на&nbsp;C-подобном языке OpenGL ES&nbsp;Shading Language или GLSL.
Шейдеры выполняют роли преобразователя &laquo;сырых&raquo; координат и&nbsp;матриц в&nbsp;финальные координаты и&nbsp;окрашивают их&nbsp;в&nbsp;нужный цвет.

![image](/assets/posts/webgl-babylonjs-p1/shaders.png)

Рис.&nbsp;3. Слева набор вершин и&nbsp;камера до&nbsp;применения шейдеров. Справа&nbsp;&mdash; после.

Каждый материал описывается своими парами шейдеров: *Вершинный (Vertex Shader)* и&nbsp;*Фрагментный (Fragment Shader)*.

*Вершинный шейдер*

```

attribute vec3 aPosition;
void main(void) {
  gl_Position = vec4(aPosition, 1.0);
}

```

Перед нами пример вершинного шейдера. Он&nbsp;состоит из&nbsp;объявленных переменных до&nbsp;функции и&nbsp;самой функции main.
Взаимодействие между&nbsp;JS кодом и&nbsp;шейдерными программами происходит по&nbsp;следующему алгоритму.

* Объявляется переменная в&nbsp;JS коде;
* Создается ссылка на&nbsp;требуемый атрибут в&nbsp;шейдерной программе;
* Объявленное через&nbsp;JS код значение передается в&nbsp;шейдерную программу через ссылку;

Первые два пункта, можно поменять местами, без ущерба для работы программы.

Переменные задаются посредством&nbsp;JS перед запуском отрисовки, а&nbsp;сама main() функция устанавливает преобразованные координаты в&nbsp;глобальную переменную gl_Position типа vec4 (массив 4&nbsp;значений).

```vec4(aPosition, 1.0);``` &mdash; такую запись следует трактовать так: преобразовать значение aPosition типа vec3 (массив 3&nbsp;значений) и&nbsp;значения 1.0&nbsp;к vec4(массиву 4&nbsp;значений).
Так мы&nbsp;получаем искомое значение gl_Position, где первые 3&nbsp;значения массива это координаты xyz, а&nbsp;о&nbsp;четвертом значении мы&nbsp;узнаем немного позже, когда будем писать свой шейдер.

Чаще всего в&nbsp;шейдер передают матрицы преобразования (Model, View, Projection) и&nbsp;потом умножают матрицы на&nbsp;координаты точки, например:

```
vec4 outPosition = worldViewProjection * vec4(aPosition, 1.0);
gl_Position = outPosition;

```
В&nbsp;примере worldViewProjection это матрица модели-вида-проекции, необходимая для корректного отображения точки в&nbsp;пространстве.
Так мы&nbsp;получаем конечные координаты вершины. В&nbsp;этом и&nbsp;заключается задача вершинного шейдера.

Матрицы преобразования перемещают точки в&nbsp;нужную позицию. При этом каждая матрица выполняет свою роль:

* Матрица модели&nbsp;&mdash; это матрица, которая преобразует локальные координаты (относительно центра модели) в&nbsp;глобальные координаты. Образуется путем перемножения матрицы переноса, поворота и&nbsp;масштабирования. Которые в&nbsp;свою очередь нужны для смещения точки по&nbsp;осям, поворота вокруг осей и&nbsp;масштабирования.
* Матрица вида&nbsp;&mdash; это матрица, которая переносит точку из&nbsp;глобальной системы координат в&nbsp;систему координат камеры. Мы&nbsp;переносим не&nbsp;камеру, а&nbsp;мир относительно камеры.
* Матрица проекции&nbsp;&mdash; это матрица, которая дает нам необходимые плоскости отсечения (мин. и&nbsp;макс. координаты отображения), а&nbsp;также создает перспективу для реальности отображения.

Если их&nbsp;перемножить мы&nbsp;получим матрицу модели-вида-проекции (тип mat4), умножение которой на&nbsp;позицию точки (тип vec4) дает нам конечные координаты с&nbsp;учетом камеры, проекции и&nbsp;положения на&nbsp;сцене, то&nbsp;есть те&nbsp;самые конечные координаты gl_Position.

*Фрагментный шейдер*

```
void main(void) {
  gl_FragColor = vec4(0.0, 1.0, 0.0, 1.0);
}
```

Если задача вершинного шейдера получить координату и&nbsp;вернуть ее&nbsp;преобразованное значение,
то&nbsp;задача фрагментного шейдера&nbsp;&mdash; закрашивать в&nbsp;нужный цвет.
В&nbsp;данном примере ```vec4(0.0, 1.0, 0.0, 1.0)``` эквивалент ```rgba(0,255,0,1)``` (зеленый цвет).
Цвет можно также получить из&nbsp;текстуры посредством передачи&nbsp;UV координат (о&nbsp;них позже).

Фрагментный шейдер состоит из&nbsp;функции и&nbsp;некоторых принимаемых параметров.
Он&nbsp;устанавливает глобальной переменной gl_FragColor цветовое значение, параметры вида: ``` vec4(red, green, blue, alpha) ```.


Оба шейдера выполняются для всех вершин и&nbsp;на&nbsp;выходе мы&nbsp;имеем закрашенный участок, соответствующий вершинам и&nbsp;их&nbsp;цветам.
Вот еще несколько интересных моментов о&nbsp;GLSL:

-	GLSL типизированный язык и&nbsp;многие параметры требуют именно переменных типа float, поэтому vec2(0,1) часто будет ошибкой, нужно писать так vec2(0.0, 1.0);
-	GLSL работает с&nbsp;особыми типами, такими как mat4/3/2/1, vec4/3/2/1, упрощающими работу с&nbsp;массивами и&nbsp;матрицами;
-	GLSL имеет множество встроенных функций, например dot(vec3, vec3)&nbsp;&mdash; скалярное произведение векторов;
-	GLSL имеет несколько типов квалификаторов (для указания точности расчетов, а&nbsp;значит и&nbsp;влияния на&nbsp;быстродействие)
-	GLSL имеет несколько спец. типов переменных
    +	 Attribute&nbsp;&mdash; глобальный атрибут для передачи координат в&nbsp;вершинный шейдер
    +	 Uniform&nbsp;&mdash; переменные которые можно использовать в&nbsp;обоих шейдерах
    +	 Varying&nbsp;&mdash; переменные через которые вершинный шейдер может общаться с&nbsp;фрагментным

В&nbsp;последующих частях мы&nbsp;напишем пару шейдеров и&nbsp;освоим некоторые моменты работы с&nbsp;ними.

Во время нашей работы мы&nbsp;столкнемся с&nbsp;необходимостью производить вычисления с&nbsp;матрицами, подготавливать буферы вершин, нормалей, индексов и&nbsp;не&nbsp;только.
Упросить жизнь нам поможет фреймворк под названием BabylonJS.

<br/>

## BabylonJS

[BabylonJS](http://www.babylonjs.com/)&nbsp;&mdash; фреймворк для работы с&nbsp;3D&nbsp;графикой.
С&nbsp;его помощью можно строить примитивы, накладывать текстуры, импортировать шейдеры или модели, подключать физические движки, добавлять тени, анимировать объекты.

Одна из&nbsp;возможностей BabylonJS&nbsp;&mdash; экспорт моделей из&nbsp;3D-редакторов, например _Blender_.
Благодаря этому мы&nbsp;можем создать 3D&nbsp;модель не&nbsp;кодом, а&nbsp;в&nbsp;предназначанной для этого программе.

Попробовать BabylonJS можно [в&nbsp;песочнице](http://babylonjs-playground.azurewebsites.net/?2).

<br/>

## Делаем космос. Окружение и материалы

Начнем работу с&nbsp;создания страницы и&nbsp;подключения всех необходимых библиотек.
Создаем папку *web*, в&nbsp;ней папку *js*&nbsp;и&nbsp;файл index.html. В&nbsp;нем мы&nbsp;опишем верстку страницы и&nbsp;код взаимодействия с&nbsp;BabylonJS. Что получится в&nbsp;итоге можно посмотреть [здесь](https://github.com/zivaaa/bjs-tutorial/blob/master/web/p1.html).

Скачиваем из&nbsp;репозитория библиотеку [babylon.js](https://github.com/BabylonJS/Babylon.js/tree/master/dist).
Для управления камерой нам пригодится библиотека [hand.js](https://handjs.codeplex.com/). Обе библиотеки кладем в&nbsp;папку&nbsp;*js*.
Получим такую структуру проекта:

```
-web/
--js/
---babylon.js
---hand.minified-1.2.js
--index.html

```
Далее пишем код страницы

```
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title></title>
    <script src="js/babylon.js"></script>
    <script src="js/hand.minified-1.2.js"></script>
    <style>
        html, body {
            width: 100%;
            height: 100%;
            padding: 0;
            margin: 0;
            overflow: hidden;
        }

        #canvas {
            width: 100%;
            height: 100%;
        }
    </style>
</head>
<body>
<canvas id="canvas"></canvas>
<script>

</script>
</body>

```

Мы добавили элемент canvas, в котором BabylonJS отобразит нашу сцену.
Нам не нужно задавать элементу canvas аттрибуты width и height, т.к. BabylonJS сам задаст их во время инициализации движка, исходя из стилевых размеров canvas.

Весь джаваскрипт код запишем внутри тега ```<script>```.

Подключаем BabylonJS:

```
//здесь будут основные конфиги
var config = {

};

//проверяем, поддерживается ли работа фреймворка
if (BABYLON.Engine.isSupported()) {
    var canvas = document.getElementById("canvas"); //находим канвас, в котором будем рисовать сцену
    var engine = new BABYLON.Engine(canvas, true); //создаем движок
    var scene = new BABYLON.Scene(engine); //создаем сцену


    engine.runRenderLoop(function() { //инициируем перерисовку
        scene.render(); //перерисовываем сцену (60 раз в секунду)
    });
}
```

Если сейчас открыть страницу в&nbsp;браузере, мы&nbsp;увидим синию страницу (цвет по-умолчанию) с&nbsp;ошибкой в&nbsp;консоли ```Uncaught Error: No camera defined```.
Камера в&nbsp;3D&nbsp;определяется матрицей вида и&nbsp;реализует либо ортогональную, либо перспективную проекцию (которая ближе к&nbsp;реальности).

_(в мире 3D&nbsp;не камера перемещается по&nbsp;миру, а&nbsp;мир перемещается относительно камеры)_

В BabylonJS есть несколько видов камер, например:
 - FreeCamera (камера, которую можно свободно перемещать по сцене)
 - ArcRotateCamera (камера, которая смотрит на точку и вращающается вокруг, находясь всегда на определенном радиусе)
 - FollowCamera (камера, которая находится всегда позади объекта)

Нам понадобится ```ArcRotateCamera```, для перемещения вокруг нашей будущей планеты.

Мы&nbsp;задаем ей&nbsp;имя, повороты alpha и&nbsp;beta, радиус вращения и&nbsp;позицию.
В&nbsp;движке BabylonJS многие параметры будут задаваться с&nbsp;помощью объекта ```Vector(4/3/2/1)```. Например позиция камеры как раз может задаваться как ```new BABYLON.Vector3(0.0, 0.0, 0.0);```.
```
//создаем камеру, которая вращается вокруг заданной цели (это может быть меш или точка)
var camera = new BABYLON.ArcRotateCamera("Camera", -Math.PI / 2, 3*Math.PI / 7, 110, new BABYLON.Vector3(55, 5, 55), scene);

scene.activeCamera = camera; //задаем сцене активную камеру, т.е. ту через которую мы видим сцену

camera.attachControl(canvas, true); //добавляем возможность управления камерой

```

Пока что у нас пустая сцена.

![image](/assets/posts/webgl-babylonjs-p1/p1-scene0.png)

Рис. 4. Базовая сцена.

Начнем наполнять сцену объектами с&nbsp;элемента *skybox*. Skybox&nbsp;&mdash; это огромный куб обтянутый соответствующей текстурой.
Все объекты должны располагаться внутри него, а&nbsp;он&nbsp;должен имитировать окружение (в&nbsp;нашем случае&nbsp;&mdash; космос).
Создадим куб:

```
//создаем скайбокс
var skybox = BABYLON.Mesh.CreateBox("universe", 10000.0, scene);
```

Мы создали наш первый меш (модель куба размером 10000), но он не видим. Камера находится внутри куба, а видимость изнутри поумолчанию отключена.
Надо визуализировать куб.

В кубе 6 граней и на каждую нужна подходящая текстура.

![image](/assets/posts/webgl-babylonjs-p1/p1-skybox.png)

Рис. 5. Текстуры сторон скайбокса.

Скачаем текстуры&nbsp;из [папки universe в&nbsp;репозитории](https://github.com/zivaaa/bjs-tutorial/tree/master/web/textures/universe) и&nbsp;добавим в&nbsp;папку web.
Каждая текстура принадлежит определенной грани. Чтобы применить их, надо объявить материал для куба.
Материал&nbsp;&mdash; это объект который, отвечает за&nbsp;оформление меша, которому он&nbsp;задан. Например, возьмем кирпич.
Он&nbsp;шероховат, имеет оранжевый цвет, на&nbsp;солнце не&nbsp;имеет бликов и не излучает свет.
Весь комплект этих характеристик можно назвать материалом.

Тоже самое с&nbsp;материалом в&nbsp;BabylonJS. Материалу можно задать текстуру, силу отражения света, цвет отражения, видимость, прозрачность и&nbsp;т.д..

А&nbsp;теперь, когда с&nbsp;материалом более-менее понятно объявим его и&nbsp;назначим нашему скайбоксу.

В&nbsp;BabylonJS есть несколько видов материалов, но&nbsp;самый общий&nbsp;&mdash; StandartMaterial, он&nbsp;нам прекрасно подойдет.

```
BABYLON.StandardMaterial("universe", scene);
```

Еще бывают шейдерые материалы, но о них поговорим позже.
Создадим материал для скайбокса, зададим ему текстуру и цвет.

```
var skyboxMaterial = new BABYLON.StandardMaterial("universe", scene); //создаем материал
skyboxMaterial.backFaceCulling = false; //Включаем видимость меша изнутри
skyboxMaterial.reflectionTexture = new BABYLON.CubeTexture("textures/universe/universe", scene); //задаем текстуру скайбокса как текстуру отражения
skyboxMaterial.reflectionTexture.coordinatesMode = BABYLON.Texture.SKYBOX_MODE; //настраиваем скайбокс текстуру так, чтобы грани были повернуты правильно друг к другу
skyboxMaterial.disableLighting = true; //отключаем влияние света
skybox.material = skyboxMaterial; //задаем матерал мешу
```

Свет, который мы&nbsp;добавим потом, не&nbsp;должен отражаться в&nbsp;нашем скайбоксе.
Поэтому мы&nbsp;создаем материал, в&nbsp;котором выключаем влияние света.

Итак, мы&nbsp;создали материал и&nbsp;назначили его нашему скайбоксу. Теперь мы&nbsp;можем увидеть сцену с&nbsp;космическим окружением.

![image](/assets/posts/webgl-babylonjs-p1/p1-skybox-show.png)

Рис.&nbsp;6. Сцена с&nbsp;готовым скайбоксом.

_Зажмите ЛКМ или понажимайте на&nbsp;стрелочки, чтобы камера вращалась_


У&nbsp;стандартного материала есть еще много параметров, для кастомизации отображения:

```
skyboxMaterial.diffuseColor = new BABYLON.Color3(0.0, 0.5, 0.4); //цвет который получится при освещении

/*или*/

skyboxMaterial.diffuseTexture = new BABYLON.Texture("texture.png", scene); //текстура, которая видна при освещении
```

Мы можем задать как цвет, так и текстуру.

```
skyboxMaterial.emissiveColor = new BABYLON.Color3(0, 0.2, 0.2); //собственный цвет, которым "светится" меш

/*или*/

skyboxMaterial.emissiveTexture = new BABYLON.Texture("texture.png", scene); //собственная текстура, которой "светится" меш
```

```
skyboxMaterial.specularColor = new BABYLON.Color3(0, 0.2, 0.2); //цвет блика

/*или*/

skyboxMaterial.specularTexture = new BABYLON.Texture("texture.png", scene); //текстура блика
```

Другие возможности можно изучить на сайте [генератора материалов BabylonJS](http://materialeditor.raananweber.com/).

<br/>

## Создаем базовое окружение

Теперь приступим к&nbsp;наполнению сцены. Мы&nbsp;создадим 2&nbsp;меша, символизирующие планеты (Луну и&nbsp;Землю) и&nbsp;установим освещение.

Для начала обновим конфиг

```
//здесь будут основные конфиги
var config = {
    PLANET_RADIUS: 50, //радиус земли
    PLANET_V: 300, // количество вершин
    MOON_RADIUS: 25, //радиус луны
};
```

Теперь создадим меши (сферы). Делать это будем с помощью метода ```BABYLON.Mesh.CreateSphere```, похожий на ```BABYLON.Mesh.CreateBox```, который мы использовали ранее.

```
 //Земля
 var planet = BABYLON.Mesh.CreateSphere("planet", config.PLANET_V, config.PLANET_RADIUS, scene, true);
 planet.position = new BABYLON.Vector3(-250.0, -10,0, -250.0); // задаем позицию на сцене

 var moon = BABYLON.Mesh.CreateSphere("moon", 25, config.MOON_RADIUS, scene); //Луна
 moon.parent = planet; //задаем родителя &mdash; Землю
 moon.position = new BABYLON.Vector3(-102.0, 0,0, 0.0); //задаем позицию луны

 camera.target = planet; //Задаем точку вращения камеры
```

Мы&nbsp;установили Землю как родителя Луны, поэтому Луна отображается в&nbsp;системе координат Земли.
Также мы&nbsp;установили _target_ камере, чтобы она вращалась относительно Земли.

Вот что получилось на&nbsp;данном этапе:

![image](/assets/posts/webgl-babylonjs-p1/p2-meshes.png)

Рис.&nbsp;7. Два сферических меша.

Меши черные, потому что на&nbsp;сцене нет источника света.
При этом, отсутствие источника света ни&nbsp;коем образом не&nbsp;затрагивает скайбокс, ведь свет на&nbsp;него никак не&nbsp;влияет.

Добавим источник света.

```
//создаем точечный источник света в точке 0,0,0
var lightSourceMesh = new BABYLON.PointLight("Omni", new BABYLON.Vector3(0.0, 0,0, 0.0), scene);
/*цвет света*/
lightSourceMesh.diffuse = new BABYLON.Color3(0.5, 0.5, 0.5);
```

В&nbsp;BabylonJS есть несколько видов источников света, но&nbsp;нам нужен точечный, имитирующий Солнце.
При добавлении источника света все меши со&nbsp;стандартным материалом и&nbsp;которые способны воспринимать свет, будут автоматически отрендерены с&nbsp;расчетом влияния света.

```
 //создаем точечный источник света в точке 0,0,0
 var lightSourceMesh = new BABYLON.PointLight("Omni", new BABYLON.Vector3(0.0, 0,0, 0.0), scene);
 /*цвет света*/
 lightSourceMesh.diffuse = new BABYLON.Color3(0.5, 0.5, 0.5);
```

Мы создали источник света в точке _0,0,0_ цветом _0.5,0.5,0.5_ и получили результат:

![image](/assets/posts/webgl-babylonjs-p1/p2-light.png)

Рис. 8. Освещенные меши.

Код этой части проекта [здесь](https://github.com/zivaaa/bjs-tutorial/blob/master/web/p2.html).

<br/>

## Накладываем текстуры

Перейдем к&nbsp;материалам с&nbsp;текстурами.
Начнем с&nbsp;Луны, потому как она будет у&nbsp;нас проще чем Земля.
Создадим *StandartMaterial* и&nbsp;зададим основную текстуру

```

var moonMat = new BABYLON.StandardMaterial("moonMat", scene); //Материал Луны
moonMat.diffuseTexture = new BABYLON.Texture("textures/moon.jpg", scene); //задаем текстуру планеты

moon.material = moonMat; //задаем материал

```

Текстуры можно взять [здесь](https://github.com/zivaaa/bjs-tutorial/blob/master/web/textures/)

![image](/assets/posts/webgl-babylonjs-p1/moon.jpg)

Рис.&nbsp;9. Диффузная текстура.

Результат:

![image](/assets/posts/webgl-babylonjs-p1/p3-moon-1.png)

Продолжим улучшать вид Луны. Добавим bump текстуру. Это текстура которая добавляет диффузной текстуре объем, когда она освещена.
Это никак не&nbsp;скажется на&nbsp;геометрии, но&nbsp;меш будет выглядеть не&nbsp;как гладкий шар, а&nbsp;как будто у&nbsp;него есть неровности.
Эта текстура называется _картой нормалей_.

![image](/assets/posts/webgl-babylonjs-p1/moon_bump.jpg)

Рис.&nbsp;10. Карта нормалей.

```
moonMat.bumpTexture = new BABYLON.Texture("textures/moon_bump.jpg", scene);
```

![image](/assets/posts/webgl-babylonjs-p1/p3-moon2.png)

Рис. 11. Меш с применением карты нормалей.

Теперь рассмотрим карту освещения (specular map). Это текстура, которая определяет степень влияния света на определенные фрагменты меша.

![image](/assets/posts/webgl-babylonjs-p1/moon_spec.jpg)

Рис. 12. Карта освещения.

Применим ее, чтобы избавится от яркого блика в центре Луны.

```
moonMat.specularTexture = new BABYLON.Texture("textures/moon_spec.jpg", scene);
```

![image](/assets/posts/webgl-babylonjs-p1/p3-moon-3.png)

Рис.&nbsp;13. Результат применения карты освещения.

Для текстурирования Земли мы:

* Применим карту высот
* Создадим шейдерный материал (ShaderMaterial)
* Напишем кастомные шейдеры

_Карта высот_ &mdash; это черно-белая тектура, где светлые фрагменты определяют высоту на&nbsp;которую будут смещаться вершины. В&nbsp;отличие от&nbsp;карты нормалей, это затронет геометрию меша.

![image](/assets/posts/webgl-babylonjs-p1/earth-height.png)

Рис.&nbsp;14. Карта высот.

Карта намеренно перевернута по&nbsp;горизонтали, чтобы вершины смещались соответственно текстуре Земли, которую мы&nbsp;применим позже. Применим карту высот, а&nbsp;также повернем меш на&nbsp;180 по&nbsp;оси&nbsp;z:

```
planet.rotation.z = Math.PI;
planet.applyDisplacementMap("/textures/earth-height.png", 0, 1); //применяем карту высот - смещение => от 0 для черных фрагментов до 1 для белых
```

Параметр конфига _PLANET_V_ можно и&nbsp;увеличить, тем самым увеличив детализацию планеты, но&nbsp;это повлияет на&nbsp;производительность сцены.

![image](/assets/posts/webgl-babylonjs-p1/p3-earth-1.png)

Рис.&nbsp;15. Результат применения карты высот.

_(Это не&nbsp;относится к&nbsp;материалу, а влияет на&nbsp;геометрию, поэтому материал еще не&nbsp;нужен)_

Пора переходить к&nbsp;текстурам. Стандартный материал нам уже не&nbsp;подойдет, надо переходить на&nbsp;шейдерный. Будем не&nbsp;просто накладывать текстуру, а&nbsp;отображать ее&nbsp;в&nbsp;зависимости от&nbsp;освещения.

Для реализации нам понадобятся:

1. ShaderMaterial
2. Дневная и&nbsp;ночная текстура Земли
3. Шейдеры

Объявим материал

```
var planetMat = new BABYLON.ShaderMaterial("planetMat", scene, {
            vertexElement: "vertexPlanet",
            fragmentElement: "fragmentPlanet",
        },
        {
            attributes: ["position", "normal", "uv"],
            uniforms: ["world", "worldView", "worldViewProjection", "diffuseTexture", "nightTexture"],
        });
```

Мы&nbsp;задали параметры:

*	Имя
*	Сцена
*	Набор шейдеров _(тип шейдера : id&nbsp;шейдера)_
*	Передаваемые в&nbsp;шейдер атрибуты

Описание шейдера можно положить в&nbsp;файл, а&nbsp;можно написать прямо в&nbsp;разметке HTML.
Чтобы описать шейдер в разметке, нужно создать элемент script с&nbsp;типом ```application/vertexShader``` или ```application/fragmentShader```.
```
<script type="application/vertexShader" id="vertexPlanet">
    precision highp float;

    // Attributes
    attribute vec3 position;
    attribute vec3 normal;
    attribute vec2 uv;

    // Uniforms
    uniform mat4 world;
    uniform mat4 worldViewProjection;

    // Varying
    varying vec2 vUV;
    varying vec3 vPositionW;
    varying vec3 vNormalW;

    void main(void) {
        vec4 outPosition = worldViewProjection * vec4(position, 1.0);
        gl_Position = outPosition;

        vPositionW = vec3(world * vec4(position, 1.0));
        vNormalW = normalize(vec3(world * vec4(normal, 0.0)));

        vUV = uv;
    }
</script>
```

Разберем построчно этот вершинный шейдера.

```
//...
precision highp float;
//...
```
В&nbsp;GLSL квалификатор точности precision определяет количество байт выделяемых под&nbsp;переменную. В&nbsp;данном случае float.
Это оказывает влияние не&nbsp;только на&nbsp;точность расчетов, но&nbsp;и&nbsp;на&nbsp;производительность.
Мы&nbsp;задаем повышенную точность переменных типа float для более корректного отображения нашей планеты.
В&nbsp;BabylonJS лучше всегда использовать квалификатор highp.
```
//...
// Attributes
attribute vec3 position;
attribute vec3 normal;
attribute vec2 uv;
//...
```

Это переменные типа ```attribute```, которые принимает только вершинный шейдер. То, что они&nbsp;должны быть переданы в&nbsp;шейдер мы&nbsp;указали ранее в&nbsp;материале:

```... attributes: [position, normal, uv], ...```

```position``` &mdash; переменная атрибут типа ```vec3``` (т.е. массив из&nbsp;3&nbsp;элементов типа float). Это координаты текущей вершины.

```normal``` &mdash; атрибут типа ```vec3```, его смысл&nbsp;&mdash; вектор нормали, это вектор перпендикулярный грани (см. рисунок ниже). Он&nbsp;нужен нам для определения ориентации поверхности, и&nbsp;задается для каждой вершины.

Определение ориентации нужно, чтобы понять&nbsp;&mdash; какая сторона плоскости является внешней, а&nbsp;какая внутренней.

![image](/assets/posts/webgl-babylonjs-p1/p3-normal.jpg)

Рис. 16. Нормаль к 4 плоскости (она также является нормалью ко всем четырем вершинам).

```Uv``` – атрибут типа ```vec2```, представляет собой координату текстуры, которая соответствует вершине (см. рис ниже).

![image](/assets/posts/webgl-babylonjs-p1/p3-uv.png)

Рис.&nbsp;17. UV&nbsp;координаты. Они отсчитываются слева-направо и&nbsp;снизу-вверх от&nbsp;0.0 до&nbsp;1.0.

_UV_ координаты задаются для каждой вершины. Не&nbsp;каждая вершина соответствует всем возможным координатам. Например при задании двум вершинам _uv_ координат ([0.1, 0,1] и&nbsp;[0.5, 0.5]) область между ними будет закрашена как и&nbsp;ожидается.
Значения цвета каждого фрагмента между вершинами будут интерполироваться относительно конечных координат (наглядный пример интерполяции&nbsp;&mdash; градиент).
Но&nbsp;это работа фрагментного шейдера, а&nbsp;не&nbsp;вершинного. В&nbsp;данном случае мы&nbsp;будем передавать _UV_ координату (координату текселя) фрагментному шейдеру посредством *varying* переменной.
```
//...
// Uniforms
uniform mat4 world;
uniform mat4 worldViewProjection;
//...
```

```Uniform``` переменные&nbsp;&mdash; это переменные, которые могут быть переданы в&nbsp;оба шейдера.

```World``` &mdash; переменная типа ```mat4``` (матрица 4&nbsp;на 4, наполненная значениями типа ```float```), при умножении вектора координат на&nbsp;которую мы&nbsp;получаем вершину в&nbsp;глобальной системе координат (относительно центра сцены, а&nbsp;не&nbsp;относительно центра модели)

```worldViewProjection``` &mdash; матрица 4&nbsp;на 4, матрица мира-вида-проекции. После умножения на&nbsp;нее вершина перемещается в&nbsp;нужное положение относительно камеры, т.е свое конечное положение.
```
//...
 // Varying
varying vec2 vUV;
varying vec3 vPositionW;
varying vec3 vNormalW;
//...
```

```Varying``` переменные передают значения из&nbsp;одного шейдера в&nbsp;другой при условии, что они названы одинаково.
Передавать мы&nbsp;будем координату текселя, координату в&nbsp;мировой системе координат и&nbsp;ее&nbsp;нормаль единичной длины.

```vUV``` &mdash; это uv&nbsp;координаты.

```vPositionW``` &mdash; это позиция вершины в&nbsp;глобальной системе координат

```vNormalW``` &mdash; это нормализованная нормаль вершины

Теперь к расчетам:

```
//...
void main(void) {
    vec4 outPosition = worldViewProjection * vec4(position, 1.0);
    gl_Position = outPosition;

    vPositionW = vec3(world * vec4(position, 1.0));
    vNormalW = normalize(vec3(world * vec4(normal, 0.0)));

    vUV = uv;
}
//...
```

Сначала мы&nbsp;описываем, то&nbsp;что должен делать вершинный шейдер&nbsp;&mdash; определить конечную позицию каждой вершины. Для этого умножаем матрицу вида проекции (предоставляемую и&nbsp;уже рассчитанную BabylonJS) на&nbsp;позицию вершины.

```worldViewProjection * vec4(position, 1.0)```

Умножаем переменную mat4&nbsp;на переменную vec4, которую формируем из&nbsp;атрибута *position* _(x,y,z&nbsp;&mdash; 3&nbsp;значения)_ и&nbsp;значения 1.0, которое станет 4-ым параметром.
В&nbsp;данном случае vec4&nbsp;&mdash; это позиция. Если&nbsp;же задать 0.0, то&nbsp;значением vec4 станет направлением.

Затем делаем необходимые для нашего кастомного отображения действия:

```
//...
vPositionW = vec3(world * vec4(position, 1.0));
vNormalW = normalize(vec3(world * vec4(normal, 0.0)));

vUV = uv;
//...
```

Передаем во&nbsp;фрагментный шейдер глобальную координату вершины, ее&nbsp;нормализованную нормаль и&nbsp;координату текселя (это ```varying``` переменные). Зачем? Выясним, когда будем разбирать и&nbsp;улучшать фрагментный шейдер.
А&nbsp;пока мы&nbsp;уже разобрали объявление *ShaderMaterial* и&nbsp;одного из&nbsp;двух шейдеров.
Нам надо чтобы в&nbsp;зависимости от&nbsp;освещенности использовалась одна или другая текстура, поэтому объявим их&nbsp;в&nbsp;материале

```
var diffuseTexture = new BABYLON.Texture("textures/earth-diffuse.jpg", scene);
var nightTexture = new BABYLON.Texture("textures/earth-night-o2.png", scene);

planetMat.setVector3("vLightPosition", lightSourceMesh.position); //задаем позицию источника света
planetMat.setTexture("diffuseTexture", diffuseTexture); //задаем базовую текстуру ландшафта материалу
planetMat.setTexture("nightTexture", nightTexture);//задаем ночную текстуру материалу
```

![image](/assets/posts/webgl-babylonjs-p1/earth-diffuse.jpg)

Рис. 18. Текстура ландшафта.

![image](/assets/posts/webgl-babylonjs-p1/earth-night-o2.png)

Рис. 19. Текстура ночных городов.

Таким образом мы задали обе текстуры. Чтобы они передавались в шейдере мы уже настроили нужные параметры этой строчкой

```
 uniforms: ["world", "worldView", "worldViewProjection", "diffuseTexture", "nightTexture"]
```

Все необходимые параметры настроены и&nbsp;будут передаваться в&nbsp;шейдеры. Напишем фрагментный шейдер в&nbsp;элементе ```<script>``` с&nbsp;типом &laquo;application/fragmentShader&raquo; и&nbsp;посмотрим как он&nbsp;работает.
Сначала каркас, а&nbsp;расчет позже

```
<script type="application/fragmentShader" id="fragmentPlanet">
    precision highp float;

    // Varying
    varying vec2 vUV;
    varying vec3 vPositionW;
    varying vec3 vNormalW;

    // Refs
    uniform vec3 lightPosition;
    uniform sampler2D diffuseTexture;
    uniform sampler2D nightTexture;


    void main(void) {
        //calculating
    }
</script>
```

Необходимые переменные, для отрисовки каждого фрагмента:

```varying vec2 vUV``` &mdash; uv&nbsp;координаты текстуры

```varying vec3 vPositionW``` &mdash; позиция вершины на&nbsp;сцене

```varying vec3 vNormalW``` &mdash; нормализованная нормаль вершины

_Нормализованный вектор_ &mdash; это вектор, длина которого равна&nbsp;1 (для его получения каждую координату нужно разделить на&nbsp;длину вектора). В&nbsp;GLSL есть специальная функция нормализации ```normalize()```.
Т.е. ```vNormalW``` &mdash; это нормаль.

```uniform vec3 lightPosition``` &mdash; позиция источника света

```uniform sampler2D diffuseTexture``` &mdash; переданная в&nbsp;шейдер текстура ландшафта

```uniform sampler2D nightTexture``` &mdash; переданная в&nbsp;шейдер ночная текстура

С&nbsp;глобальными данными шейдера разобрались, переходим к&nbsp;главной функции ```main``` определяющей конечный цвет фрагментов.
Сначала опишем задачи, которые должен выполнять шейдер:

* Окрашивать меш в&nbsp;нужную текстуру
* Учитывать положение источника освещения и&nbsp;делать ярче ту&nbsp;часть меша, которая непосредственно освещена
* Смешивать текстуры, в&nbsp;зависимости от&nbsp;степени освещенности

Приступим к&nbsp;написанию шейдера:

```
//...
void main(void) {
    vec3 direction = lightPosition - vPositionW;
    vec3 lightVectorW = normalize(direction);

//...
}
```

Первым делом мы&nbsp;определяем вектор от&nbsp;источника света до&nbsp;вершины и&nbsp;нормализуем.
Происходит это просто: вычитаем из&nbsp;координат источника света (_lightPosition_) координаты вершины (_vPositionW_), а&nbsp;потом применяем встроенную функцию нормализации вектора _(normalize)_.
Теперь у&nbsp;нас есть ```lightVectorW```. Его будем использовать для определения коэффициента освещенности фрагмента соответствующего текущей вершине.
Поможет нам в&nbsp;этом *скалярное произведение векторов*, а&nbsp;именно тот факт, что скалярное произведение нормализованных векторов дает нам косинус угла между ними.
А&nbsp;так как нормализованная нормаль вершины у&nbsp;нас уже есть (```vNormalW```, ее&nbsp;нам рассчитал вершинный шейдер), то&nbsp;остается только выполнить скалярное произведение и&nbsp;получить косинус угла.

```
//...
// diffuse
float lightDiffuse = max(0.05, dot(vNormalW, lightVectorW));
//...
```

Теперь выводим коэффициент освещенности ```lightDiffuse``` с&nbsp;помощью следующих операций:

1. Используем функцию ```dot()``` - это встроенная функция расчета скалярного произведения
2. Ее&nbsp;результат кладем во&nbsp;встроенную функцию ```max()```, которая вернет наибольшее из&nbsp;чисел _0.05_ (чтобы неосвещенная часть меша была не&nbsp;совсем черной) и&nbsp;результат выполнения _dot_.
Поэтому результат будет не&nbsp;меньше 0.05, но&nbsp;и&nbsp;не&nbsp;больше 1.0 (т.к. значение cos всегда меньше 1).

Почему именно так (схематичный рисунок):

![image](/assets/posts/webgl-babylonjs-p1/p3-cos.jpg)

Рис.&nbsp;20. Углы между нормалями вершин и&nbsp;вектором направления света на&nbsp;эти вершины.

Чем больше угол между вектором направления света на&nbsp;вершину и&nbsp;нормалью вершины, тем ниже коэффициент освещенности и&nbsp;тем темнее будет фрагмент.

```
//...
vec3 color;
vec4 nightColor = texture2D(nightTexture, vUV).rgba;
vec3 diffuseColor = texture2D(diffuseTexture, vUV).rgb;

```

Мы&nbsp;объявили переменную для конечного цвета ```vec3 color```,
переменную ```vec4 nightColor``` &mdash; в&nbsp;которую мы&nbsp;поместили цвет соответствующий _uv_ координатам текстуры,
переменную ```vec3 diffuseColor``` &mdash; с&nbsp;которой сделали тоже самое, только цвет будет браться из&nbsp;базовой текстуры.

Между переменными цветов, взятых из&nbsp;текстур есть небольшие отличия. Например *.rgb*&nbsp;&mdash; способ обращения к&nbsp;1,2 и&nbsp;3&nbsp;элементу массива _(можно также использовать x/y/z)_ в&nbsp;результате которого получим ```vec3``` с&nbsp;этими значениями.
```texture2D``` возвращает ```vec4```, где 4&nbsp;значение&nbsp;&mdash; это альфа составляющая. Но&nbsp;для _diffuseTexture_ она всегда равно 1.0, а&nbsp;вот для _nightColor_ большая часть точек прозрачна. Точки отвечающие за&nbsp;&laquo;ночные города&raquo; почти полностью непрозрачны, мы&nbsp;это используем далее, когда будем получать результирующий цвет.

```
//...
color = diffuseColor * lightDiffuse + (nightColor.rgb * nightColor.a * pow((1.0 - lightDiffuse), 6.0));
gl_FragColor = vec4(color, 1.0);
```

Умножаем ```diffuseColor``` на ```lightDiffuse``` и&nbsp;получаем цвет обычной текстуры под влиянием светового коэффициента.
Так если точка напрямую освещена, то&nbsp;значения будут примерно такие&nbsp;&mdash; ```vec3(0.0, 0.4, 0.5) * 0.9```. Так мы&nbsp;получим почти тот&nbsp;же цвет, что есть на&nbsp;текстуре, если&nbsp;же _lightDiffuse_ будет ближе к&nbsp;0, то&nbsp;и&nbsp;цвет будет почти черным.

Прибавляем составляющую ночной текстуры ```+ (nightColor.rgb * nightColor.a * pow((1.0, lightDiffuse), 6.0))```.
Умножаем оригинальный цвет точки на&nbsp;альфа-составляющую ```nightColor.rgb * nightColor.a``` (которая будет = 0&nbsp;почти везде кроме городов, а&nbsp;значит в&nbsp;этом случае влияния на&nbsp;результирующий цвет оказано не&nbsp;будет вообще).
А&nbsp;потом домножаем на _обратный_ коэффициент освещенности в&nbsp;степени 6.0 ```pow((1.0, lightDiffuse), 6.0)```.
_Обратный_ потому что в&nbsp;этой части&nbsp;&mdash; чем меньше сам коэффициент, тем больше его влияние (мы&nbsp;ведь вычитаем из&nbsp;максимального значения коэффициента равного 1.0).

Степень&nbsp;6, нужная для экспоненциального роста значения, чтобы светимость городов была заметна, когда значение коэффициента осещения lightDiffuse близко к&nbsp;нулю. Если&nbsp;же степень убрать, то&nbsp;светимость городов будет расти линейно, что приведет к&nbsp;нежелательному проявлению света городов на&nbsp;освещенной части Земли.

Чем меньше коэффициент&nbsp;&mdash; тем сильнее выражена безовая текстура, чем больше&nbsp;&mdash; тем больше выражена ночная.

Наконец ```gl_FragColor = vec4(color, 1.0);``` задает финальный цвет фрагмента&nbsp;&mdash; vec4(vec3, float). Именно так можно составлять нужный тип, GLSL все разберет и&nbsp;создаст нужную переменную _vec4_ из _vec3_ и&nbsp;еще одной переменной.

Финальный код шейдера:

```
precision highp float;

// Varying
varying vec2 vUV;
varying vec3 vPositionW;
varying vec3 vNormalW;

// Refs
uniform vec3 lightPosition;
uniform sampler2D diffuseTexture;
uniform sampler2D nightTexture;


void main(void) {
    vec3 direction = lightPosition - vPositionW;
    vec3 lightVectorW = normalize(direction);

    // diffuse
    float lightDiffuse = max(0.05, dot(vNormalW, lightVectorW));

    vec3 color;
    vec4 nightColor = texture2D(nightTexture, vUV).rgba;
    vec3 diffuseColor = texture2D(diffuseTexture, vUV).rgb;

    color = diffuseColor * lightDiffuse + (nightColor.rgb * nightColor.a * pow((1.0 - lightDiffuse), 6.0));
    gl_FragColor = vec4(color, 1.0);
}

```

Добавим еще одну строчку в ```runRenderLoop``` для того, чтобы заставить планету вращаться и разглядеть результат со всех сторон.

```
planet.rotation.y+= 0.001; //поворот на 0.001 радиана
```

![image](/assets/posts/webgl-babylonjs-p1/p3-result.png)

Рис. 21. Сцена с Землей.

Код примера можно посмотреть [здесь](https://github.com/zivaaa/bjs-tutorial/blob/master/web/p3.html).

 [Продолжение](/webgl-babylonjs-p2)
