---
layout: post
title:  "Знакомимся с WebGL и BabylonJS (часть 2)"
date:   2016-09-24 20:00:00 +0300
permalink: /webgl-babylonjs-p2/
tags: [webgl, babylonjs, js]
keywords: "webgl, babylonjs, js"
author: zivaaa
---

Продолжение урока по созданию 3D сцены средствами браузера.

<!--more-->

## Добавляем атмосферу

Земля готова, но на ней не хватает атмосферы. Сымитируем ее следующими эффектами:

* Вращающиеся облака
* Краевая голубая подсветка
* При взгляде с обратной стороны на источник света должен быть эффект заката (оранжевое свечение)

Для создания этих эффектов нам потребуется:

* Сферический меш размером чуть больше Земли и установленный в той же позиции
* Тестура облаков
* Шейдерный материал
* Пара шейдеров

![image](/assets/posts/webgl-babylonjs-p1/earth-c.jpg)

Рис. 1. Текстура атмосферы

Объявим материал и меш:

```
var cloudsMaterial = new BABYLON.ShaderMaterial("cloudsMaterial", scene, {
            vertexElement: "cloudsVertex",
            fragmentElement: "cloudsFragment",
        },
        {
            attributes: ["position", "normal", "uv"],
            uniforms: ["world", "worldView", "worldViewProjection", "cloudsTexture", "lightPosition", "cameraPosition"],
            needAlphaBlending: true
        });

var cloudsTexture = new BABYLON.Texture("textures/earth-c.jpg", scene);

cloudsMaterial.setTexture("cloudsTexture", cloudsTexture);
cloudsMaterial.setVector3("cameraPosition", BABYLON.Vector3.Zero());
cloudsMaterial.backFaceCulling = false;


var cloudsMesh = BABYLON.Mesh.CreateSphere("clouds", config.PLANET_V, config.PLANET_RADIUS + config.ENV_H, scene, true);
cloudsMesh.material = cloudsMaterial;
cloudsMesh.rotation.z = Math.PI;
cloudsMesh.parent = planet;
```

Кроме положения источника света нам потребуется следить за положением камеры, чтобы реализовать нужные эффекты.
Поэтому в функции ```renderLoop``` добавим комманды обновления этих значений.

```
var shaderMaterial = scene.getMaterialByName("cloudsMaterial");
shaderMaterial.setVector3("cameraPosition", scene.activeCamera.position);
shaderMaterial.setVector3("lightPosition", lightSourceMesh.position);
```

Приступим к созданию шейдеров.
*Вершинный шейдер*

```
<script type="application/vertexShader" id="cloudsVertex">
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

Он идентичен вершинному шейдеру Земли. Вся сложность в описании фрагментного шейдера.

Первым делом создадим каркас шейдера

```
<script type="application/fragmentShader" id="cloudsFragment">
    precision highp float;

    varying vec3 vPositionW;
    varying vec3 vNormalW;

    varying vec2 vUV;

    uniform sampler2D cloudsTexture;
    uniform vec3 cameraPosition;
    uniform vec3 lightPosition;


    void main(void) {
       //...
    }
</script>
```

``` vPositionW ``` - позиция вершины

``` vNormalW ``` - нормаль вершины

``` vUV ``` - uv координаты текстуры

``` cloudsTexture ``` - текстура облаков

``` cameraPosition ``` - позиция камеры в мире

``` lightPosition ``` - позиция источника света

Здесь все должно быть понятно, перейдем к расчетам

```
//...

