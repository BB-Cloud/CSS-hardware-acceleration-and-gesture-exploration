# CSS-hardware-acceleration-and-gesture-exploration
> 这是我制作一个手势答题小游戏的学习和体会 ~ <br/>
> 该游戏是运行在移动端（手机居多）的，所以在一些地方对安卓和IOS进行分开处理。

<h3>节流函数</h3>

      function throttle (fn, delay, mustRunDelay) {
          let timer = null;
          let starttime = +new Date();

          return function () {
              var context = this,
                  args = arguments,
                  curTime = +new Date();
              clearTimeout(timer);
              if (!starttime) {
                  starttime = curTime
              }
              if (curTime - starttime >= mustRunDelay) {
                  fn.apply(context, args)
                  starttime = curTime
              } else {
                  timer = setTimeout(function () {
                      fn.apply(context, args)
                  }, delay)
              }
          }
      }
  
这一段是为了隔一段时间去触发一次动作，防止手势持续滑动触发太多次连带移动的事件

调用如下，这里对安卓的间隔更长一些

    this.$page
        .on('pan', '.first', throttle(function (e) {
               //此处省略一些代码
            }, 0, isIOS() ? 40 : 100))


> 在第一遍代码成型时候，我拿着苹果手机开开心心玩游戏，结果拿来安卓机一测，顿时炸毛。卡顿、不能动、怎么也不能动……<br/>
> 于是我开始了漫长的修锅之旅~~~~ <br/>
> 终于找到你：CSS开启硬件加速

<h3>CSS开启硬件加速</h3>

    .a{
        transform: translate3d(1px, 2px, 2px); //示例
    }
    .b{
        transform: translateZ(0); //示例
    } 
    .a{
        will-change: transform;
        transform: translate3d(1px, 2px, 2px); //示例
    }
    
大概有这三种方式去处理，我自己总结为：3d加速、Z加速、预判加速。这里只是给了一个transform的例子，其他的3d同理，比如scale3d
不要小看这种处理方式，在安卓里的体验已经进了一大步了，可能稍微低端的安卓机还不能特别流畅，但是大多数已经可以正常运行了。
那么为什么IOS一直可以正常运行呢？可能苹果系统就是做了什么优化处理之类的，具体我没有探究过……

