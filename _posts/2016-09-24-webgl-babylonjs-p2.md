---
layout: post
title:  "Знакомимся с WebGL и BabylonJS (часть 2)"
date:   2016-09-24 20:00:00 +0300
permalink: /webgl-babylonjs-p2/
tags: [webgl, babylonjs, js]
keywords: "webgl, babylonjs, js"
author: zivaaa
---

Продолжим создание нашей космической сцены с&nbsp;помощью BabylonJS. В [первой части](/webgl-babylonjs-p1) мы&nbsp;создали окружение, Землю и&nbsp;Луну и&nbsp;наложили материалы. Нам осталось добавить атмосферу, солнечный свет и&nbsp;анимацию.

<!--more-->

## Добавляем атмосферу

Земля готова, но&nbsp;на&nbsp;ней не&nbsp;хватает атмосферы. Сымитируем ее&nbsp;следующими эффектами:

* Вращающиеся облака
* Краевая голубая подсветка
* При взгляде с&nbsp;обратной стороны на&nbsp;источник света должен быть эффект заката (оранжевое свечение)

Для создания этих эффектов нам потребуется:

* Сферический меш размером чуть больше Земли и&nbsp;установленный в&nbsp;той&nbsp;же позиции
* Текстура облаков
* Шейдерный материал
* Пара шейдеров

![image](/assets/posts/webgl-babylonjs-p1/earth-c.jpg)

Рис. 1. Текстура атмосферы.

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

Кроме положения источника света нам потребуется следить за&nbsp;положением камеры, чтобы реализовать нужные эффекты.
Поэтому в&nbsp;функции ```renderLoop``` добавим команды обновления этих значений.

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

Он&nbsp;идентичен вершинному шейдеру Земли. Вся сложность в&nbsp;описании фрагментного шейдера.

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

``` vPositionW ``` &mdash; позиция вершины

``` vNormalW ``` &mdash; нормаль вершины

``` vUV ``` &mdash; uv&nbsp;координаты текстуры

``` cloudsTexture ``` &mdash; текстура облаков

``` cameraPosition ``` &mdash; позиция камеры в&nbsp;мире

``` lightPosition ``` &mdash; позиция источника света

Здесь все должно быть понятно, перейдем к&nbsp;расчетам

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

