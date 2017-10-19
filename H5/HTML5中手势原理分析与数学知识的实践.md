
## 实现原理
所有的手势都是基于浏览器原生事件`touchstart`,`touchmove`,`touchend`,`touchcancel`进行的上层封装，因此封装的思路是通过一个个相互独立的事件回调仓库`handlebus`,然后在原生`touch`事件中符合条件的时机触发并传出计算后的参数值，完成手势的操作。

## 基础数学知识函数
我们常见的坐标系属于线性空间,或称向量空间(Vector Space).这个空间是由点(Point)好的向量(Vector)所组成的集合.

### 点(Point)
可以理解为坐标点,例如原点O(0,0),A(-1,2),通过原生事件的`touches`可以获取触摸点的坐标,参数`index`代表第几个触摸点;
```js
getPoint(ev, index){
    return {
        x: Math.round(ev.touches[index].pageX),
        y: Math.round(ev.touches[index].pageY)
    }
}
```

### 向量(Vector)
是坐标系中一种**既有大小也有方向的线段**,例如由原点O(0,0)指向点A(1,1)的箭头线段,称为向量a,则a = (1-0,1-0) = (1,1).
如下图所示,其中i与j向量称为该坐标系的单位向量,也称为基向量,我们常见坐标系单位为1,即i=(1, 0),j=(0,1).
![图1](https://sfault-image.b0.upaiyun.com/135/825/1358258230-59847ed4a6e63_articlex)

获取向量的函数:
```javascript
getVector(p1, p2) {
    let x = Math.round(p1.x - p2.x),
        y = Math.round(p1.y - p2.y);
    return {x, y};
}
```

### 向量模
代表**向量的长度**,记为`|a|` 是一个标量,没有大小,没有方向.
几何意义代表的是以`x,y`为直角边的直角三角形的斜边,通过勾股定理进行计算.
![图2](https://sfault-image.b0.upaiyun.com/189/378/1893784804-59847f0835cd5_articlex)

`getLength`函数:
```js
getLength(v1) {
    return Math.sqrt(v1.x*v1.x + v1.y*v1.y)
}
```

### 向量的数量积
向量同样也具有可以运算的属性,它可以进行加,减,乘,数量积和向量积等运算.
数量积也称为点积,被定义为公式:
> 当a=(x1,y1),b=(x2,y2),则a*b=|a|*|b|*cosθ = x1*x2 + y1*y2;

### 共线定理
共线,即两个向量处于平行的状态,当`a=(x1,y1),b=(x2,y2)`,则存在唯一的一个实数λ,使得`a=λb`,带入坐标后,可以得到`x1*y2 = x2*y1`;

因此当`x1*y2 - x2*y1 > 0`时,即斜率`ka > kb`,所以此时b向量相对于a向量是属于顺时针旋转,反之,则为逆时针.

### 旋转角度

两个向量的夹角:
> cosθ=(x1*x2 + y1*y2)/(|a|*|b|)

然后通过共线定理可以判断出旋转的方向,函数定义为:
```js
getAngle(v1, v2) {
    // 判断方向, 顺时针为1,逆时针为-1
    let direction = v1.x * v2.y - v2.x * v1.y > 0 ? 1 : -1,
        // 两个向量的模;
        len1 = this.getLength(v1),
        len2 = this.getLength(v2),
        mr = len1 * len2,
        dot,
        r;
    if (mr === 0) {
        return 0;
    }
    // 通过数量积公式可以推导出:
    // cos = (x1 * x2 + y1 * y2)/(|a| * |b|);
    dot = v1.x * v2.x + v1.y * v2.y;
    r = dot /mr;
    if  (r > 1) {
        r = 1;
    }
    if (r < -1) {
        r = -1; 
    }
    return Math.acos(r) * direction * 180 / Math.PI;
}
```

### 矩阵与变换

由于空间最本质的特征是其可以容纳运动,因此在线性空间中,
> 我们用向量来刻画对象, 而矩阵便是用来描述对象的运动;

#### 矩阵如何描述运动
通过一个坐标系基向量便可以确定一个向量,例如`a = (-1,2)`,我们通常约定的基向量是i=(1,0),j=(0,1);因此:
> a = -1i + 2j = -1*(1, 0) + 2* = (-1 + 0, 0 + 2) = (-1, 2);

而矩阵变换的,其实便是通过矩阵转换了基向量,从而完成了向量的变换;

例如上面的例子,把a向量ton故宫矩阵(1,2,3,0)进行变换,此时基向量i由(1, 0 )变换成(1, -2)与j由(0,1)变换成(3,0),沿用上面的推导,则:
> a = -1i + 2j = -1*(-1,2) + 2*(3,0) = (5, -2);

如下图所示:
A图表示变换之前的坐标系,此时a=(-1,2),通过矩阵变换后,基于i, j的变换引起了坐标系的变换,变成了下图B,因此a向量由(-1, 2)变换成了(5, -2);
> 其实向量与坐标系的关联不变(a=-i + 2j),是基向量引起坐标系变化,然后坐标系沿用关联导致了向量的变化;

![](https://sfault-image.b0.upaiyun.com/328/770/3287703221-59848033abb15_articlex)

#### 结合代码

其实CSS的`transform`等变换便是通过矩阵进行的,我们平时所写的`translate/rotate`等语法类似于一种封装好的语法糖,便于快捷使用,而在底层都会被转换成矩阵的形式.例如`transform:translate(-30px,-30px)`编译后会被转换成`transform:matrix(1,0,0,1,30,30)`;

通常在二维坐标系中,只需要2\*2的矩阵便足以描述所有的变换了,但由于css是处于3D环境中的,因此css中使用的是3\*3的矩阵,表示为:

![](https://sfault-image.b0.upaiyun.com/521/131/521131991-5984804c9c4ae_articlex)

其实第三行的`0,0,1`代表的就是z轴的默认参数.这个矩阵中,`(a,b)`即为坐标轴的`i`基,而`(c,d)`即为`j`基,`e`为`x`轴的偏移量,`f`为`y`轴的偏移量;因此上栗便很好理解,`translate`并没有导致`i,j`基改变,只是发生了偏移,因此`translate(-30px, -30px) =>matrix(1,0,0,1,30,30)`.

所有的`transform`语句,都会发生对应的转换,如下:
```js
// 发生偏移，但基向量不变；
transform:translate(x,y) ==> transform:matrix(1,0,0,1,x,y)

// 基向量旋转；
transform:rotate(θdeg)==> transform:matrix(cos(θ·π/180),sin(θ·π/180),-sin(θ·π/180),cos(θ·π/180),0,0)

// 基向量放大且方向不变；
transform:scale(s) ==> transform:matrix(s,0,0,s,0,0)
```

`translate/rotate/scale`等语法十分强大,让代码更为可读且方便书写,但是`matrix`有着更强大的转换特性,通过`matrix`,可以发生任何方式的变换,例如我们常见的**镜像对称**,`transform:matrix(-1,0,0,1,0,0)`;

![](https://sfault-image.b0.upaiyun.com/429/250/4292507316-5984806781903_articlex)

#### MatrixTo

然而`matrix`虽然强大,但可读性却不好,而且我们的写入是通过`translate/rotate/scale`的属性,然而通过`getComputedStyle`读取到的`transform`却是`matrix`:
> transform:matrix(1.41421,1.41421,-1.41421,1.41421,-50,-50);

请问这个元素发生了怎么样的变化?这就一脸懵逼了.

因此,需要有个方法,来将`matrix`翻译成我们更为熟悉的`translate/rotate/scale`方式.

前4个参数会同时受到`rotate`和`scale`的影响,且有两个变量,因此需要通过前两个参数根据上面的转换方式列出两个不等式:

> cos(θ·π/180)*s=1.41421;
sin(θ·π/180)*s=1.41421;

将两个不等式相除,即可以求出`θ`和`s`了,函数如下:

![](https://sfault-image.b0.upaiyun.com/199/610/1996102881-5984807d26d7f_articlex)

## 手势原理

图例:

> 圆点: 代表手指的触碰点;
两个圆点之间的虚线段: 代表双指操作时组成的向量;
a向量/A点: 代表在touchstart时获取的初始向量/初始点;
b向量/B点: 代表在touchmove时获取的实时向量/实时点;
坐标轴底部的公式代表需要计算的值;

### Drag(拖动事件)

![](https://sfault-image.b0.upaiyun.com/602/224/602224203-5984808fdd5cf_articlex)

上图是模拟了拖动手势,由`A`点移动到`B`点,要计算的便是这个过程的偏移量;
因此我们在`touchstart`中记录初始点`A`的坐标:

```js
// 获取初始点A
let startPoint = getPoint(ev, 0);
```

然后在`touchmove`事件中获取当前点并实时计算出`△x`与`△y`：

```js
// 实时获取点B
let curPoint = getPoint(ev, 0);

// 通过A,B两点,实时的计算出位移增量,触发drag事件并传出参数;
_evevntFire('drag', {
    delta: {
        delatX: curPoint.x - startPoint.x,
        delatY: curPoint.y - startPoint.y
    },
    origin: ev
})
```

> Tips: fire 函数即遍历执行drag事件对应的回调仓库即可;

### Pinch(双指缩放)

![](https://sfault-image.b0.upaiyun.com/126/654/1266545224-598480a384b3a_articlex)

上图是双指缩放的模拟图,双指由`a`向量放大到`b`向量,通过初始状态时的`a`向量的模与`touchmove`中获取的`b`向量的进行计算,便可得出缩放值:

```js
// touchstart中计算初始双指的向量模
let vector1 = getVector(secondPoint, startPoint);
let pinchStartLength = getLength(vecotr1);

// touchmove中实时计算双指向量模
let vector2 = getVector(curSecPoint, curPoint);
let pinchLength = getLength(vector2);
this._eventFire('pinch', {
    delta: {
        scale: pinchLength / pinchStartLength
    },
    origin: ev
})
```

### Rotate(双指旋转)

![](https://sfault-image.b0.upaiyun.com/858/061/858061720-598480b5a3ad5_articlex)

初始时双指向量`a`,旋转到`b`向量, `θ`便是我们需要的值，因此只要通过我们上面构建的getAngle函数，便可求出旋转的角度：

```js
// a向量
let vector1 = getVector(secondPoint, startPoint);

// b向量
let vector2 = getVector(curSecPoint, curPoint);

// 触发事件
this._eventFire('rotate', {
    delta: {
        rotate: getAngle(vertor1, vector2)
    },
    origin: ev
})
```

### singlePinch(单指缩放)

![](https://sfault-image.b0.upaiyun.com/379/638/3796387613-598480cc21d2f_articlex)

单指缩放和danzhi旋转需要多个特有概念:

> 操作元素(opeartor): 需要操作的元素.上面三个手势都不关心操作元素,因为单纯靠手势自身,便能计算得出正确的参数值,而单指缩放和旋转需要依赖于操作元素的基准点(操作元素的中心点)进行计算;

> 按钮: 因为单指的手势与拖动(drag)手势是相互冲突的,需要一种特殊的交互方式来进行区分,这里是通过特定的区域来区分,类似于一个按钮,当在按钮操作时,是单指缩放或者旋转,而在按钮区域外,则是常规的拖动.

图中`a`向量单指放大到`b`向量,对操作元素进行了中心放大,此时缩放值即为`b`向量的模/`a`向量的模.

```js
// 计算单指操作时的基准点,获取操作元素的中心点
let singleBasePoint = getBasePoint(operator);

// touchstart中计算初始向量模
let pinchV1 = getVector(startPoint, singleBasePoint);
singlePinchStartLength = getLength(pinchV1);

// touchmove中计算实时向量模
let pinchV2 = getVector(curPoint, singleBasePoint);
singlePinchLength = getLength(pinchV2);

// 触发事件
this._eventFire('singlePinch', {
    delta: {
        scale: singlePinchLength / singlePinchStartLength,
    }
})
```

### singleRotate(单指旋转)

![](https://sfault-image.b0.upaiyun.com/285/525/2855252767-598480e13aa7d_articlex)

```js
// 获取初始向量与实时向量
let rotateV1 = getVector(startPoint, singleBasePoint);
let rotateV2 = getVector(curPoint, singleBasePoint);

// 通过getAngle获取旋转角度并触发事件
this._eventFire('singleRotate', {
    delta: {
        rotate: getAngle(rotateV1, rotateV2)
    },
    origin: ev
})
```

### 运动增量

由于`touchmove`事件是个高频率的实时触发事件,一个拖动操作,其实触发了N次的`touchmove`事件,因此计算出来的值只是一种增量,即代表的是一次`touchmove`事件增加的值,只代表一段很小的值,并不是最终的结果值,因此需要外部维护一个位置数据,类似于:
```js
// 真实位置数据
let dragTrans = {
    x: 0,
    y: 0
}

// 累加上所传出的增量deltaX与deltaY
dragTrans.x += ev.delta.deltaX;
dragTrans.y += ev.delta.deltaY;

// 通过transform直接操作元素
set($drag, dragTrans);
```

### 初始位置

维护外部的这个位置数据,如果初始值像上述那样直接取0,则遇到css设置了`transform`属性的元素便无法正确识别了,会导致操作元素开始时瞬间跳回(0,0)点,因此需要初始时取获取一个元素真实的位置值,再进行维护与操作,需要用到`getComputedStyle`方法与`matrixTo`函数:

```js
// 获取css transform属性, 此时得到的是一个矩阵数据
let style = window.getComputedStyle(el, null);
let cssTrans = style.transform;

// 按规则进行转换,得到:
let initTrans = _.matrixTo(cssTrans);

// {x:-50,y:-50,scale:2,rotate:45};
// 即该元素设置了：transform:translate(-50px,-50px) scale(2) rotate(45deg);
```

