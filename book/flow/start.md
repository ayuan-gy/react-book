# React是如何调度任务的

首先我们要知道哪些操作会让React开始一次任务调度

1. `ReactDOM.render`或者`ReactDOM.hydrate`
2. `this.setState`
3. `this.forceUpdate`
4. suspend component promise resolve or reject