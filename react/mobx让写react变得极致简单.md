之前一个后台项目，因为vue没有像样的ui组件库，所以选择了react，ui库用的ant.design(这个赞一个)，使用redux做状态管理。虽然业务完成也算顺利，但是redux的很多东西，经常性的让我觉得，有必要那么绕吗，感觉给开发带来了很多不必要的麻烦。然后经常纠结，当初如果选vue，然后直接用bootstrap会不会写的更舒爽。

直到看到了redux作者（这里给Dan Abramov大神点32个赞，主动推荐自己成名项目的alternative）的一条推特 
>unhappy with redux? try mobx.

[mobx项目地址](https://github.com/mobxjs/mobx)

然后我尝试了一下，然后之后项目的选型，就再也没纠结过。react + mobx 给我的感觉，写着就像略麻烦的vue，麻烦的地方在于，多了几个`decorator`,完全可以忽略。

如果你熟悉OOP，而且没有因为学习FRP，就觉得OOP一文不值，那么我强烈安利你mobx。如果你已经熟悉OOP，那么上手mobx是很轻松的事，上手后的代码编写更是熟悉加自然。如果你是函数式大神，请轻喷，我只是想提高开发效率早点下班，不想深究谁的理念更先进~~


#mobx的理念

redux的源码很少，但是理念却非常足量，记得当时我看文档，看得云里雾里，读了几遍，但是完全不知道该怎么去写代码，去读了两天源码，代码很少，很快读完，但还是不知道怎么写。。。mobx也有相应的一套理念，不过了解个几分钟，就可以上手用起来了。

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
  @observable price = 0;
  @observable amount = 1;
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

使用mobx开发的流程：

步奏一：根据领域模型，写model.这个model就是最基本的es6 class加些decorator,像下面这样：

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

对于一般项目，领域模型建模是理解业务的基石，上面这种写法，一眼就能对领域模型有大致的了解。redux的reducer里，虽然也有领域模型的所有信息，但是噪声太多，还有一些相关代码在`action`里。一个领域模型被拆了以后，又引出文件组织的问题，到底是所有action发一个文件夹下，还是根据业务模型，把action和相对应reducer放一起。

reducer的理念好，`nextState = f(curState,action)`,整个状态的流转都包含在里面，但是对于一般业务，我的感觉是，完全清楚状态的流转，是不必要的。感觉很像以前做物理题，根据能量守恒，知道初始状态和最终状态可以方便了处理很多问题，试图弄清中间过程的所有状态，没有必要，甚至会提高出错的概率.而写mobx，基本只要关注实例对象的属性变化，这是一般工程师熟悉的处理方式。

步奏二：写react组件

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

这种管理状态的方式，和redux的全局单个state tree的模式很不一样。这种更灵活些，可以`new`多个实例，各个view有自己的state;如果是应用全局的state，这个state可以被多个view依赖，使用者做好全局单例的处理即可。本质上就是对model的处理，想怎么处理，完全取决于使用者。也可以接着依赖注入的工具，比如`InversifyJS`去做这方面的处理。你能把`已有的编程经验`，很好的使用在这套开发流程中。

从前端开发的角度，每个页面相对独立，自己管理自己的状态，是大家都能hold住得一种开发方式。我接手一个项目，基本是以线上每个页面为出发点，对项目进行熟悉；平时维护，产品的需求也xx页面要加什么东西，改什么东西。页面相对独立好维护，改动以后出问题的可能性也小。如果一个spa中，有多个状态是要多个页面共享，会让人感觉乱乱的，这种情况用redux，会清晰很好。但是很多状态不需要放到全局，用redux你会把很多不需要多个页面共享的状态，也放到全局。

#其他的api
`mobx`还提供了其他一些api，相对高级点，比如

1. `untrack`：在某一时刻使用了state的某个属性，但是不想对这个state属性产生依赖

2. `transaction`:在`transaction`块中执行的属性修改，只会在块结束时，触发一次`Derivations`的变更或执行。这就避免了不必要的多次副作用，比如多次 render react组件。

3. `useStrict`： 非严格模式下的mobx，任何地方都可以修改state，这样很快就会让state的管理难以维护。严格模式下，只有标记了`@action`或在`runInAction`中的代码，才能修改state。这个强烈建议使用

4. spy & intercept 做单个state或全局所有state的拦截。这给log等功能，提供了很好的便利。

还有一些，大家去读文档吧。

#mobx的一些优缺点

有了大致了解后，我说一下我感觉到mobx的优点和缺点：

###优点：

1.简单直观。

   首先领域模型处理这块，和已有的编程习惯非常符合。而且领域模型相关的东西非常集中，有哪些属性，action一目了然，和后端api可以对应的很好。对模型的异步修改，只要从服务器拉来数据后，简单的赋值就可以了，非常直观。

   redux中的领域模型，分散在action，reducer中，异步请求要依赖中间件。经常需要在多个文件中切来切去，增加记忆压力，增加开发时间。

   相应的，redux中会有大量的样板代码，而mobx几乎没有。

2.性能

 mobx按照你的感觉写，自然就会有不错的性能。这点，我看的各种资料，已经twitter上大家的讨论，基本是共识

3.state管理更加灵活

redux的`Single source of truth`虽然说着很美好，但是实际操作起来，有些场景很别扭。

比如用户设置地址时的省市联动功能，选择省以后，要触发`fetchCityByProvince`,选择城市后，要触发`fetchCountryByCity`,然后还有两个相应的reducer。现在页面有三处需要用户填写地址的地方，这就需要3套完全一样的 `action`，`reducer`。虽然可以适当的包装，重用大部分的action和reducer逻辑，但是总归要起三个`action.type`的名字，三个reducer生成的`state`的名字。而使用mobx的话，new 3个AddressModal的class实例即可。

我比较能接收的方式还是 最小化全局的状态，页面再自己维护属于自己的状态。使用mobx，全局的状态可以使用全局单例，各自独立的状态，各个页面，各个组件自己去new 自己的实例，自己管理。


###缺点：

1.社区很不够成熟。社区这块没法和redux比，包括各种库和示例教程。不过mobx和已有的编程习惯匹配，比如之前怎么用jquery处理异步的，直接就能用到mobx上。我安利的两个同学，都反映官方的get start（英文）看着有点蛋疼。然后我说，想想你们写redux的第一个demo，然后他们表示刚才的疼不算什么~

2.经过`@observable`包装过的属性，已经不是js的原生属性了，可能会在一些数据处理方面和调试上带来不便。不过mobx支持一些常用的原生处理方法，比如包装后的array，也支持map,filter能方法。另外`有toJS`方法，可以转成原生的js对象或数组。

3.大型项目
大家都在说redux适合大型项目，这个我反驳不了。

本人写的项目，最大也就十几个页面左右的spa；有些页面录入数据很复杂，单个页面有几十个组件。
所以在我的认知范围内，我是没觉得有什么项目是用redux会比mobx爽的。另外代码量和维护难度虽不是完全正比，但是起码是正相关关系。另外领域模型的清晰程度来说，我也认为mobx完胜


#体验mobx
环境还是有些难配的，如果使用`decorator`,还需要环境支持es7。想体验的同学，可以直接使用[starter](https://github.com/mobxjs/mobx-react-boilerplate).祝体验愉快。
