之前一个后台项目，因为vue没有像样的ui组件库，所以选择了react，ui库用的ant.design(这个赞一个)，使用redux做状态管理。虽然业务完成也算顺利，但是redux的很多东西，经常性的让我觉得，有必要那么绕吗，感觉给开发带来了很多不必要的麻烦。然后经常纠结，当初如果选vue，然后直接用bootstrap会不会写的更舒爽。

直到看到了redux作者（这里给Dan Abramov大神点32个赞，主动推荐自己成名项目的alternative）的一条推特 
>unhappy with redux? try mobx.

[mobx项目地址](https://github.com/mobxjs/mobx)

然后我尝试了一下，然后之后项目的选型，就再也没纠结过。react + mobx 给我的感觉，写着就像略麻烦的vue，麻烦的地方在于，多了几个`decorator`,完全可以忽略。

#mobx的理念

redux的源码很少，但是理念却非常足量，记得当时我看的云里雾里，读了几遍，但是完全不知道该怎么去写代码。。。mobx也有相应的一套理念，不过了解个几分钟，就可以上手用起来了。

###mobx的几个关键组成
1. state

	大致算是react的state的拓展：`state`驱动着你的应用，通常是特定的领域对象。比如todo任务的列表，或者ui相关的状态，比如选中的元素，loading状态等。这些`state`是被观察的本源，其他的东西，都是从这些`state`派生出来的。
	
2. action

	一段改变state的代码。这个实在没啥好解释的。
	
3. Derivations(派生物)	
	
	从`state`派生出来的东西.包括
	
	a)派生的属性，比如尚未完成的todo任务列表，使用 todo 列表派生。这些派生属性，也可以作为其他属性的依赖，这样一直派生下去。
	
	b)带有副作用的反应（reactions），比如todo列表变动后，在console里输入些东西，ui上的变动。
	
	如果Derivations所依赖的state一改变，Derivations也要相应的改变或者触发一些副作用。
	
相对应，mobx提供了几个重要的`decorator`和重要方法.（api当然不止这些，核心是这几个，还有很多优化类，补强功能类，和规范代码类的api）

1. `@observable`

	在你的`state`（通常是类的实例属性）前，加上这个就可以了，这个字段会成为`被观察的本源`。比如
	
  ```javascript	
  		@observable price:number = 0;
      	@observable amount:number = 1;
  ```
	
2. `@computed`

	派生出来的属性。

  ```javascript
          @computed get total(){
              return this.price * this.amount;
          }
  ```        
    
	如果price或amount任意一个发生改变，total也会相应改变。
	
3. `autorun	`

	派生副作用

```javascript	
		autorun(()=>{console.log(total.get())})
```
		
这个匿名函数，依赖于total产生io的副作用。只要total变化，就会打印total的值。这里实际上形成了一个派生链，`（price,amount）=> total => console.log(total.get())`
	
对于ui来说，`ui = f(state,props)`,把f看成是render函数，那么

```javascript	
		autorun(()=>{render(state,props)}
```		

	即可以更新view。	autorun里函数的执行，也是根据state的变化，自动触发的。
	
根据这几个基本组成，是否能大致想象自动更新view的流程？	
		
#和react结合

基本原理就是 autorun依赖于state,state变化后，自动触发react的re-render。react的mobx binding，官方已经有了，`mobx-react`，只需要引人其中的`@observer`，修饰一下react的组件，这个组件就拥有了自动更新的能力，而且`mobx-react`做了很精细的控制，只有react组件依赖的属性变了，才会触发组件的重绘。也就是说，不用使用者怎么操心，这套方案的性能就很优秀。而使用`redux`的化，还要配合`reselect`，多写很多代码去达到这样的效果，甚至做不到。比如下面的代码

  ```javascript
  
  const profileView = observer(props => {
    if (props.person.nickName)
      return <div>{props.person.nickName}</div>
    else
      return <div>{props.person.fullName}</div>
  });
  
  ```
如果nickName不为空，那么mobx知道，这个view只依赖于nickName，fullName的变动不会引起view的变化，从而不会从新render组件。

使用mobx开发的流程，根据领域模型，写model.这个model就是最基本的es6 class加些decorator,像下面这样：

```javascript
	
	import {observable} from "mobx";

	class OrderLine {
    	@observable price = 0;
    	@observable amount = 1;

    	constructor(price) {
        	this.price = price;
    	}

    	@computed get total() {
        	return this.price * this.amount;
    	}
    	
    	@action setPrice(price){
    		this.price = 10;
    	}
	}
	...
	export default OrderLine;

```

暂时不考虑`@observable`等decorator的作用，这种类，对于已经使用es6工作的同学，没写过上千，也写过几百个吧。而且这东西，对于后端同学来说，也很熟悉（因为后台项目实在太多，我们的有些后台界面，是后端同学维护的），甚至可以根据`api`,自动生成大部分的东西。测试一个model，也是一件很容易的事。

