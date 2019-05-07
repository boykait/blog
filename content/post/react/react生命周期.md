#### props和state的关系和作用

render方法返回的结果并不是真正的DOM元素，而是一个虚拟的表现，类似于一个DOM tree的结构的对象。react之所以效率高，就是这个原因， 虚拟DOM在内存中是展开运用的，运行机制和原理？

render中如何改变state会导致什么问题呢？ 就是死循环

作用
constructor(props) {
    super(props)
}

父组件render导致子组件的从新渲染的优化方法
shouldComponentUpdate

个人觉得使用redux后就是将更多的保存在state中的数据交给props来保存使用redux就是讲

#### react组件类型
 &emsp;&emsp;我们需要知道一件事就是react的组件包括无状态和有状态的组件，无状态组件主要通过函数的形式来创建
 
#### react组件生命周期
&emsp;&emsp;一个组件的生命周期无非四个阶段：初始化-挂载-更新-销毁，每个阶段react都定义了相应的钩子函数。

##### 1. 创建阶段
&emsp;&emsp;创建阶段的工作主要是考虑如何去实例化一个组件对象，组件实例化阶段的所有操作都只会被执行一次，react组件实例化所包括的钩子函数如下：

    - getDefaultProps
    - getInitialState
    - componentWillMount
    - render
    - componentDidMount

```
import React, {Component} from 'react';
import { Input, Select, Icon } from 'antd';
const Option = Select.Option;
class App extends Component {
  constructor(props) {
    super(props);
    console.log('constructor is called');
    let {val} = this.props;
    console.log(val);
  }
  componentWillMount() {
    console.log('componentWillMount is called')
  }
  componentDidMount() {
    console.log('componentDidMount is called')
  }

  componentWillReceiveProps(nextProps, nextContext) {
    console.log('componentWillReceiveProps is called')
  }

  shouldComponentUpdate(nextProps, nextState, nextContext) {
    console.log('shouldComponentUpdate is called')
  }

  componentWillUpdate(nextProps, nextState, nextContext) {
    console.log('componentWillUpdate is called')
  }

  componentDidUpdate(prevProps, prevState, snapshot) {
    console.log('componentDidUpdate is called')
  }

  componentWillUnmount() {
    console.log('componentWillUnmount is called')
  }

  render() {
    console.log('render is called')
    return (
      <div>
      </div>
    );
  }
}

export default App;

```

&emsp;&emsp;ok，在上面的一段程序中展现了React16中的主要的四个阶段的钩子函数： 
##### 初始化阶段
&emsp;&emsp;`constructor`即使没有显示定义也会被默认定义，但如果显示地定义了constructor则必须显示的设置super方法，而且如果在接下来需要使用钩子方法上传人的props参数的话(通过this.props)，也同样需要将props作为实参传人super方法，因为在constructor函数中通过设置super才能让App类继承父类的this对象。    
&emsp;&emsp;在constructor方法中我们可以设置props以及state，props主要用于数据共享传递，比如可以在父组件在调用 ` <App name="app测试"> ` 将name属性传递给App组件，同样我们也能将方法作为属性传递给子组件，供子组件回调父组件的方法，props经常被用作渲染组件和初始化状态，当一个组件被实例化之后，它的props是只读的，不可改变的；state是react组件的内部数据状态，可以通过组件内部调用this.setState方法来进行数据的变更，修改state属性会导致组件重新渲染。 

##### 挂载阶段
&emsp;&emsp;`componentWillMount`可以在服务端被调用，也可以在浏览器端被调用，在浏览器端的这方法里的代码调用setState方法不会触发重渲染，所以它一般不会用来作加载数据之用，它也很少被使用到，当然也可以在里面去设置state。    
&emsp;&emsp;`render`内部返回一个JSX元素，主要生成页面需要的虚拟DOM结构，用来表示组件的输出。
&emsp;&emsp;`componentDidMount`该方法发生在render方法成功调用并且真实的DOM已经被渲染之后，一般在组件需要请求服务器数据来进行页面初始化的工作就在这个方法中进行。

#### 更新阶段
&emsp;&emsp;`componentWillReceiveProps`组件的 props 属性可以通过父组件来更改，会触发该方法，但父组件不保证传递的props属性是发生了变化的，这需要根据传递的属性`nextProps`和当前的`this.props`进行对比来确定是否需要进行下一步操作。    
&emsp;&emsp;`shouldComponentUpdate`，该方法会接收两个参数`nextProps`和 `nextState`，在这个阶段可以进一步通过判断当nexProps和this.props以及nextState和this.state的变动情况来确定是否需要继续进行render操作，以达到优化渲染的目的。    
&emsp;&emsp;`componentWillUpdate`，你已经拥有下一个属性和状态，它们可以在这个方法中任由你处 置。你可以利用这个方法在渲染之前进行最后的准备。注意在这个生命周期方法中你不能再触发 setState()。如果你想基于新的属性计算状态，你必须利用 componentWillReceiveProps方法来实现。    
&emsp;&emsp; `render` 同挂载阶段作用一样。    
&emsp;&emsp;`componentDidUpdate` 此方法在组件更新后被调用，可以操作组件更新的DOM，prevProps和prevState这两个参数指的是组件更新前的props和state 

##### 销毁阶段
&emsp;&emsp;`componentWillUnmount`，卸载组件，这个过程可以执行一些组件的清理工作，比如在组件内部使用了定时器，则可以在该方法中进行处理。

