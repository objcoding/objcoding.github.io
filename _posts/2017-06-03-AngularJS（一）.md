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

AngularJS的表达式在页面中是以双大括号**｛｛expression｝｝**来表示的，大括号里面可以填写文字，表达式，变量等。

***注：由于双大括号会被网页解释导致看不到，所以本文代码块中的AngularJS表达式都用中文双大括号｛｛｝｝表示。***

- 变量：

```html
<div>合计: ｛｛sum｝｝元</div>
```

sum变量是从$scope作用域拿的，后面会讲到。

- 表达式：

```html
 <p>我的第一个表达式: ｛｛5+5｝｝</p>
```

- 字符串：

```html
<div ng-app="" ng-init="firstName='John';lastName='Doe'">
<p>姓名： ｛｛firstName + " " + lastName｝｝</p>
</div>
```

### 2. 指令

AngularJS的指令可扩展html属性，AngularJS可通过指令与页面进行交互，且可以允许开发者自定义指令。AngularJS的指令都有ng-前缀，如下：

- ng-app：在页面上初始化一个AngularJS应用程序

```html
<div ng-app="">
     <p>你输入的为： ｛｛1+1｝｝</p>
</div>
```

如果ng-app=""，那么会使用AngularJS默认初始化一个应用程序，如果不为空，则需要自定义一个应用程序：

```javascript
angular.module('myApp', ['ngRoute'])
    .config(['$locationProvider', '$routeProvider', function ($locationProvider, $routeProvider) {
    ......
    }]).controller('AppCtrl', ['$scope','$http', function ($scope,$http) {
        ......
        }])
```

```html
<div ng-app="myApp">
     <p>你输入的为： ｛｛1+1｝｝</p>
</div>
```

- ng-model（模型）：可把作用域的值绑定到应用程序中，或者可以把元素值绑定到输入域中，这种绑定采用双向绑定机制，即任何一方改变，另一方也跟着改变。

元素绑定：

```html
<div ng-app="" ng-init="firstName='John'">
     <p>在输入框中尝试输入：</p>
     <p>姓名：<input type="text" ng-model="firstName"></p>
     <p>你输入的为： ｛｛firstName｝｝</p>
</div>
```

作用域绑定：

```javascript
angular.module('myApp')
  .controller('AppCtrl', ['$scope','$http', function ($scope,$http) {
        $scope.shop.name="zhangsan"
        }])
```

```HTML
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

模型的数值由控制器控制，因此控制器通常操作数据，把数据存到$scope作用域中。控制器有ng-controller指令定义。

```javascript
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

```html
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
                <td>｛｛t.name｝｝</td>
            </tr>
            </tbody>
    </table>
</div>
......
```

### 5. 过滤器

过滤器在表达式中用 “|  过滤器” 表示：

```html
<div ng-app="myApp" ng-controller="personCtrl">
	<p>姓名为｛｛lastName | uppercase｝｝</p>
</div>
```

这里使用了AngularJS的默认过滤器uppercase，表示把字符串转换成大写。

也可以在指令中加过滤器：

```html
<div ng-app="myApp" ng-controller="namesCtrl">
    <ul>
      <li ng-repeat="x in names | orderBy:'country'">
        ｛｛x.name + ',' + x.country｝｝
      </li>
    </ul>
</div>
```

这里意思是列表按照country来排序。

我们也可以自定过滤器：

```javascript
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

```html
<tr>
    <td>积分成本费</td>
    <td>｛｛platformData.provisionsReceivables | AmountIntegralThousandsFormat｝｝元</td>
</tr>
```

### 6. 服务

Angular自定义服务有很多个方法，每个方法又有区别，但他们都是＄provide的封装形式。而Angular本身也提供了很多非常优秀的服务，如＄Http，＄scope，＄routeParams，＄q等等，他们有个共同的特征，即前缀都是美元符号＄。

在写自定服务之前，先来讲讲Angular的MVC设计模式，这种模式思想与后台的MVC设计思想如出一致，据说Angular是一位后台开发者为了方便操作前端页面而发明的，所以Angular也参照了后台MVC的分层思想：

Dao层：在Angular中指的是model，主要用来写Ajax从后台获取数据；

Service层：在Angular中指的是服务，主要用来写前端业务，也可充当数据容器；

Controller层：在Angular中指的是控制器，主要用来处理数据，它最终的目的是为了向页面提供需要展示的数据，且不提倡在控制器中写过多的逻辑代码。

#### 6.1 value

Value是一个简单的值对象，用于向控制器传递值。

```javascript
var myApp = angular.module("mainApp", []);
myApp.value("defaultInput", 5);