void main(void) {
    vec3 viewDirectionW = normalize(cameraPosition - vPositionW); //Нормализованный вектор взгляда от камеры до вершины

    // Light
    vec3 direction = lightPosition - vPositionW; //Направление от источника света до вершины
    vec3 lightVectorW = normalize(direction); //Получение нормализованного вектора

    // lighting
    float lightCos = dot(vNormalW, lightVectorW); //Получаем косинус между направлением нормали вершины и направлением "луча света" вершину
    float lightDiffuse = max(0., lightCos); //рассчитываем коэффициент освещенности от 0 до 1

    vec3 color = texture2D(cloudsTexture, vUV).rgb; //получаем RGB составляющую цвета текселя по переданной UV координате из текстуры
    float globalAlpha = clamp(color.r, 0.0, 1.0); //определяем альфа составляющую

    gl_FragColor = vec4(color * lightDiffuse, globalAlpha);
}

```

В текстуре нет альфа составляющей и прозрачная часть меша должна рассчитываться из цветовой.
Поэтому получаем цвет текселя и присваиваем альфа составляющей значение красной составляющей. Мы могли бы взять любую другую, т.к. текстура черно-белая и они все равны.
Чем чернее тем прозрачнее будет текстура в каждом обрабатываемом фрагменте.

В конце умножаем цвет фрагмента на коэффициент освещенности, чтобы сделать облака в теневой части темнее.

![image](/assets/posts/webgl-babylonjs-p2/p4-env1.png)

Рис. 2. Первая версия атмосферы.

Чтобы правильно отобразить центр и края сферы нам нужна функция преломления.

```
float computeFresnelTerm(vec3 viewDirection, vec3 normalW, float bias, float power)
{
    float fresnelTerm = pow(bias + dot(viewDirection, normalW), power);
    return clamp(fresnelTerm, 0., 1.);
}
```

В зависимости от значений ```bias``` и ```power``` мы получим коэффициент по которому определим край атмосферы для подсветки.
Функция ```clamp``` работает так:

 - она принимает такие аргументы clamp([проверяемое значение], [минимальное значение], [максимальное значение]);
 - возвращает [проверяемое значение], если оно в пределах второго и третьего аргумента. Если [проверяемое значение] выход за пределы минимума и максимума, то возвращает минимум или максимум соответственно;

Добавим обработку фрагмента в зависимости от коэффициента ```fresnelTerm```, который показывает степень преломления света (коэффициент Френеля). Нам он покажет когда надо рисовать краевую подсветку.

Вот обновленный шейдер

```
//...

float computeFresnelTerm(vec3 viewDirection, vec3 normalW, float bias, float power)
{
    float fresnelTerm = pow(bias + dot(viewDirection, normalW), power);
    return clamp(fresnelTerm, 0., 1.);
}


void main(void) {
    vec3 viewDirectionW = normalize(cameraPosition - vPositionW); //Нормализованный вектор взгляда от камеры до вершины

    // Light
    vec3 direction = lightPosition - vPositionW; //Направление от источника света до вершины
    vec3 lightVectorW = normalize(direction); //Получение нормализованного вектора

    // lighting
    float lightCos = dot(vNormalW, lightVectorW); //Получаем косинус между направлением нормали вершины и направлением "луча света" вершину
    float lightDiffuse = max(0., lightCos); //рассчитываем коэффициент освещенности от 0 до 1

    vec3 color = texture2D(cloudsTexture, vUV).rgb; //получаем RGB составляющую цвета текселя по переданной UV координате из текстуры
    float globalAlpha = clamp(color.r, 0.0, 1.0); //определяем альфа составляющую

    // Fresnel
    float fresnelTerm = computeFresnelTerm(viewDirectionW, vNormalW, 0.72, 5.0); //меняйте значения чтобы изменить эффект преломления

    float resultAlpha; //результирующая альфа составляющая

    if (fresnelTerm < 0.95) {
        //это краевая, подсвечиваемая зона сферы
        float envDiffuse = clamp(pow(fresnelTerm - 0.92, 1.0/2.0) * 2.0, 0.0, 1.0); //коэффициент рассеивания для смягчения границы перехода между центральной части и частию сияния
        resultAlpha = fresnelTerm * envDiffuse * lightCos; //получаем прозрачность умножая коэффициент Френеля на косинус мешду светом и взглядом на вершину
        color = color / 2.0 + vec3(0.0,0.2,0.4); //уменьшаем базовый текстурный цвет на 4 и прибавляем голубой
    } else {
        //это центр сферы, должен демонстрировать
        resultAlpha = fresnelTerm * globalAlpha * lightDiffuse;
    }

    gl_FragColor = vec4(color * lightDiffuse, resultAlpha);
}

