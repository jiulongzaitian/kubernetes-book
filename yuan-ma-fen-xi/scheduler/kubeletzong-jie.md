## 总结

```
作者：李昂 邮箱：liang@haihangyun.com
```

kubelet可以说是一个典型的由事件驱动的异步架构。通过kubelet我们再次审视了kubernetes的设计理念，声明式设计保证kubelet总是去让系统向用户所期待的状态所演进。最后我们总结一下kubernetes三个重要的特性：

* 声明式\(Declarative\)：总是定义用户设定期望的状态。
* 水平触发\(Level-triggered\)：当事件触发后会产生通知，而在之后任意时间都会检测到这个被触发的事件。
* 异步\(asynchronous\)：由多个异步循环控制整个系统，使系统状态总是像用户期待的转变。