然后写react组件

``` javascript

	import React, {Component} from 'react';
	import {observer} from "mobx-react";
	import OrderLine from './model/OrderLine'

	@observer
	class OrderLineView extends Component {
		constructor(){
			super();
			this.orderLine = new OrdereLine();
			//模拟属性的变化
			setInterval(()=>{
				this.orderLine.setPrice(Math.random()*20)
			},1000)
		}
    	render() {
        	return (
        		<div>
            		{this.orderLine.price} * {this.orderLine.amount} = {this.orderLine.total}
            	</div>
            )
    	}
	}
	
```	

这里的react组件，只是加了`@observer`,其他都是基本写法。在state管理上，组件的state，是上面的定义的class OrderLine的实例对象，可以通过父组件传入，也可以自己管理。然后OrderLine被观察的属性一变动，界面马上就相应的变动.

这种管理状态的方式，和redux的全局单个state tree的模式很不一样。这种更灵活些，可以`new`多个实例，各个view有自己的state;如果是应用全局的state，这个state可以被多个view依赖，使用者做好全局单例的处理即可。本质上就是对model的处理，想怎么处理，完全取决于使用者。

#其他的api
`mobx`还提供了其他一些api，相对高级点，比如
1. `untrack`：在某一时刻使用了state的某个属性，但是不想对这个state属性产生依赖
2. `transaction`:在`transaction`块中执行的属性修改，只会在块结束时，触发一次`Derivations`的变更或执行。这就避免了不必要的多次副作用，比如多次 render react组件。
3. `useStrict`： 非严格模式下的mobx，任何地方都可以修改state，这样很快就会让state的管理难以维护。严格模式下，只有标记了`@action`或在`runInAction`中的代码，才能修改state。这个强烈建议使用
4. spy & intercept 做单个state或全局所有state的拦截。这给log等功能，提供了很好的便利。

还有一些，大家去读文档吧。

#mobx的一些优缺点

有了大致了解后，我说一下我感觉到mobx的优点和缺点：

优点：
1.简单直观。

   首先领域模型处理这块，和已有的编程习惯非常符合。而且领域模型相关的东西非常集中，有哪些属性，action一目了然，和后端api可以对应的很好。对模型的异步修改，只要从服务器拉来数据后，简单的赋值就可以了，非常直观。

   redux中的领域模型，分散在action，reducer中，异步请求要依赖中间件。所以领域模型到底是什么样，需要在多个文件中切来切去，才能知道全貌，增加记忆压力，增加切文件的时间。

   相应的，redux中会有大量的样板代码，而mobx几乎没有。

2.性能

 mobx按照你的感觉写，自然就会有不错的性能。

3.state管理更加灵活

redux的`Single source of truth`虽然说着很美好，但是实际操作起来，有些场景很别扭。

比如用户设置地址时的省市联动功能，选择省以后，要触发`fetchCityByProvince`,选择城市后，要触发`fetchCountryByCity`,然后还有两个相应的reducer。现在页面有三处需要用户填写地址的地方，这就需要3套完全一样的 `action`，`reducer`。虽然可以适当的包装，重用大部分的action和reducer逻辑，但是使用上非常不爽。而使用mobx的话，new 3个AddressModal的class实例，搞定。

不过使用mobx，你需要根据情况管理state，是多个实例，还是全局唯一的实例，要求会高一些。不过管理模型，对工程师来说是必备技能。


缺点：
1.社区很不够成熟。社区这块没法和redux比，包括各种库和示例教程。不过mobx和已有的编程习惯匹配，比如之前怎么用jquery处理异步的，直接就能用到mobx上。

2.经过`@observable`包装过的属性，已经不是js的原生属性了，可能会在一些数据处理方面和调试上带来不便。不过mobx支持一些常用的原生处理方法，比如包装后的array，也支持map,filter能方法。另外`有toJS`方法，可以转成原生的js对象或数组。

3.大型项目
大家都在说redux适合大型项目，本人写的项目，最大也就十几个页面左右的spa；有些页面录入数据很复杂，单个页面就有几十个组件。
所以在我的认知范围内，我是没觉得有什么项目是用redux会比mobx爽的。另外代码量和维护难度虽不是完全正比，但是起码是正相关关系。
就像之前大家都说java适合写大型项目，不知道是真有体验，还是就随口说说。不过大家都那么说，我又没有实际体验，暂时承认吧。。。


#体验mobx
环境还是有些难配的，如果使用`decorator`,还需要环境支持es7。想体验的同学，可以直接使用[starter](https://github.com/mobxjs/mobx-react-boilerplate).祝体验愉快。