```

Если ```fresnelTerm``` < 0.95 - значит что пора рисовать подсветку. При больших значениях рисуем как обычно.
В строке ```resultAlpha = fresnelTerm * lightCos``` используется _lightCos_, а не обработанный коэффициент освещения, это пригодится дальше, при расчете взгляда с темной стороны планеты.
Мы также учли коэффициент освещения и облака будут постепенно пропадать в тени.

В результате у нас получится такая сцена:

![image](/assets/posts/webgl-babylonjs-p2/p4-env2.png)

Рис. 2. Краевая подсветка атмосферы

Улучшим атмосферу эффектом заката. Для этого нужна переменная, по которой можно судить о том, что "вектор света" и "вектор взгляда" направлены друг на друга, а также определить нужную зону эффекта.

```
//...
//эффект заката
float backLightCos = dot(viewDirectionW, lightVectorW); //косинус угла между вектором взгляда на вершину и вектором луча света
float cosConst = 0.9; //граница расссчета эффекта заката. 0.9 => угол в ~(155 - 205) градусов
//...
```

``` backLightCos ``` - этот коэффициент будет основой расчета новой подсветки, по факту это косинус угла между вектором взгляда на вершину и вектором луча света на вершину.
``` cosConst ``` - граница косинуса для эффекта заката

Реализуем часть шейдера для ситуации подсветки заката (см. комментарии):

```
//...
//эффект заката
float backLightCos = dot(viewDirectionW, lightVectorW); //косинус между вектором взгляда на вершину и вектором луча света
float cosConst = 0.9; // граница расчета эффекта заката. 0.9 => угол в ~(155 - 205) градусов

//если угол между вектором взгляда и лучом света  ~(155 - 205) градусов
if (backLightCos < -cosConst) {
   //Обработка свечения с обратной стороны
   float sunHighlight = pow(backLightCos+cosConst, 2.0); //коэффициент подсветки
   if (fresnelTerm < 0.9) {
       //если это край атмосферы (подсвечиваемая часть) то для нее такой расчет
       sunHighlight *= 65.0; //увеличиваем коэффициент подсветки заката
       resultAlpha = sunHighlight; //устанавливаем его как прозрачность
       color *= lightDiffuse; //умножаем основной цвет на коэффициент освещенности
       color.r += sunHighlight; //увеличиваем красную составляющую на коэффициант подсветки заката
       color.g += sunHighlight / 2.0; //увеливаем зеленую составляющую на тот же коэффициент но в 2 раза меньше (чтобы был оранжевый цвет)
       gl_FragColor = vec4(color, resultAlpha);
       return;
   } else {
       //свечение центральной части сферы
       sunHighlight *= 95.0; //увеличиваем коэффициент подсветки заката
       sunHighlight *= 1.0 + lightCos; //уменьшить (lightCos < 0.0) свечение при приближении к центр сферы (ограничиваем свечение краями, иначе - подсветим то что не может быть подсвечено)
       color = vec3(sunHighlight,sunHighlight / 2.0,0.0);
       resultAlpha = sunHighlight; //устанавливаем его как прозрачность
       gl_FragColor = vec4(color, resultAlpha);
       return;
   }
}

//...
```

Если текущая вершина соответствует эффекту заката, то после присваивания цвета _gl_FragColor_ происходит вызов _return_. При этом работа шейдера для таких вершин заканчивается.
Для остальных вершин все будет как раньше.
Стоит отметить что все "магические" коэффициенты подобраны опытным путем, для конкретного размера. Так что при изменении размер меша, надо будет изменить и их. Для удобства их можно вынести в управляемые uniform параметры.

Финальный фрагментный шейдер:

```
precision highp float;

