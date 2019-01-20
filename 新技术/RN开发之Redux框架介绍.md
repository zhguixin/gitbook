Redux作为一个数据管理框架。集中管理各个组件所有的数据，View层根据数组值更高界面，Model层更高数据后存储到Redux框架中。

主要包括三个部分：

* action，由组件发出一个action更改数据
* store，存储数据
* reducer，是一个函数，对应接收initState/actionType，两个参数，返回值为：newState，该函数能不能放耗时操作或者异步操作

#### 具体使用

```react
// createStore 在 react-redux模块下
<Provider store={createStore(reducers)}>
	{routes}
</Provider>
```

创建一个store来存储数据，被他包含的子组件可以访问到store中的数据。

先来看看`reducers` 的实现。一个典型的`reducer` 是一个包含有switch/case的函数，接收两个参数：state和action，返回一个state。

```react
// reducer.js
const FETCH_MOVIES = 'movies/FETCH_MOVIES'

export default fetchMoviesActionCreator: (movies) => ({
    type: FETCH_MOVIES,
    movies
}) // 定义一个创建 action 的函数

const initialState = {
	movies: [],
	movie: {}
}

export default reducer(state = initialState, action) {
    switch(action.type) {
        case FETCH_MOVIES:
            return {
                ...state,
                all: action.movies
            }
        case FETCH_MOVIE:
            return {
                ...state,
                current: action.movie
            }            
    } // end switch
}
    
```

Movies作为一个子组件，被包含在`<Provider/>` 标签下。

```react
const {
	fetchMoviesActionCreator
} = require('reducer.js')

class Movies extends Component {
    componentWillMount() {
        // dispatch 这个 action 的时候，不传参就是 initState
        // 获取网络操作得到的响应数据可以传递进去，异步耗时操作可以放到此处
    	this.props.fetchMovies() // dispatch action
    }
    ...
}

module.exports = connect(state => ({
     // key值：movies，便于通过props.movies获得
     // value：在reducer中定义的 movies，其中all 在switch/case 中定义
     movies: state.movies.all 
}), {
     fetchMovies: fetchMoviesActionCreator // 重命名
})(Movies)
```

其中的`connect` 是一个函数，第一个参数将state（上面的例子只取走了movies）添加到组件的属性props中；第二个参数，将一个action添加到组件的属性中去。