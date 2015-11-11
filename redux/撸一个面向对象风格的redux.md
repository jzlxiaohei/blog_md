#撸一个面向对象风格的redux
###redux使用感受
各种优点就不说了，有两点觉得很不爽的地方：

1. 示例工程，文件夹太散。具体而言`action`,`reduce`,`store`,三者的对应关系很固定，分在三个文件夹下，实在太散。

2. `redux`的源码使用了大量的`函数式`的pattern。本人不是很习惯（好像暴露了-_——！）

所以,我就撸了一个面向对象风格的简易版，代码数少到不行，[代码在github上](https://github.com/jzlxiaohei/zlux)，（还附带了`react-redux`的`connect`的简易版，不过方面的代码注释掉了，因为这是为了配合react的功能，而redux的设计有独立于react的初衷。

在本人的一个demo项目中，使用下来，感觉还行。[demo地址](http://dapigu.wallstcn.com/react-test.html) 样式上主要适配手机，pc上也能看，另外控制台有列表页加载数据的log，如下图。demo的代码暂时不开放，需要整理。

![screen for console](console.png)






##store的简单实现（无中间件功能）
1.`action`，`reduce`必须要使用者提供.同`redux`一样，要求每次`reduce`返回全新的`state`对象

`dispatch`的功能相对固定，另外提供一个`subscribe`方法，用来注册监听器，`dispatch`时，通知所有监听者。

根据这些要求，写一个BaseStore。用户需要继承BaseStore，提供`reduce`函数

	export default class BaseStore{
    
        listeners=[]
    
        dispatch(action){
            this.state = this.reduce(action);
            this.listeners.forEach(listen=>{
                listen(action,this.getState())
            })
        }
    
        subscribe(listener){
            var index = this.listeners.length
            this.listeners.push(listener);
    
            return (index)=>{
                return ()=>{
                    this.listeners.splice(index);
                }
            }(index)
        }
    
    
        reduce(action){
            throw new Error('subClass fo BaseStore should implement reducer function')
        }
    
        getState(){
            return this.state;
        }
    
    }
    
    
使用的时候，只需继承`BaseStore`,提供`reduce`函数就行

	import Immutable from 'immutable';
    import {BaseStore} from 'zlux'
    
    const ActionTypes={
        ADD:'ADD',
        DELETE_BY_ID:'DELETE_BY_ID',
        UPDATE:'UPDATE'
    }
    
    export default class SimpleStore extends BaseStore{
        __className ='PostListStore'
    
        state = new Immutable.List()
    
        reduce(action){
            if(action.type === ActionTypes.ADD){
                return state.push(action.payLoad)
            }
            if(action.type === ActionTypes.DELETE_BY_ID){
                let id = action.payLoad.id;
                return state.filter(item=>{return item.id !==id})
            }
            if(action.type == ActionTypes.UPDATE){
                var id = action.payLoad.id;
                var index = state.findIndex(item=>{return item.id == id});
                //if index == -1, 这里不考虑update时，没有相应item的情况
                return state.set(index,action.payLoad)
            }
    
            return state; //注意：默认返回原state
        }
        
        //提供方便外接调用的方法
        add(payLoad){
            this.dispatch({
                type:ActionTypes.ADD,
                payLoad
            })
        }
        
        deleteById(payLoad){
            this.dispatch({
                type:ActionTypes.DELETE_BY_ID,
                payLoad
            })
        }
        
        update(payLoad){
            this.dispatch({
                type:ActionTypes.UPDATE,
                payLoad
            })
        }
    }

然后像这样使用：
	
   	var ss = new SimpleStore()
   	ss.add({id:1,content:'hello'})
   	ss.update({id:1,content:'world'})
   	ss.deleteById({id:1})

每次调用dispatch，`ss.getState()`都会返回最新的state。因为使用了`immutable`的list，确保每次的`state`都是全新的。

##中间件
关于redux的中间件的来龙去脉，官方文档已经说得不能在详细，[文档地址](http://rackt.github.io/redux/docs/advanced/Middleware.html)

这里实现中间件，故意做的和`express`中的用法很像。如下：

	simpleStore.use(
    	function(next,action,store){
      		console.log('before')
      		next()
      		console.log('after')
        }，
        function(next){
        	if(isLogin){
        		next()
        	}
        	else{
        		goToLogin();
        	}
        }
	)

稍微有些区别，

1. 需要用中间件，要一次性的传个`use`函数，多次使用`use`，后面的会覆盖前面的。

2. 中间件函数中，参数只有next是必须的。`action`和`store` 都是自动注入的。需要用就写上，不需要用，就不用管。
3. 暂时没使用error first的模式。个人认为`state`中完全可以体现错误信息。
	
中间件实现的核心代码如下：

	use(...fns){
        this.middlewareFns = fns;
        var _this =this;
        this.wrappedDispatch= this.middlewareFns.reduceRight((a,b)=>{
            return ()=>{
                b(a,_this.__curAction,_this)
            }
        },this.__dispatch)//__dispatch是原始的dispatch实现。
    }
    
    //改写上面的dispatch实现。
    dispatch(action){
        this.__curAction = action
        this.wrappedDispatch();
    }


#简易connect

`flux`，`redux`等的实现都是不依赖于react，只要合适，任何环境下都能使用。为了更好的和`react`配合使用，redux官方还提供了 `react-redux`.

`react-redux`里的`connect`，帮助`Redux store`（由`provider`传入的合并过的store）的中状态、方法和`container`进行绑定。

`react-redux`要求整个应用只有一个`redux store`,是由多个单纯store（使用`单纯store`来区分`redux store`）是合并而成。`container`可以对应多个单纯store。

使用者(也就是你)选取`Redux store`中的需要的`state`，`dispatch`, 交由`connect`去绑定到react组件的props中。
指定了哪些 `Store/State` 属性被映射到 `React Component` 的 `props`，这个过程被称为 `selector`.
如果非常在意性能，避免不必要的计算，还需要通过`reselect`这样的库。


而我这里要求单纯的store和container是一对多的关系。如果一个`container`需要多个store，那么通过拆分container，而不是合并store。
这样就要求`container`是最小的页面构成单位，应该做到原子化。
这样，container是否需要通过`setState()`来render组件，只要比较`对应`的`单纯store`的state，是不是同一个（还记得吗，任何的状态改变，都返回全新的state，所以这个判断非常快）
只所以这样要求，是因为好实现（找了那么多接口，最后暴露了）。当然我好实现，使用者就得多写点代码，但是结构上，我个人觉得更清晰点。

不过因为我还没写个特别大型的项目，不知道拆分`container`而不是合并store，是不是能满足复杂应用的开发。

    import {Component} from 'react';
    import getStoreShape from './getStoreShape.js'
    
    export default (WrappedContainer,store) => {
        return class extends Component{
    
            state={}
    
            constructor(props,context){
                super(props,context)
    
                this.state.props = store.getState();
                
                this.unsubscribe = store.subscribe(()=>{
                    //直接判断state的引用是否变化，shouldComponentUpdate都不需要了
                    if(this.state.props == store.getState()){return ;}
                    //这里的props随便取的名字，没有意义
                    //setState只是用于通知重新渲染。
                    this.setState({
                        props:store.getState()
                    })
                })
            }
    
            componentWillUnmount(){
                this.unsubscribe();
            }
    
            //context机制，TODO test case
            static childContextTypes={
                store:getStoreShape
            }
    
            getChildContext() {
                return { store: this.store };
            }
    
            getWrappedInstance() {
                return this.refs.wrappedInstance;
            }
    
            render(){
                return(
                    <WrappedContainer ref='wrappedInstance' {...this.props} store={store} />
                )
            }
        }
    }

整体是，就是一个 [high order component](https://github.com/gaearon/redux/blob/cdaa3e81ffdf49e25ce39eeed37affc8f0c590f7/docs/higher-order-stores.md)

在`constructor`里,注册了对`store`的监听。这是一个单纯store，直接比较state是否变化，就知道要不要从新render。

使用的话，贴点代码：

    //postListStore 是已经实例化的单纯store，可以被多个container公用。
    const PostListElement = enhanceWithStore(PostListContainer,postListStore)
    const PostDetailElement = enhanceWithStore(PostDetailContainer,postDetailStore)
    //...
    //使用react-router
    React.render(
        (
            <Router>
                <Route path="/" component={App}>
                    <IndexRoute  component={PostListElement}
                                 onEnter={()=>{ utils.Scroll.restoreScroll('PostList') }}
                                 onLeave={()=>{ utils.Scroll.saveScroll('PostList') }} />
    
                    <Route path="posts/:postId" component={PostDetailElement} />
                </Route>
            </Router>
        ),
        document.getElementById('mount-dom')
    )

#其他
核心的功能就这些。
真正serious的实际项目，大家还是用用`redux`吧，配套齐全。
自撸的项目，自己先踩踩坑~。