varying vec3 vPositionW;
varying vec3 vNormalW;

varying vec2 vUV;

uniform sampler2D cloudsTexture;
uniform vec3 cameraPosition;
uniform vec3 lightPosition;


float computeFresnelTerm(vec3 viewDirection, vec3 normalW, float bias, float power)
{
    float fresnelTerm = pow(bias + dot(viewDirection, normalW), power);
    return clamp(fresnelTerm, 0., 1.);
}


void main(void) {
    vec3 viewDirectionW = normalize(cameraPosition - vPositionW); //Нормализованный вектор взгляда от камеры до вершины

    // Light
    vec3 direction = lightPosition - vPositionW; //Направление от источника света до вершины
    vec3 lightVectorW = normalize(direction); //Получение нормализованного вектора

    // lighting
    float lightCos = dot(vNormalW, lightVectorW); //Получаем косинус между направлением нормали вершины и направлением "луча света" вершину
    float lightDiffuse = max(0., lightCos); //рассчитываем коэффициент освещенности от 0 до 1

    vec3 color = texture2D(cloudsTexture, vUV).rgb; //получаем RGB составляющую цвета текселя по переданной UV координате из текстуры
    float globalAlpha = clamp(color.r, 0.0, 1.0); //определяем альфа составляющую

    // Fresnel
    float fresnelTerm = computeFresnelTerm(viewDirectionW, vNormalW, 0.72, 5.0);

    float resultAlpha; //результирующая альфа составляющая


    if (fresnelTerm < 0.95) {
        //это краевая, подсвечиваемая зона сферы
        float envDiffuse = clamp(pow(fresnelTerm - 0.92, 1.0/2.0) * 2.0, 0.0, 1.0); //коэффициент рассеивания для смягчения границы перехода между центральной части и частию сияния
        resultAlpha = fresnelTerm * envDiffuse * lightCos; //получаем прозрачность умножая коэффициент Френеля на косинус мешду светом и взглядом на вершину
        color = color / 2.0 + vec3(0.0,0.5,0.7); //уменьшаем базовый текстурный цвет на 4 и прибавляем голубой
    } else {
        //это центр сферы, должен демонстрировать
        resultAlpha = fresnelTerm * globalAlpha * lightDiffuse;
    }

    //эффект заката
    float backLightCos = dot(viewDirectionW, lightVectorW); //косинус между вектором взгляда на вершину и вектором луча света
    float cosConst = 0.9; // граница расчета эффекта заката. 0.9 => угол в ~(155 - 205) градусов

    //если угол между вектором взгляда и лучом света  ~(155 - 205) градусов
    if (backLightCos < -cosConst) {
       //Обработка свечения с обратной стороны
       float sunHighlight = pow(backLightCos+cosConst, 2.0); //коэффициент подсветки
       if (fresnelTerm < 0.9) {
           //если это край атмосферы (подсвечиваемая часть) то для нее такой расчет
           sunHighlight *= 65.0; //увеличиваем коэффициент подсветки заката
           float envDiffuse = clamp(pow(fresnelTerm - 0.92, 1.0/2.0) * 2.0, 0.0, 1.0);
           resultAlpha = sunHighlight; //устанавливаем его как прозрачность
           color *= lightDiffuse; //умножаем основной цвет на коэффициент освещенности
           color.r += sunHighlight; //увеличиваем красную составляющую на коэффициант подсветки заката
           color.g += sunHighlight / 2.0; //увеливаем зеленую составляющую на тот же коэффициент но в 2 раза меньше (чтобы был оранжевый цвета)
           gl_FragColor = vec4(color, resultAlpha);
           return;
       } else {
           //свечение центральной части сферы
           sunHighlight *= 95.0; //увеличиваем коэффициент подсветки заката
           sunHighlight *= 1.0 + lightCos; //уменьшить (lightCos < 0.0) свечение при приближении к центр сферы (ограничиваем свечение краями, иначе - подсветим то что не может быть подсвечено)
           color = vec3(sunHighlight,sunHighlight / 2.0,0.0);
           resultAlpha = sunHighlight; //устанавливаем его как прозрачность
           gl_FragColor = vec4(color, resultAlpha);
           return;
       }
    }

    gl_FragColor = vec4(color * lightDiffuse, resultAlpha);
}
```

Мы получили такой результат:

![image](/assets/posts/webgl-babylonjs-p2/p4-result1.png)
![image](/assets/posts/webgl-babylonjs-p2/p4-result2.png)

Рис. 3. Готовая атмосфера Земли

Код этой части можно посмотреть [здесь](https://github.com/zivaaa/bjs-tutorial/blob/master/web/p4.html)

---

## Включаем солнце

Мы уже создали Землю и Луну, и познакомились с материалами и шейдерами. Настало время добавить больше красоты в наше космическое пространство.

В этой части мы познакомимся с процедурными текстурами BabylonJS и пост-обработкой. Создадим меш, который будет испускать солнечные лучи.

Добавим меш, который в последствии станет Солнцем:

```
//...

