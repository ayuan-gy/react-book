# 开启

魔改源码：把`enableSuspense`改成`true`

# 特性

1. 如果你throw了一个`promise`，这个组件会被`unmount`，是的是字面意思上的`unmount`，在resolve之后会被重新`mount`，注意对应的生命周期都会被调用
