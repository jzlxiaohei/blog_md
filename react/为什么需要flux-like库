我觉得应该有不少同学上来学React,觉得甚是惊艳，看着看着，发现facebook 安利了一个`flux`，图画的巨复杂,然后各种例子都有用这个东西。然后突然大家又都推荐`Redux`,号称最简单的`flux-like`的实现，结果实现的源码是很简单，但是文档是源码的几十倍，概念甩一脸，写个简单东西，要建十几个文件，写得云里雾里。

有没有想到，为什么要用Flux这类东西？

#React的数据流向
React只是一个view层的解决方案，光有界面没用，还得加上数据才行。React通过`props`和`state`去渲染界面，所以有个很形象的描述`UI = f(props,state)`.

有数据，就有数据通信的问题。react是单向数据流，父组件通过`props`把数据传递给子组件。但是数据的流向不可能只有这一种。


1. 祖父组件到孙子组件。
	这个看上去只是父到子的衍生,但是祖祖祖父到孙子组件呢。这个react在（好像是）0.13时通过`context`基本解决了，关于context，之前没接触的同学，可以看[文档](https://facebook.github.io/react/docs/context.html)

2. 子到父。下面是一种方案：父给子传递一个函数，子在只要调用这个函数，父就能得到相关的数据。
3. 非父子关系：基本可以叫做是兄弟关系，以网页为例，总归有一个共同的祖先`<body>`，但是有可能是非常非常远的兄弟。这个怎么处理。

组件的一些关系和相应的通信方式，官方有简单的说明，见[文档](https://facebook.github.io/react/tips/communicate-between-components.html)

对于上面的`2`，`3`两点,用react本事的机制，写出来都很别扭，特别是第3点。两个不相关的地方，要数据通信，最简单就是一个`全局变量`吗。当然光有全局变量还不行，你改了全局变量，其他所有对这个变量感兴趣React组件的都要被`通知到`,这样才能相应改变界面。你如果接触到设计模式，应该能想到`观察者模式`。

其实官方文档中，也有些小线索。


>For communication between two components that don't have a parent-child relationship, you can set up your own global event system. Subscribe to events in componentDidMount(), unsubscribe in componentWillUnmount(), and call setState() when you receive an event. Flux pattern is one of the possible ways to arrange this.

这段的关键是`you can set up your own global event system`。所以你只要去研究以前各种事件系统是怎么设计，就可以自己撸一套 `**ux`了。

# **UX的关键构成

梳理了一下，React需要配合`Flux-Like`的库去使用，是因为要解决通信问题。通信的关键有一下几点：

1. 数据，这个不用说的吧。因为react中，通过setState去触发改变界面，命名成`state`
2. 各种事件，叫`event`可能更只管，但是大家都叫它`action`,一个意思，发生了一个动作。这个动作有一下几个关系属性：什么动作，谁发出的（这个我看各个类flux库中，好像都没处理），有没有额外信息吗。
3. 事件发生，要分发出去，包括改变数据，然后通知给所有监听数据变化的Listeners
4. 注册监听者。

这些都是大家能想象到。好了，根据这几点，一个简单的`Myux`就可以写了

	class Myux{
		state:{},
		actionTypes:{},
		dispatch(){},
		subscribe(){}
		listeners:[]
	}
	
与上面四个组成部分项对应。来个例子吧，以计数器为例吧。

	
	class CountStore{
		static actionTypes:{
			UP:'UP', //你英语好，你用increase
			DOWN:"DOWN"
		}
		
		state:0 //数据，计数开始是零
		listeners:[]
		
		dispatch(actionType){
			if(actionType === CountStore.actionTypes.UP){
				this.state++;
			}
			if(actionType === CountStore.actionTypes.DOWN){
				this.state--;
			}
			this.listeners.forEach((ln)=>{
				ln(actionType,this,undefined)//对应什么动作，谁发出的，额外信息。
			})
		}
		
		subscribe(ln){
			this.listeners.push(ln)
			//返回一个函数，调用，就取消注册
			return ()=>{
				const index = this.listeners.indexOf(ln);
				if(index !== -1){
					this.listeners.splice(index,1)
				}
			}
		}
	}
	
react的组件里，只要注册成listener，然后state发生变化，被通知到，调用`setState`进行视图更新就好。

	class CountComponent extends React.Component{
		constructor(props,context){
        	super(props,context)
        	const store = this.props.store;
        	this.state = store.getState();
        	
    	}
		
		componentDidMount(){
			this.unsubscribe = store.subscribe((actionType,store)=>{
        		if(this.state !== store.getState()){
        			this.setState(store.getState());
        		}
        	})
		}
		
		componentWillUnmount(){
			if(typeof this.unsubscribe === 'function'){
				this.unsubscribe();
			}
		}
		
		
		render(){
			const state = this.state
			return <div>{state}</div>
		}
	}
	
使用吗,直接mount到body上，会报warning，忽略...

	const countStore = new CountStore()
	ReactDOM.render(
		<CountComponent store={countStore}/>,
		document.body
	)
	
这样，只有在任何地方，`countStore.dispatch(upOrDown)`,`CountComponent`里的数字就会加加减减。
可以想想一下，如果页面有2，3个组件要根据计数器的数值，做界面的相应变化，都是可以轻松满足的。

当然，如果只有一个组件用需要这个store，那么单纯代码上，这样写，要多写很多东西。但是谁知道以后页面不会加一个要公用这个store的组件,这时候这个`store`就是组件间通信的法宝了。

##实际情况要复杂

上面只有一个store，这是store还只有一个state,这个太简单了。实际，你的应用可能要维护多个状态。怎么办

1. 一个store里一个state，然后多个store
	
		ListStore => ListState
		DetailStore => DetailState
	
2. 全局就一个store，state是一个状态树，整个应用需要的state，都在这个树里。

		GlobalStore => state:{list:[],detail:{}} //...
		
还有一个问题`dispatch`,是全局一个`dispatch`,还是每个store一个`dispatch`。

这些分歧，加上函数式等，就导致了有`flux`,`reflux`,`redux`。。

还有各个事件之间，有可能存在依赖关系，A事件后，B也触发。又要加`waitFor`和`中间件`等概念。不过整体来说，就这些东西。

#各种库特点大串烧：
##redux的特点
redux的文档里，有三大原则，有了上面的概念，我们来对照看一下

1. Single source of truth

	就是更改应用一个state tree，储存在一个store里，这种情况，也只能有一个`dispatch`

2. State is read-only

	state是全局的，不是自读的，很维护。这个上面没有体现，但是也是很自然的想法。

3. Mutations are written as pure functions

	这个算`redux`最大的特点，引入了`reducers`的概念，和第二点有相辅相成的感觉。

另外还有中间件系统。

##flux的特点
单dispatch,多store多state，用`waitFor`处理store的依赖。

##reflux
多dispatch,多store多state。

#后记
事件系统，just it。



