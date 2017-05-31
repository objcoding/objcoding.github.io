---
layout: post
title:  "AngularJS的基本用法"
categories: FrontEnd
tags: AngularJS JavaScript
author: zch
---

* content
{:toc}
### 1. 表达式

AngularJS的表达式在页面中是以双大括号**{{ expression }}**来表示的，大括号里面可以填写文字，表达式，变量等。

*注：本文代码块中的AngularJS表达式都用了``包围，是为了能够在页面上显示出表达式。*

- 变量：

```
<div>合计: `{{sum}}`元</div>
```

sum变量是从$scope作用域拿的，后面会讲到。

- 表达式：

```
 <p>我的第一个表达式: `{{ 5 + 5 }}`</p>
```

- 字符串：

```
<div ng-app="" ng-init="firstName='John';lastName='Doe'">
<p>姓名： `{{ firstName + " " + lastName }}`</p>
</div>
```

### 2. 指令

AngularJS的指令可扩展html属性，AngularJS可通过指令与页面进行交互，且可以允许开发者自定义指令。AngularJS的指令都有ng-前缀，如下：

- ng-app：在页面上初始化一个AngularJS应用程序

```
<div ng-app="">
     <p>你输入的为： `{{ 1 + 1 }}`</p>
</div>
```

如果ng-app=""，那么会使用AngularJS默认初始化一个应用程序，如果不为空，则需要自定义一个应用程序：

```
angular.module('myApp', ['ngRoute'])
    .config(['$locationProvider', '$routeProvider', function ($locationProvider, $routeProvider) {
    ......
    }]).controller('AppCtrl', ['$scope','$http', function ($scope,$http) {
        ......
        }])
```

```
<div ng-app="myApp">
     <p>你输入的为： `{{ 1 + 1 }}`</p>
</div>
```

- ng-model（模型）：可把作用域的值绑定到应用程序中，或者可以把元素值绑定到输入域中，这种绑定采用双向绑定机制，即任何一方改变，另一方也跟着改变。

元素绑定：

```
<div ng-app="" ng-init="firstName='John'">
     <p>在输入框中尝试输入：</p>
     <p>姓名：<input type="text" ng-model="firstName"></p>
     <p>你输入的为： `{{ firstName }}`</p>
</div>
```

作用域绑定：

```
angular.module('myApp')
  .controller('AppCtrl', ['$scope','$http', function ($scope,$http) {
        $scope.shop.name="zhangsan"
        }])
```

```
<div ng-app="myApp">
     <div class="form-group" ng-controller="AppCtrl">
     	 <input type="text" class="form-control" ng-model="shop.name">
     </div>
</div>
```

- 其它指令

AngualrJS的指令非常丰富，比如ng-controller绑定一个控制器、ng-repeat类似于for循环指令、ng-if类似于if判断指令、ng-click点击事件等等，都是非常好用的指令。

### 3. 作用域

在说作用域之前，我们先了解一下AngualrJS应用的组成：

1. veiw：视图，即html页面；
2. model：模型，即页面所需要的数据；
3. Controller：控制器，可以对模型的数据进行修改或添加或删除。

前面我们有提到过作用域这东西，指的就是model，它是AngualrJS在HTML域JS之间传值的一个非常重要的东西，通常我们从后台请求回来的数据都会放在作用域中，然后HTML通过作用域拿到数据，展示给用户。**$scope只作用于其所在的控制器ng-controller包含的html。**还有一个$rootScope根作用域，它是作用于ng-app包含的html。

### 4. 控制器

模型的数值由控制器控制，因此控制器通常操作数据，向后台请求数据，把数据存到$scope作用域中。控制器有ng-controller指令定义。

```
angular.module('myApp')
.controller('ShopCtrl', ['$scope', '$http', function ($scope, $http) {
  function requestTableData() {
      $http({
          method: "GET",
          url: HostManageService._shopHost + "/api/shop/shops/" + $scope.searchParams.page,
      }).then(function (data, status) {
          $scope.tableData = data.shop;
      })
  }
  requestTableData();
}])	
```

```
......
<div ng-app="myApp">
    <table ng-controller="ShopCtrl">
            <thead>
            <tr>
                <th>姓名</th>
            </tr>
            </thead>
            <tbody>
            <tr ng-repeat="t in tableData">
                <td>`{{t.name}}`</td>
            </tr>
            </tbody>
    </table>
</div>
......
```

### 5. 过滤器

过滤器在表达式中用 “|  过滤器” 表示

```
<div ng-app="myApp" ng-controller="personCtrl">
	<p>姓名为 `{{ lastName | uppercase }}`</p>
</div>
```

这里使用了AngularJS的默认过滤器uppercase，表示把字符串转换成大写。

也可以在指令中加过滤器：

```
<div ng-app="myApp" ng-controller="namesCtrl">
    <ul>
      <li ng-repeat="x in names | orderBy:'country'">
        `{{ x.name + ', ' + x.country }}`
      </li>
    </ul>
</div>
```

这里意思是列表按照country来排序。

我们也可以自定过滤器：

```
//金额积分千分位格式
.filter('AmountIntegralThousandsFormat', function () {
	return function (input) {
		if(typeof(input)=="undefined"){
			return;
		}
		return input.toString().replace(/\B(?=(\d{3})+(?!\d))/g, ",");
	};
});
```

```
<tr>
    <td>积分成本费</td>
    <td>`{{platformData.provisionsReceivables | AmountIntegralThousandsFormat}}`元</td>
</tr>
```

### 6. 自建服务



### 7. $Http



### 8. 路由



### 9. 依赖注入