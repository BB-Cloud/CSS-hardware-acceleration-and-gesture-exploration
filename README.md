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


> 如果硬件加速是输出，节流函数是辅助，那么最后一点的缓存节点就是坦克了吧

<h3>最后一点比较重要的——对节点进行缓存</h3>

为什么说它是坦克？因为输出和辅助都有了啊！一个队伍如果想最佳配备，肯定是需要坦克的。尤其是在大量手势和动画存在的本游戏中。
所以，在手势丰富，动画丰富的页面中，有了硬件加速和节流函数只是胜算较大，稳赢的话还需要对重复调用的节点进行缓存。
就比如这段滑动核心的部分

     this.$page
        .on('panstart', '.first', function (e) {
            if (typeof touchId !== 'undefined') {
                return
            }
            touchId = e.touchEvent.changedTouches[0].identifier;
            e.preventDefault() //阻止默认的滚动页面事件
            $this = $(this)
            $right = $this.find('.option-can')
            $left = $this.find('.option-not')
            $this.css({
                '-webkit-transition': 'none',
                'transition': 'none'
            })
            self.firstCurrentQueNum = $this.index()
            windowWidth = $(window).width()
            windowHeight = $(window).height()
        })
        .on('pan', '.first', throttle(function (e) {
            if (!self.btnEnable) {return}
            if (e.touchEvent.changedTouches.length > 1) {return}
            if (e.touchEvent.changedTouches[0].identifier !== touchId) {
                return
            }
            $this.css({
                'transform': 'translate3d(' + e.displacementX + 'px, ' + e.displacementY + 'px, 0)',
                '-webkit-transform': 'translate3d(' + e.displacementX + 'px, ' + e.displacementY + 'px, 0)',
            })
            let alpha = Math.abs(e.displacementX) / windowWidth * 4;
            if (alpha > 1) {
                alpha = 1
            }
            if (e.displacementX < 0) {
                $left.removeClass('hidden').css('opacity', alpha);
                $right.addClass('hidden');
            } else {
                $right.removeClass('hidden').css('opacity', alpha);
                $left.addClass('hidden');
            }
        }, 0, isIOS() ? 40 : 100))
        .on('panend', '.first', function (e) {
            if (!self.btnEnable) {return}
            if (e.touchEvent.changedTouches.length > 1) {return}
            if (e.touchEvent.changedTouches[0].identifier !== touchId) {
                return
            }
            self.btnEnable = false
            let x = 0, y = 0;
            //当它满足以下条件飞出去，不满足就不飞
            if (Math.abs(e.displacementX) > windowWidth / 7.5) {
                //偏移的距离测算
                if (e.displacementX / e.displacementY > windowWidth / windowHeight) {
                    // 偏x
                    x = windowWidth * 2;
                    y = Math.abs(e.displacementY / e.displacementX) * x
                } else {
                    // 偏y
                    y = windowHeight * 2;
                    x = Math.abs(e.displacementX / e.displacementY) * y
                }
                //偏移的方向测算
                if (e.displacementX < 0) {
                    if (e.displacementY < 0) {
                        // 左上
                        x = -x;
                        y = -y;
                    } else {
                        // 左下
                        x = -x;
                    }
                } else {
                    if (e.displacementY < 0) {
                        // 右上
                        y = -y;
                    } else {
                        // 右下
                    }
                }
                // 选择正误判断
                if (e.displacementX < 0) {
                    e.isLeft = true
                } else {
                    e.isLeft = false
                }
                self.options(e.isLeft, self.firstCurrentQueNum, result);
                //对偏移方向和距离测算后的飞离
                $this.css({
                    '-webkit-transition': 'all .7s ease-in',
                    'transition': 'all .7s ease-in',
                });
                setTimeout(() => {
                    $this.css({
                        'transform': 'translate3d(' + x + 'px, ' + y + 'px, 0)',
                        '-webkit-transform': 'translate3d(' + x + 'px, ' + y + 'px, 0)',
                    });
                }, 20);

                $this.on('webkitTransitionEnd transitionEnd', function(){
                    self.changeList();
                    setTimeout(() => {
                        $(this).removeClass('first').off()
                    }, 20)
                    self.btnEnable = true;
                    touchId = undefined;
                })

            } else {
                setTimeout(() => {
                    $this.css({
                        'transform': 'translate3d(0, 0, 0)',
                        '-webkit-transform': 'translate3d(0, 0, 0)',
                    });
                    $left.addClass('hidden');
                    $right.addClass('hidden')
                    self.btnEnable = true;
                    touchId = undefined
                }, 20)
            }
        })
  
在panstart部分，要把后续会进行获取节点的操作进行省略，提高性能，所以对节点在开始被挪动的时候进行缓存，防止每移动一点点就获取一次。这也是为了体验流畅，性能提升做了很大的贡献的一步。

(ps:在最后说明一点，panstart、pan、panend都是自行封装的手势库的参数，类似于touchstart、touchmove、touchend)