myApp.controller('CalcController', ['$scope','defaultInput', function($scope, defaultInput) {
   $scope.number = defaultInput;
}]);
```

#### 6.2 service

用service创建服务，是一个构造函数，Angular会用new来创建一个对象，因此用service创建服务无需返回对象，里面属性通常用this表示：

```javascript
var app = angular.module('app' ,[]);
app.service('people', function () {
     this.name = 'zhangsan';
});
app.controller('myCtrl', ['people', function (people) {
 	$scope.name = people.name;
}]);
```

***注：要把people服务注入myCtrl控制器里，我们通常使用内联样式，即把服务的名字以数组元素的形式与函数组成一个数组，其中函数的传入参数即为数组元素，这样可以有效防止js文件以压缩形式传送过程中服务名字丢失的情况。***

#### 6.3 factory

factory也就是一个普通的Function函数，需要提供一个返回的值：

```javascript
var app = angular.module('app' ,[]);
app.factory('people', function () {
  	var service = {};
    service.name = "zhangsan";
    var age;
    service.setAge = function(newAge){
        age = newAge;
    }
    service.getAge = function(){
        return age; 
    }
    return service;
});
app.controller('myCtrl', ['people', function (people) {
  	people.setAge = 20;
  	$scope.age = people.getAge;
 	$scope.name = people.name;
}]);
```

#### 6.4 $provide

$provide叫提供商，而服务会通过提供商来定义，它是service和factory的老大哥，因为他们都是provide的高度封装形式，其实上面的写法是利用了Angular的语法糖，service和factory也可以通过如下来写：

```javascript
var app = angular.module('myApp', []);
app.config(['$provide', function($provide) {
  $provide.service('people', function() {
    this.name = "zhangsan";
  })
}]);
app.controller('myCtrl', ['$scope', 'people', function($scope, people) {
  $scope.name = people.name;
}])
```

```javascript
var app = angular.module('myApp', []);
app.config(['$provide', function($provide) {
  $provide.service('people', function() {
    var service = {};
    service.name = "zhangsan";
    var age;
    service.setAge = function(newAge){
        age = newAge;
    }
    service.getAge = function(){
        return age; 
    }
    return service;
  })
}]);
app.controller('myCtrl', ['$scope', 'people', function($scope, people) {
  	people.setAge = 20;
  	$scope.age = people.getAge;
 	$scope.name = people.name;
}])
```

$provide是唯一一个可以写在config函数中的服务，config是用于在Angular启动前的配置工作，如我们需要预先定义路由（下面会讲到）：

```javascript
angular.module('myApp', ['ngRoute'])
    .config(['$routeProvider', function ($routeProvider) {
        $routeProvider
            .when('/shop/shops', {
                templateUrl: '/shops/shops.html',
                controller: 'ShopShopsCtrl'
            })
}])
```

*注：这里需要注意再config函数中注入provide服务，参数名的格式必须是：服务名 + Provider。*

现在我们编写一个“原生”的服务：

```javascript
var app = angular.module('myApp', []);
app.provider('people', function() {
  this.$get = function() {
    var service = {};
    service.name = "zhangsan";
    var age;
    service.setAge = function(newAge){
        age = newAge;
    }
    service.getAge = function(){
        return age; 
    }
    return service;
  }
})
app.controller('myCtrl', ['$scope', 'people', function($scope, people) {
  	people.setAge = 20;
  	$scope.age = people.getAge;
 	$scope.name = people.name;
}])
```

*注：provider不可注入内置服务。*

有没有发现，它跟用factory创建people服务是很类似，但是factory创建看起来简洁许多了，这也是为什么说factory是provide的高度封装了，这里的$get函数用来返回provide的一个实例，且是必须的，否则会创建失败，为什么说这个get函数是必须的呢？

我们再来谈谈Angular的$injector注入器，它是负责创建provide创建的服务的实例，只要provide提供了一个带有参数的可注入的函数，我们就可以通过注入器获取这个服务的实例，比如我们现在要获取上面我们已经创建好的people服务：

```javascript
var people = $injector.get("people");
people.setAge = 20;
var age = people.getAge;
var name = people.name;
```

### 7. $http

$http可以说是Angular最为核心的服务之一了，它是Angular高度封装Ajax的一个服务，用于异步请求后台数据，可以说它就是angular的Dao层了，它的基本写法如下：

```javascript
// 简单的 GET 请求，可以改为 POST
$http({
	method: 'GET',
	url: '/someUrl'
}).then(function successCallback(response) {
		// 请求成功执行代码
	}, function errorCallback(response) {
		// 请求失败执行代码
});
```

截取一段在项目中的实例：

```javascript
function requestTableData() {
    if ($scope.searchParams.isSearching) {
      AlertService.alert("搜索进行中。。。");
      return;
    }
    $scope.searchParams.isSearching = true;
    $http({
      method: "GET",
      url: HostManageService._shopHost + "/api/shop/shops/"+$scope.searchParams.page,
      data: {
        shopCode: $scope.searchParams.shopCode,
        shopName: $scope.searchParams.shopName,
        industry: $scope.searchParams.industry,
        shopStatus: $scope.searchParams.status
      }
    }).then(function successCallback (response) {
      if (response.data.errcode == 0) {
        if (response.data.shop.data.length == 0) {
          AlertService.alert("暂时没有搜索数据,请重新搜索!");
          $scope.tableData = [];
        } else {
          $scope.tableData = response.data.shop.data;
        }
        $scope.searchParams.page = response.data.shop.page;
        $scope.searchParams.hasNext = response.data.shop.hasNext;
      } else {
        AlertService.alert(response.data.errmsg);
      }
      $scope.searchParams.isSearching = false;
    }, function errorCallback(response) {
      AlertService.alert("列表内容请求出错了!");
      $scope.searchParams.isSearching = false;// 开启重新搜索
    });
}
requestTableData();
```



### 8. 路由