//добавляем параметр радиуса солнца в конфиг
var config = {
    //...
    SUN_RADIUS: 20,// радиус Солнца
    //...
};

//...

//...
//солнце
var sun = BABYLON.Mesh.CreateSphere("sun", 15, config.SUN_RADIUS, scene, true);
//...
```

У нас появился серый меш радиусом 20.
Текстура Солнца должна изменяться, а для этого нужна особая текстура под названием ```ProceduralTexture```.

Создадим материал и применим в нем процедурную текстуру.

```ProceduralTexture``` - такой вид текстуры позволяет нам генерировать ее визуализацию в GPU через фрагментный шейдер.
К счастью Babylon js уже имеет нужную нам динамическую текстуру под названием  ```FireProceduralTexture```, ее и применим.

Скачиваем скрипт с текстурой в папку ```materials``` и подключаем на страницу, как указано ниже.

```
<script src="materials/babylon.fireProceduralTexture.js"></script>
```

Теперь создаем материал и используем на него динамическую текстуру.

```
//создаем материал для Солнца
var sunMaterial = new BABYLON.StandardMaterial("sunMaterial", scene);
//создаем процедурную текстуру (128 - это разрешение)
var fireTexture = new BABYLON.FireProceduralTexture("fire", 128, scene);
//задаем 6 основный цветов
fireTexture.fireColors = [
    new BABYLON.Color3(1.0,.7,0.3),
    new BABYLON.Color3(1.0,0.7,0.3),
    new BABYLON.Color3(1.0,.5,0.0),
    new BABYLON.Color3(1.0,.5,0.0),
    new BABYLON.Color3(1.0,1.0,1.0),
    new BABYLON.Color3(1.0,.5,0.0),
];

//задаем материалу emissiveTexture
sunMaterial.emissiveTexture = fireTexture;