В&nbsp;текстуре нет альфа составляющей и&nbsp;прозрачная часть меша должна рассчитываться из&nbsp;цветовой.
Поэтому получаем цвет [текселя](https://ru.wikipedia.org/wiki/%D0%A2%D0%B5%D0%BA%D1%81%D0%B5%D0%BB_(%D0%B3%D1%80%D0%B0%D1%84%D0%B8%D0%BA%D0%B0)) и&nbsp;присваиваем альфа составляющей значение красной составляющей. Мы&nbsp;могли&nbsp;бы взять любую другую, т.к. текстура черно-белая и&nbsp;они все равны.
Чем чернее тем прозрачнее будет текстура в&nbsp;каждом обрабатываемом фрагменте.

В&nbsp;конце умножаем цвет фрагмента на&nbsp;коэффициент освещенности, чтобы сделать облака в&nbsp;теневой части темнее.

![image](/assets/posts/webgl-babylonjs-p2/p4-env1.png)

Рис.&nbsp;2. Первая версия атмосферы.

Чтобы правильно отобразить центр и&nbsp;края сферы нам нужна функция преломления.

```
float computeFresnelTerm(vec3 viewDirection, vec3 normalW, float bias, float power)
{
    float fresnelTerm = pow(bias + dot(viewDirection, normalW), power);
    return clamp(fresnelTerm, 0., 1.);
}
```

В&nbsp;зависимости от&nbsp;значений ```bias``` и ```power``` мы&nbsp;получим коэффициент по&nbsp;которому определим край атмосферы для подсветки.
Функция ```clamp```, служит для контроля границ значения. Она принимает такие аргументы ```clamp(value, min, max)```, где

```value``` &mdash;&nbsp; целевое проверяемое значение;

```min``` &mdash;&nbsp; минимальное выходное значение;

```max``` &mdash;&nbsp; максимальное выходное значение;

Возвращает *clamp* значение *value*, только если оно лежит в пределах *min* и *max*, иначе вернет соответственно либо *max* либо *min*.

Добавим обработку фрагмента в&nbsp;зависимости от&nbsp;коэффициента ```fresnelTerm```, который показывает степень преломления света (коэффициент Френеля). Нам он&nbsp;покажет когда надо рисовать краевую подсветку.

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

Если ```fresnelTerm``` &lt; 0.95&nbsp;&mdash; пора рисовать подсветку. При больших значениях рисуем как обычно.
В&nbsp;строке ```resultAlpha = fresnelTerm * lightCos``` используется _lightCos_, а&nbsp;не&nbsp;обработанный коэффициент освещения. Это пригодится дальше, при расчете взгляда с&nbsp;темной стороны планеты.
Мы&nbsp;также учли коэффициент освещения и&nbsp;облака будут постепенно пропадать в&nbsp;тени.

В&nbsp;результате у&nbsp;нас получится такая сцена:

![image](/assets/posts/webgl-babylonjs-p2/p4-env2.png)

Рис.&nbsp;3. Краевая подсветка атмосферы.

Улучшим атмосферу эффектом заката. Для этого нужна переменная, по&nbsp;которой можно судить о&nbsp;том, что &laquo;вектор света&raquo; и&nbsp;&laquo;вектор взгляда&raquo; направлены друг на&nbsp;друга, а&nbsp;также определить нужную зону эффекта.

```
//...
//эффект заката
float backLightCos = dot(viewDirectionW, lightVectorW); //косинус угла между вектором взгляда на вершину и вектором луча света
float cosConst = 0.9; //граница расссчета эффекта заката. 0.9 => угол в ~(155 - 205) градусов
//...
```

``` backLightCos ``` &mdash; этот коэффициент будет основой расчета новой подсветки, по факту это косинус угла между вектором взгляда на вершину и вектором луча света на вершину.

``` cosConst ``` &mdash; граница косинуса для эффекта заката

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

Если текущая вершина соответствует эффекту заката, то&nbsp;после присваивания цвета _gl_FragColor_ происходит вызов _return_. При этом работа шейдера для таких вершин заканчивается.
Для остальных вершин все будет как раньше.
Стоит отметить что все &laquo;магические&raquo; коэффициенты подобраны опытным путем, для конкретного размера. Так что при изменении размер меша, надо будет изменить и&nbsp;их. Для удобства их&nbsp;можно вынести в&nbsp;управляемые uniform параметры.

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

Мы&nbsp;получили такой результат:

![image](/assets/posts/webgl-babylonjs-p2/p4-result1.png)

Рис.&nbsp;4. Свечение атмосферы с&nbsp;обратной стороны.

Код этой части можно посмотреть [здесь](https://github.com/zivaaa/bjs-tutorial/blob/master/web/p4.html)

<br/>

## Включаем солнце

Мы&nbsp;уже создали Землю и&nbsp;Луну, и&nbsp;познакомились с&nbsp;материалами и&nbsp;шейдерами. Настало время добавить больше красоты в&nbsp;наше космическое пространство.

В&nbsp;этой части мы&nbsp;познакомимся с&nbsp;процедурными текстурами BabylonJS и&nbsp;пост-обработкой. Создадим меш, который будет испускать солнечные лучи.

Добавим меш, который позже станет Солнцем:

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

У&nbsp;нас появился серый меш радиусом 20.
Текстура Солнца должна изменяться, а&nbsp;для этого нужна особая текстура под названием ```ProceduralTexture```.

Создадим материал и&nbsp;применим в&nbsp;нем процедурную текстуру.

```ProceduralTexture```&mdash; эта текстура генерируется динамически &nbsp;в&nbsp;GPU через фрагментный шейдер.
К&nbsp;счастью Babylon js&nbsp;уже имеет нужную нам динамическую текстуру под названием ```FireProceduralTexture```, ее&nbsp;и&nbsp;применим.

Скачиваем скрипт с&nbsp;текстурой в&nbsp;папку ```materials``` и&nbsp;подключаем на&nbsp;страницу, как указано ниже.

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

fireTexture.fireColors = [...] &mdash; здесь задаются шесть основный цветов для генерируемой текстуры, меняя их&nbsp;мы&nbsp;меняем цвет огненной текстуры на&nbsp;Солнце.

У&nbsp;нас есть меш, текстура которого динамически меняется со&nbsp;временем. На&nbsp;анимации ниже не&nbsp;вращающийся меш, а&nbsp;динамическая текстура.

![image](/assets/posts/webgl-babylonjs-p2/p5-procedural.gif)

Рис.&nbsp;5. Динамическая текстура на&nbsp;солнце (gif).

Эта текстура позволит создать эффект имитирующий лучи света.
Эффект называется *god rays* и&nbsp;он&nbsp;есть в&nbsp;BabylonJS.
То, что мы&nbsp;будем использовать дальше&nbsp;&mdash; это эффект _пост-обработки_ или _post process_.

Мы&nbsp;берем сцену на&nbsp;конкретном этапе, рендерим&nbsp;ее, и&nbsp;после изменяем, перед тем как вывести на&nbsp;экран.

К&nbsp;эффектам пост-обработки относятся такие эффекты как:

* blur (размытие)
* сепия
* черно-белый фильтр
* эффект &laquo;god rays&raquo;
* bloom (сияние)

Некоторые из&nbsp;эффектов довольно ресурсозатратны (например размытие).

Алгоритм создания пост-обработки в&nbsp;WebGL:

1. Сначала делаем один прогон сцены и&nbsp;рендерим ее&nbsp;в&nbsp;буфер
2. Имея готовую отрисованную сцену в&nbsp;буфере производим манипуляции с&nbsp;изображением (например инвертируем пиксели, для эффекта негатива) посредством специального шейдера
3. Выводим на&nbsp;экран результат обработки

Шейдер, который реализует пост-обработку, работает с&nbsp;текстурой, полученной после отрисовки сцены, а&nbsp;не&nbsp;обрабатывет всю сцену снова. В&nbsp;этом его главное отличие от&nbsp;шейдеров модели.
Именно потому что шейдер пост-обработки имеет доступ ко&nbsp;всей сцене сразу&nbsp;&mdash; через него и&nbsp;нужно делать конечные преобразования перед выводом на&nbsp;экран.

В&nbsp;BabylonJS есть несколько готовых эффектов, а&nbsp;также возможность создания своего эффекта.
Используется для этого ```BABYLON.PostProcess``` и&nbsp;применяется к&nbsp;нужной камере.

Воспользуемся эффектом ```VolumetricLightScatteringPostProcess```, который после настройки даст нам необходимый результат.

```
//...

//создаем эффект &laquo;god rays&raquo; (name, pixel ratio, camera, целевой меш, quality, метод фильтрации, engine, флаг reusable)
var godrays = new BABYLON.VolumetricLightScatteringPostProcess('godrays', 1.0, camera, sun, 100, BABYLON.Texture.BILINEAR_SAMPLINGMODE, engine, false);

//...
```

И&nbsp;вот что мы&nbsp;получим на&nbsp;выходе, добавив 1&nbsp;строчку:

![image](/assets/posts/webgl-babylonjs-p2/p5-godrays1.png)

Рис.&nbsp;6. Эффект &laquo;god rays&raquo;.

Можно улучшить качество эффекта увеличив *quality*. Это изменит количество сэмплов для отображаемых лучей.
Можно уменьшить второй параметр ```pixel ratio```, тогда качество картинки упадет в&nbsp;целом, потому что он&nbsp;влияет на&nbsp;разрешение картинки.
Однако эти параметры напрямую влияют на&nbsp;качество эффекта, а&nbsp;кастомизировать его нужно немного по-другому.

Вот какими параметрами нужно управлять:

```
//...

godrays.exposure = 0.95;
godrays.decay = 0.96815;
godrays.weight = 0.78767;
godrays.density = 1.0;

//...
```

Установив новые значения мы&nbsp;увеличили яркость лучей и&nbsp;теперь эффект &laquo;god rays&raquo; стал более выразителен.

![image](/assets/posts/webgl-babylonjs-p2/p5-godrays2.png)

Рис.&nbsp;7. Эффект &laquo;god rays&raquo; после настройки.

Так BabylonJs позволяет добавлять на&nbsp;сцену различные эффекты.
Вот еще пара примеров эффектов, которые можно добавить:

```
var postProcess = new BABYLON.BlackAndWhitePostProcess("bandw", 1.0, null, null, engine, true); //black and white
var postProcess = new BABYLON.BlurPostProcess("Horizontal blur", new BABYLON.Vector2(1.5, 0), 1.0, 1.0, null, null, engine, true); //blur

camera.attachPostProcess(postProcess);
```

Код приложения на текущий момент [здесь](https://github.com/zivaaa/bjs-tutorial/blob/master/web/p5.html).

<br/>

## Анимация

Анимация создается путем изменения положения мешей перед каждым рендером.
Например можно менять параметр ```SomeMesh.position.x++``` и&nbsp;двигать меш по&nbsp;X при каждой отрисовке.
Или&nbsp;же задать полный набор позиций в&nbsp;определенный момент:

```SomeMesh.position = new BABYLON.Vector3(5.0, 5.0, 4.0);```

Это приведет к&nbsp;установке меша в&nbsp;указанные координаты.
Тоже самое относится к&nbsp;вращению, только вместо ```position``` надо использовать ```rotation```.

В&nbsp;нашей сцене уже реализовано вращение Земли. Т.к. Луна привязана к&nbsp;Земле, то&nbsp;вращается вместе с&nbsp;ней.

Мы&nbsp;отключим эту привязку через параметр _parent_ и&nbsp;зададим для Луны вращение по&nbsp;эллиптической траектории вокруг Земли.

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

* ```focus``` &mdash; коэффициент растяжения круга по&nbsp;одной из&nbsp;осей, если оно равно&nbsp;1, то&nbsp;траектория будет не&nbsp;эллиптическая, а&nbsp;круговая

* ```r``` &mdash; базовый радиус вращения Луны. Рассчитывается, как дистанция между точками статическим методом BABYLON.Vector2.Distance(vec2, vec2).
Дистанция считается как корень из&nbsp;суммы квадратов разностей координат
```sqrt((y2-y1)^2 + (x2-x1)^2)```

Функция расчета новой позиции (будем вращать в&nbsp;плоскости&nbsp;xz):

```
function getNewEllipticPosition(p) {
    p.angle += p.delta;
    //x + R*sin(a), y, z + R*cos(a)
    return new BABYLON.Vector3(p.x + p.r * Math.sin(p.angle), p.y, p.z + p.focus * p.r * Math.cos(p.angle));
}
```

```getNewEllipticPosition``` получает объект настроек, добавляет шаг поворота и&nbsp;рассчитывает новую позицию возвращая новые _xyz_ координаты.

Добавляем команду выполнения в&nbsp;функции ```renderLoop``` и&nbsp;видим как Луна вращается вокруг Земли.
Также Луна вращается вокруг своей оси&nbsp;y:

```
engine.runRenderLoop(function() { //инициируем перерисовку

    //...
    moon.position = getNewEllipticPosition(moonEllipticParams);
    moon.rotation.y += 0.006; //вращение вокруг оси y
    //...
});
```

![image](/assets/posts/webgl-babylonjs-p2/p6-result.gif)

Рис. 8. Вращение Луны вокруг земли в глобальных координатах.

Код приложения на текущий момент [здесь](https://github.com/zivaaa/bjs-tutorial/blob/master/web/p6.html).

<br/>

## Финальный аккорд

Добавим космическую пыль. Это будут небольшие частички, которые будут разбросаны по&nbsp;пространству в&nbsp;случайном порядке.
Частички называются *спрайтами*. ```Спрайт```&mdash;это простой объект, представляющий собой прямоугольник с&nbsp;наложенной текстурой, повернутый &laquo;лицом&raquo; к&nbsp;камере.

Спрайты обычно используются в&nbsp;3D&nbsp;для создания:

* отдаленных объектов, отрисовка которых в&nbsp;виде 3D&nbsp;объектов была&nbsp;бы накладной;
* атмосферных объектов (облака);
* эффектов (огонь, дождь и&nbsp;прочее)
* простых анимированных элементов (например насекомых на&nbsp;полу)

Для частиц и&nbsp;эффектов в&nbsp;BabylonJS есть ```SpriteManager``` и ```ParticleSystem```.

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

В&nbsp;конфиге мы&nbsp;указали число спрайтов. Затем создали менеджер спрайтов, задали ему текстуру, число спрайтов и&nbsp;их&nbsp;размер.
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

Нужно учитывать, что подобное распределение не&nbsp;берет в&nbsp;расчет положение мешей и&nbsp;спрайты легко могут оказаться внутри мешей.
Исходя из&nbsp;области распределения частиц и&nbsp;размеров планет, для одной частицы вероятность попадания в&nbsp;планету равна примерно 3%.
А&nbsp;если рассматривать всю совокупность частиц на&nbsp;сцене, то&nbsp;хоть одна такая частица обязательно попадет в&nbsp;область планеты.
Но&nbsp;в&nbsp;нашем случае это не&nbsp;критично, так как спрайты достаточно малы и&nbsp;усложнять расчет смысла нет.

Вызываем функцию и&nbsp;звездная пыль готова!

```
//...

generateSpaceDust();

//...
```

Перед тем как завершить создание сцены, немного донастроим камеру.

Изменим поле зрения камеры, зададим минимальный и&nbsp;максимальный угол обзора, включим автовращение.

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

Рис. 9. Звездная пыль (спрайты).

Добавим автовращение в функцию ```renderLoop```.

```
camera.alpha += 0.005;
```

Наша сцена готова.

Финальный код приложения доступен [здесь](https://github.com/zivaaa/bjs-tutorial/blob/master/web/p7.html).

<br/>

## Заключение. Список Литературы

Мы&nbsp;познакомились с&nbsp;общими принципами работы с&nbsp;3D&nbsp;и&nbsp;WebGL, разобрались с&nbsp;шейдерами, спрайтами, материалами и&nbsp;научились работать с&nbsp;BabylonJS.

![image](/assets/posts/webgl-babylonjs-p2/result3.png)

Рис.&nbsp;10. Наш результат&nbsp;&mdash; готовая сцена космоса.

<br/>

---

<br/>

Для более глубокого изучения WebGL рекомендуем следующие материалы:

*Книги*
- Коичи Мацуда, Роджер Ли &laquo;WebGL. Программирование трехмерной графики&raquo;
- OpenGL ES 2.0 Programming Guide

*WebGl*
- [туториал по webgl](https://developer.mozilla.org/ru/docs/Web/API/WebGL_API/Tutorial/Getting_started_with_WebGL)
- [еще уроки по webGL](http://webgl-lessons.blogspot.ru/p/blog-page_26.html)

*Babylon js*
- [Babylon js демо](http://www.babylonjs.com)
- [шейдеры и шейдерные материалы](https://www.eternalcoding.com/?p=113)
- [официальные туториалы](https://doc.babylonjs.com/tutorials)
- [документация по движку](http://doc.babylonjs.com/classes/2.4)

*Математические основы*
- [матрицы преобразования](http://www.opengl-tutorial.org/ru/beginners-tutorials/tutorial-3-matrices/)
- [что такое матрицы и как с ними обращаться](http://pmg.org.ru/basic3d/math.htm)
- [линейная алгебра](https://habrahabr.ru/post/131931/)



