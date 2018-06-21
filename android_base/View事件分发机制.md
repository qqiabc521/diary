# **View事件分发机制**

在介绍View的事件分发机制之前，我们要先了解两个概念。

* MotionEvent：Android中用来表示各种事件的对象，例如ACTION_DOWN、ACTION_MOVE等，我们还可以通过它获取事件发生的坐标，getX/getY获取相对于当前View左上角的坐标，getRawX/getRawY获取相对于屏幕左上角的坐标。
* TouchSlop：系统所能识别的最小滑动距离，通过ViewConfiguration.get(context).getScaledTouchSlop()方法获取。

现在我们再来看看View里的事件分发机制，概括来说，可以用下面代码表示：

```java
public boolean dispatchTouchEvent(MotionEvent event){
    boolean consume = false;
    //父View决定是否拦截事件
    if(onInterceptTouchEvent(event)){
        //父View调用onTouchEvent(event)消费事件
        consume = onTouchEvent(event);
    }else{
        //调用子View的dispatchTouchEvent(event)方法继续分发事件
        consume = child.dispatchTouchEvent(event);
    }
    return consume;
}
```
我们再来具体看看各个场景中的事件分发。

## Activity的事件分发

当点击事件发生时，事件最先传递给Activity，Activity会首先将事件将诶所属的Window进行处理，即调用superDispatchTouchEvent()方法。

通过观察superDispatchTouchEvent()方法的调用链，我们可以发现事件的传递顺序：

* PhoneWinodw.superDispatchTouchEvent()
* DecorView.dispatchTouchEvent(event)
* ViewGroup.dispatchTouchEvent(event)

事件一层层传递到了ViewGroup里，关于ViewGroup对事件的处理，我们下面会说，如果superDispatchTouchEvent()方法返回false，即没有 处理该事件，则会继续调用Activity的onTouchEvent(ev)方法来处理该事件。可见Activity的onTouchEvent(ev)在事件处理的优先级是最低的。

```java
public class Activity extends ContextThemeWrapper
        implements LayoutInflater.Factory2,
        Window.Callback, KeyEvent.Callback,
        OnCreateContextMenuListener, ComponentCallbacks2,
        Window.OnWindowDismissedCallback, WindowControllerCallback {
    
        public boolean dispatchTouchEvent(MotionEvent ev) {
            if (ev.getAction() == MotionEvent.ACTION_DOWN) {
                onUserInteraction();
            }
            if (getWindow().superDispatchTouchEvent(ev)) {
                return true;
            }
            return onTouchEvent(ev);
        }
}
```

##ViewGroup的事件分发
ViewGroup作为View容器，它需要考虑自己的子View是否处理了该事件，具体说来：

* 如果ViewGroup拦截了事件，即它的onInterceptTouchEvent()返回true，则该事件由ViewGroup处理，如果ViewGroup调用了setOnTouchListener()则该接口的onTouch()方法会被调用 否则会调用onTouchEvent()方法。
* 如果ViewGroup没有拦截事件，则该事件会传递给它的子View，子View的dispatchTouchEvent()会被调用，View.dispatchTouchEvent()的处理流程前面我们已经分析过。

##View的事件分发
View没有子元素，无法向下传递事件，它只能自己处理事件，所以View的事件传递比较简单。

如果外界设置了OnTouchListener且OnTouchListener.onTouch(this, event)返回true，则表示该方法消费了该事件，则onTouchEvent(event)不再被调用。 可见OnTouchListener的优先级高于onTouchEvent(event)，这样是为了便于外界处理事件。


**关于onTouchEvent(MotionEvent event)，有两点需要说明一下：**

1. View的disable属性不会影响onTouchEvent()方法的返回值，哪怕View是disable的，只要 View的clickable或者longClickable为true，onTouchEvent()方法还是会返回true。
2. 只要clickable或者longClickable为true，onTouchEvent()方法就会消费这个事件
3. 如果View设置了OnClickListener，则performClick()会调用它的onClick方法。

上面我们提到了viewFlags里的CLICKABLE与LONG_CLICKABLE，也就是xml或者代码里可以设置的clickable与longClickable，View的LONG_CLICKABLE默认为 true，CLICKABLE默认为false，值得一提的是setOnClickListener()方法和setOnLongClickListener()会将这两个值设置为true。

通过对源码的分析，我们已经掌握了各种场景下事件分发的规律，我们再来总结一下View事件分发的相关结论。

* 事件的传递是按照Activity -> Window -> View的顺序进行的
* 一般情况下，一个事件序列只能由一个View拦截并消耗，一旦一个View拦截了该事件，则该事件序列的后续事件都会交由该View来处理。
* ViewGroup默认不拦截任何事件
* View没有onInterceptTouchEvent()方法，一但有点击事件传递给它，它的ouTouchEvent()方法就会被调用。