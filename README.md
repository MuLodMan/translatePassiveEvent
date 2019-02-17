被动事件监听:  
    被动事件监听是在[Dom规范中](https://dom.spec.whatwg.org/#dom-addeventlisteneroptions-passive)的一种新功能,这种功能通过消除滚动时的阻塞来帮助开发者提高滚动性(译者注：后面会详细说明为什么滚动时将会阻塞)。开发者可以通过给触摸和滑轮监听器传递{passive:true}。参数的方式来表示开发者不会在监听的回调函数中调用preventDefault函数。这个功能在Chrome51、FireFox49和WebKit中均已经实现。查看下面的[视频](https://www.chromestatus.com/features/5745543795965952)来观察两种被动事件监听器的表现。
####问题:

&nbsp;&nbsp;&nbsp;&nbsp;平滑的滚动表现在良好的web用户体验中起着十分重要的作用，尤其在触摸设备中。当JavaScript运行时,所有的现代浏览器通过一个专门的线程来保证滚动的流畅。但是这种优化（负责处理滚动的线程）又会部分的被touchstart 和touchmove这些监听函数所阻塞。这是因为可能在事件的处理函数中通过调用preventDefault()来阻止滚动。然而在某些场景中开发者的确想要阻止滚动,分析表明绝大多数的事件处理函数并不会真正的调用preventdefault()函数，这导致浏览器在滑动过程中受到不必要的阻塞。举个例子,在Android的chrome中80%的触摸事件会导致滑动阻塞，但其实（开发者）并不想让其真正的阻塞。当滑动开始时，这些事件中的10%将多延时100ms，更加灾难性的是其中的一些滑动事件将会延时500ms。
&nbsp;&nbsp;&nbsp;&nbsp;许多开发者了解到[在文档元素中传入空函数](https://rbyers.github.io/janky-touch-scroll.html)依然会在滑动中造成十分大的负面影响。因此开发者理所当然的希望事件处理行为[不应该有任何的副作用](https://dom.spec.whatwg.org/#observing-event-listeners)。
&nbsp;&nbsp;&nbsp;&nbsp;这里的基本问题是触摸事件可以是无限的，滚动事件也有着同样的问题。相对的,指针事件处理函数被设计成不会阻塞滚动（尽管开发者可以通过声明touch-action css属性来阻止滚动)，所以不必担心这个问题。实质上被动事件监听提案将指针事件性能属性带到了触摸和齿轮事件。
&nbsp;&nbsp;&nbsp;&nbsp;这个提案为作者提供了一个方式,在注册处理函数时它可以表明当前处理函数是否会调用该事件的preventDefault方法(例如,该事件是否可以被取消)。当没有触摸事件或者滑轮事件在某点请求取消事件时,"用户代理"就可以立即开始自由滑动而不用等待JavaScript的执行。因此，被动事件会免于受到十分严重的性能副作用的干扰。
 事件监听选项:
 &nbsp;&nbsp;&nbsp;&nbsp;首先，我们需要向事件监听器中添加一个机制。目前监听器的"捕获参数"是一个十分接近(该机制用法)的例子，但它的使用语义不明确:document.addEventListener('touchstart',handler,true)。[EventListenerOptions](https://dom.spec.whatwg.org/#dictdef-eventlisteneroptions)让我们可以更明确的使用它。对于现存的行为而言这是一个新(扩展)的属性。参考[你是否想在事件捕获和冒泡的过程中调用监听器](http://javascript.info/bubbling-and-capturing#capturing)。
###方案"被动"选项:
&nbsp;&nbsp;&nbsp;&nbsp;现在我们在函数注册时有了扩展的语法，我们可以通过添加新的passive选项来提前声明监听器不会调用preventDefault函数。如果此时监听器调用该函数（preventDefault)，那么用户代理则忽略该请求，就如同该事件的Event.cancelable=false一样。开发者可以在调用preventdefault之前和之后通过判断preventDefault的值来验证。例如：
```
   addEventListener(document, "touchstart", function(e) {
    console.log(e.defaultPrevented);  // 值为false
    e.preventDefault();   // 不会起任何作用因为监听器是"被动的"
    console.log(e.defaultPrevented);  // 依然为false
  }, Modernizr.passiveeventlisteners ? {passive: true} : false);
```
  现在当不存在监听时浏览器可以只做滚动而不会被触摸或者滑轮监听所阻塞。被动监听将不会受到性能副作用的影响。因此浏览器将会立刻响应滑动而不会被JavaScript阻塞，这可以保证用户会获得流畅的滑动体验。

  特征检测:
  因为老的浏览器会将第三个参数解释成capture为true,那么对于开发者而言使用特征检测或者使用描述在避免不可预见的结果中起了十分重要的作用。特征检测可以按照如下方式去做:
  ```
 // 通过在Option对象中设置getter来判断被动选项是否会被获取
var supportsPassive = false;
try {
  var opts = Object.defineProperty({}, 'passive', {
    get: function() {
      supportsPassive = true;
    }
  });
  window.addEventListener("testPassive", null, opts);
  window.removeEventListener("testPassive", null, opts);
} catch (e) {}
// 使用我们检测的结果,如果浏览器支持则passive则会传入,无论浏览器支持不支持capture总是false
elem.addEventListener('touchstart', fn, supportsPassive ? { passive: true } : false);
```
为了更方便的使用,你可以使用[detect it](https://github.com/rafrex/detect-it),例如:
```
 elem.addEventListener('touchstart', fn,
 detectIt.passiveEvents ? {passive:true} : false);
```
####不必取消事件:
  在某些场景下，当用户触摸或者使用滚轮时开发者开发者会主动禁止滚动，这些场景包括：
  - 对地图进行移动或者变焦。
  - 全页面或者全屏游戏。

&nbsp;&nbsp;&nbsp;&nbsp;在这些情况下,目前的的行为(阻止滑动优化)完全可以胜任，因为滚动自身可以被禁止。这种情况没有必要使用被动监听,尽管使用touch-action:'none'css属性来表明意图是一个不错的主意。
  然而,在几个通用的场景中，并不需要阻塞滚动-例如:
  - 在用户行为管理中只需要知道用户最后的操作时间
  - touchstart事件处理器隐藏一些激活的UI（像是工具帖）
  - touchstart和touchend事件处理器改变UI样式（不需要制止点击事件）
  
&nbsp;&nbsp;&nbsp;&nbsp;对于这些场景,就可以使用‘被动‘选项（需要特征检测）而不需要其他的代码变动,会使滑动体验得到巨大改善

  这里有些更加复杂的场景，在这些场景中希望滚动在某些条件下受到抑制。例如：
  - 横向滑动轮换器使其滚动，释放一个条目或者显示一个抽屉,当然只是滑动也可以适用。
- 在这种情况下通过使用 touch-action: pan-y 声明来显式的禁止纵向滑动而不必使用preventDefault(测试页面)[https://rbyers.github.io/touch-action.html]。
- 为了使其在浏览器中能保持正常工作,在部分不支持touch-action的情况下应该调用preventDefault。

-  一个UI元素(像是YouTube的声音调节器)可以通过滚轮来横向滑动而不用改变纵向滑动事件的行为。因为这里没有和“touch-action”等价的滑轮事件,这种情况只能通过"非被动"滑轮事件处理。
- 在监听器的函数中使用事件代理模式是没有办法预判出用户是否会取消事件。
 - 其中一个选项是对被动事件和非被动事件使用不同的代理(就像它们的完全是不同的事件类型)。
 -  利用上面描述touch-action也是可以的(将触摸事件当成指针事件。

####调试和测量效益:

&nbsp;&nbsp;&nbsp;&nbsp;你应该迫不及待的想通过chrome看看将事件监听器作为“被动”时有什么可能的改善(和潜在性的破坏)。你通过[这个视频](https://twitter.com/RickByers/status/719736672523407360)自己可以很容易同时比较出来。
&nbsp;&nbsp;&nbsp;&nbsp;通过观看[此视频](https://www.youtube.com/watch?v=6-D_3yx_KVI)可以得到一些关于使用Chrome开发者工具来识别出那些监听器会阻塞滚动的建议。(译者：后面都是工具介绍，有的工具网址都无法访问,就不翻译了)。

####减少和分离长时间执行的JavaScript代码依然十分重要:

 &nbsp;&nbsp;&nbsp;&nbsp;当页面的滚动展示出极大的阻塞,它总会在某些地方展示出性能问题。”被动事件监听“并不会处理这些潜在的问题,所以我们建议开发者要保证他们的应用符合[守则](https://developers.google.com/web/tools/chrome-devtools/profile/evaluate-performance/rail?hl=en)即使在低端设备中。如果你的页面在某个时刻运行超过1000毫秒,它在反应点击或者触摸时就会表现的十分呆滞。被动事件监听仅仅允许开发者将影响滚动性能的JavaScript相应与输入事件管理分开。尤其是第三方分析库的开发者有自信当任何页面使用这些库时它们的代码并不会对可感知的滚动性能有很大影响。