sun.material = sunMaterial; //присваиваем материал
sun.parent = lightSourceMesh; //прикрепляем Солнце к источнику света
```

fireTexture.fireColors = [...] - здесь задаются шесть основный цветов для генерируемой текстуры, меняя их мы меняем цвет огненной текстуры на Солнце.

У нас есть меш, текстура которого динамически меняется со временем. На анимации ниже не вращающийся меш, а динамическая текстура.

![image](/assets/posts/webgl-babylonjs-p2/p5-procedural.gif)

Рис. 1. Динамическая текстура на солнце (gif)

Эта текстура позволит создать эффект имитирующий лучи света.
Эффект называется *god rays* и он есть в BabylonJS.
То, что мы будем использовать дальше - это эффект _пост-обработки_ или _post process_.

Мы берем сцену на конкретном этапе, рендерим ее, и после изменяем, перед тем как вывести на экран.

К эффектам пост-обработки относятся такие эффекты как:

* blur (размытие)
* сепия
* черно-белый фильтр
* эффект "god rays"
* bloom (сияние)

Некоторые из эффектов довольно ресурсозатратны (например размытие).

Алгоритм создания пост-обработки в WebGL:

1. Сначала делаем один прогон сцены и рендерим ее в буфер
2. Имея готовую отрисованную сцену в буфере производим манипуляции с изображением (например инвертируем пиксели, для эффекта негатива) посредством специального шейдера
3. Выводим на экран результат обработки

Шейдер, который реализует пост-обработку, работает с текстурой, полученной после отрисовки сцены, а не обрабатывет всю сцену снова. В этом его главное отличие от шейдеров модели.
Именно потому что шейдер пост-обработки имеет доступ ко всей сцене сразу - через него и нужно делать конечные преобразования перед выводом на экран.

В BabylonJS  есть несколько готовых эффектов, а также возможность создания своего эффекта.
Используется для этого ```BABYLON.PostProcess``` и применяется к нужной камере.

Воспользуемся эффектом ```VolumetricLightScatteringPostProcess```, который после настройки даст нам необходимый результат.

```
//...

//создаем эффект "god rays" (name, pixel ratio, camera, целевой меш, quality, метод фильтрации, engine, флаг reusable)
var godrays = new BABYLON.VolumetricLightScatteringPostProcess('godrays', 1.0, camera, sun, 100, BABYLON.Texture.BILINEAR_SAMPLINGMODE, engine, false);

//...
```

И вот что мы получим на выходе, добавив 1 строчку:

![image](/assets/posts/webgl-babylonjs-p2/p5-godrays1.png)

Рис. 2. Эффект "god rays".

Можно улучшить качество эффекта увеличив *quality*. Это изменит количество сэмплов для отображаемых лучей.
Можно уменьшить второй параметр ```pixel ratio```, тогда качество картинки упадет в целом, потому что он влияет на разрешение картинки.
Однако эти параметры напрямую влияют на качество эффекта, а кастомизировать его нужно немного по-другому.

Вот какими параметрами нужно управлять:

```
//...

godrays.exposure = 0.95;
godrays.decay = 0.96815;
godrays.weight = 0.78767;
godrays.density = 1.0;

//...
```

Установив новые значения мы увеличили яркость лучей и теперь эффект "god rays" стал более выразителен

![image](/assets/posts/webgl-babylonjs-p2/p5-godrays2.png)

Рис. 3. Эффект "god rays" после настройки.

Так BabylonJs позволяет добавлять на сцену различные эффекты.
Вот еще пара примеров эффектов, которые можно добавить:

```
var postProcess = new BABYLON.BlackAndWhitePostProcess("bandw", 1.0, null, null, engine, true); //black and white
var postProcess = new BABYLON.BlurPostProcess("Horizontal blur", new BABYLON.Vector2(1.5, 0), 1.0, 1.0, null, null, engine, true); //blur

camera.attachPostProcess(postProcess);
```

Код приложения на текущий момент [здесь](https://github.com/zivaaa/bjs-tutorial/blob/master/web/p5.html)

---

## Анимация

Анимация создается путем изменения положения мешей перед каждым рендером.
Например можно менять параметр ```SomeMesh.position.x++``` и двигать меш по X при каждой отрисовке.
Или же задать полный набор позиций в определенный момент:

```SomeMesh.position = new BABYLON.Vector3(5.0, 5.0, 4.0);```

Это приведет к установке меша в указанные координаты.
Тоже самое относится к вращению, только вместо ```position``` надо использовать ```rotation```.

В нашей сцене уже реализовано вращение Земли. Т.к. Луна привязана к Земле, то вращается вместе с ней.

Мы отключим эту привязку через параметр _parent_ и зададим для Луны вращение по эллиптической траектории вокруг Земли.

Убираем привязку

```

// moon.parent = planet; //задаем родителя - Землю

```

Добавим шаг изменения положения в config

```
var config = {
    //...
    MOON_ROTATION: 0.005, //шаг вращения Луны
};
```

Добавим объект настроек для расчета позиции:

```
var moonEllipticParams;
({
    init: function() {
        moonEllipticParams = this;
        this.delta = config.MOON_ROTATION; //смещение по углу (рад.)
        this.focus = 1.5; //множитель удлинения траектории по оси
        this.angle = 0; //начальный угол
        /*центр вращения*/
        this.x = planet.position.x;
        this.y = planet.position.y;
        this.z = planet.position.z;
        //радиус вращения
        this.r = BABYLON.Vector2.Distance(new BABYLON.Vector2(moon.position.x, moon.position.z), new BABYLON.Vector2(planet.position.x, planet.position.z))
    }
}).init();
```

Все параметры достаточно просты, стоит выделить лишь:
 - ```focus``` - фокусное расстояние, если оно равно 1, то траектория будет не эллиптическая, а круговая
 - ```r``` - базовый радиус вращения Луны. Рассчитывается, как дистанция между точками статическим методом BABYLON.Vector2.Distance(vec2, vec2).
Дистанция считается как корень из суммы квадратов разностей координат
```sqrt((y2 – y1)^2 + (x2-x1)^2)```

Функция расчета новой позиции (будем вращать в плоскости xz):

```
function getNewEllipticPosition(p) {
    p.angle += p.delta;
    //x + R*sin(a), y, z + R*cos(a)
    return new BABYLON.Vector3(p.x + p.r * Math.sin(p.angle), p.y, p.z + p.focus * p.r * Math.cos(p.angle));
}
```

```getNewEllipticPosition``` получает объект настроек, добавляет шаг поворота и рассчитывает новую позицию возвращая новые _xyz_ координаты.

Добавляем команду выполнения в функции ```renderLoop``` и видим как Луна вращается вокруг Земли.
Также Луна вращается вокруг своей оси y:

```
engine.runRenderLoop(function() { //инициируем перерисовку

    //...
    moon.position = getNewEllipticPosition(moonEllipticParams);
    moon.rotation.y += 0.006; //вращение вокруг оси y
    //...
});
```

![image](/assets/posts/webgl-babylonjs-p2/p6-result.gif)

Рис. 1. Вращение Луны вокруг земли в глобальных координатах.

Код приложения на текущий момент [здесь](https://github.com/zivaaa/bjs-tutorial/blob/master/web/p6.html)

---

## Финальный аккорд

Добавим космическую пыль. Это будут небольшие частички, которые будут разбросаны по пространству в случайном порядке.
Частички называются *спрайтами*. ```Спрайт``` - это простой объект, представляющий собой прямоугольник с наложенной текстурой, повернутый "лицом" к камере.

Спрайты обычно используются в 3D для создания:

* отдаленных объектов, отрисовка которых в виде 3D объектов была бы накладной;
* атмосферных объектов (облака);
* эффектов (огонь, дождь и прочее)
* простых анимированных элементов (например насекомых на полу)

Для частиц и эффектов в BabylonJS есть ```SpriteManager``` и ```ParticleSystem```.

Нам нужен менеджер частиц.

Нам потребуется [текстура](https://github.com/zivaaa/bjs-tutorial/blob/master/textures/particle32.png) и ```SpriteManager```, который объявим, указав нашу текстуру.

```
var config = {
    //...
    DUST: 1000// количество частиц
};

//...

var spriteManagerDust = new BABYLON.SpriteManager("dustManager", "textures/particle32.png", config.DUST, 32, scene);
```

В конфиге мы указали число спрайтов. Затем создали менеджер спрайтов, задали ему текстуру, число спрайтов и их размер.
Объявим функцию генерации частиц

```
function generateSpaceDust() {
    for (var i = 0; i < config.DUST; i++) {
        var dust = new BABYLON.Sprite("dust", spriteManagerDust); //создаем спрайт
        dust.position.x = Math.random() * 500 - 250; //случайная позиция x
        dust.position.z = Math.random() * 500 - 250;//случайная позиция y
        dust.position.y = Math.random() * 150 - 75; //случайная позиция z
        dust.size = 0.4; //задаем размер - 0.4 от максимального
    }
}
```

Данная функция генерирует случае распределение 1000 спрайтов по координатам:

```-250 < x < 250```

```-250 < z < 250```

```-75 < z < 75```

Нужно учитывать, что подобное распределение не берет в расчет положение мешей и спрайты легко могут оказаться внутри мешей.
Исходя из области распределения частиц и размеров планет, для каждой частицы вероятность попадания в планету равна примерно 3%.
А если рассматривать всю совокупность частиц на сцене, то хоть одна такая частица обязательно попадет в область планеты.
Но в нашем случае это не критично, так как спрайты достаточно малы и усложнять расчет смысла нет.

Вызываем функцию и звездная пыль готова!

```
//...

generateSpaceDust();

//...
```

Перед тем как завершить создание сцены, немного донастроим камеру.

Изменим поле зрения камеры, зададим минимальный и максимальный угол обзора, включим автовращение.

Назначим новые параметры камеры:

```
//...


camera.fov = 1.5; //область видимости камеры
camera.lowerBetaLimit = 0.5; //минимальный угол beta
camera.upperBetaLimit = 2.5; //максимальный угол beta
camera.lowerRadiusLimit = config.PLANET_RADIUS + 1; //минимальный радиус камеры
camera.radius = 60; //задаем начальную дистанцию от точки фокуса

//...
```

Эти параметры задали ограничение обзора по параметру beta, установили минимальную дистанцию камеры от точки фокуса (Земли) и задали увеличенное поле зрения.

![image](/assets/posts/webgl-babylonjs-p2/p7-fovdust.gif)

Рис. 1. Звездная пыль (спрайты).

Добавим автовращение в функцию ```renderLoop```.

```
camera.alpha += 0.005;
```

Наша сцена готова.

Финальный код приложения доступен [здесь](https://github.com/zivaaa/bjs-tutorial/blob/master/web/p7.html)

---

## Заключение. Список Литературы

Мы познакомились с общими принципами работы с 3D и WebGL, разобрались с шейдерами, спрайтами, материалами и научились работать с BabylonJS.

![image](/assets/posts/webgl-babylonjs-p2/result3.png)
Рис. 1. Наш результат — готовая сцена космоса.

---

Для более глубокого изучения WebGL рекомендуем следующие материалы:

*Книги*
- Коичи Мацуда, Роджер Ли "WebGL. Программирование трехмерной графики
- OpenGL ES 2.0 Programming Guide

*WebGl*
- https://developer.mozilla.org/ru/docs/Web/API/WebGL_API/Tutorial/Getting_started_with_WebGL - туториал по webgl
- http://webgl-lessons.blogspot.ru/p/blog-page_26.html - еще уроки по webGL

*Babylon js*
- http://www.babylonjs.com - Babylon js демо
- https://www.eternalcoding.com/?p=113 - шейдеры и шейдерные материалы
- https://doc.babylonjs.com/tutorials - туториалы
- http://doc.babylonjs.com/classes/2.4 - документация

*Математические основы*
- http://www.opengl-tutorial.org/ru/beginners-tutorials/tutorial-3-matrices/ - матрицы преобразования
- http://pmg.org.ru/basic3d/math.htm - что такое матрицы и как с ними обращаться
- https://habrahabr.ru/post/131931/ - линейная алгебра



