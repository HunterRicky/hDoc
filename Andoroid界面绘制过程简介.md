#Andoroid界面绘制过程简介

###1. View 框架
Android中的View是呈树状结构的，而这其中，就有一个角色`ViewRoot`，管理着树的根，而它本身并非树的根元素。`ViewRoot`的职责就是与`WindowManagerService`进行通信。而最常见到的`Activity`内部有一个`mWindow`的成员变量，表示UI界面的框架。这个`Window`将界面和`Activity`解耦开来，是的整个界面不会影响到`Activity`的内容。此外，`Window`还需要与`WindowManagerService`进行通信，方式就是利用`ViewRoot`。

一个进程中不仅有一个`ViewRoot`，而且`Activity`和`ViewRoot`也是一对一的关系。`ViewRoot`通过`WindowManagerService`提供的`openSession()`接口来创建连接。

一个`Activity`对应唯一的`WindowManager`以及`ViewRootImpl`。`ViewRootImpl`就是对`ViewRoot`接口的实现。`ViewRootImpl`的任务是与`WindowManagerService`进行通信，其到`WindowManagerService`方向是通过`IWindowSession`完成，而反方向则是通过`IWindow`完成。

###2. 从`setContentView()`开始
在Android开发过程中，一般来说我们设置页面的内容视图是都是通过`setContentView()`方法。所以我们就从这个方法开始讲起，简要介绍Android界面的绘制过程。

```java
public void setContentView(@LayoutRes int layoutResID) {
        getWindow().setContentView(layoutResID);
        initWindowDecorActionBar();
    }
```

这里调用了`getWindow()`的该方法，这个`window`其实就是`PhoneWindow`，它的方法定义如下：

```java
@Override
    public void setContentView(int layoutResID) {
        if (mContentParent == null) {
            installDecor();
        } else if (!hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
            mContentParent.removeAllViews();
        }

        if (hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
            final Scene newScene = Scene.getSceneForLayout(mContentParent, layoutResID,
                    getContext());
            transitionTo(newScene);
        } else {
            mLayoutInflater.inflate(layoutResID, mContentParent);
        }
        mContentParent.requestApplyInsets();
        final Callback cb = getCallback();
        if (cb != null && !isDestroyed()) {
            cb.onContentChanged();
        }
        mContentParentExplicitlySet = true;
    }
```

这个方法分三步进行：

1. 判断父容器是否为空，为空则生成Decor；不为空，则删除contentParent 所有的子控件。
2. 解析layoutResID所代表的xml文件。
3. 通知Callback，ContentView发生改变。

在填充布局的过程中，最终调用到的是`inflate(XmlPullParser parser, @Nullable ViewGroup root, boolean attachToRoot)`方法：

```java
public View inflate(XmlPullParser parser, @Nullable ViewGroup root, boolean attachToRoot) {
        synchronized (mConstructorArgs) {
            Trace.traceBegin(Trace.TRACE_TAG_VIEW, "inflate");

            final Context inflaterContext = mContext;
            final AttributeSet attrs = Xml.asAttributeSet(parser);
            Context lastContext = (Context) mConstructorArgs[0];
            mConstructorArgs[0] = inflaterContext;
            View result = root;

            try {
                // Look for the root node.
                int type;
                while ((type = parser.next()) != XmlPullParser.START_TAG &&
                        type != XmlPullParser.END_DOCUMENT) {
                    // Empty
                }

                if (type != XmlPullParser.START_TAG) {
                    throw new InflateException(parser.getPositionDescription()
                            + ": No start tag found!");
                }

                final String name = parser.getName();
                ...
                if (TAG_MERGE.equals(name)) {
                    if (root == null || !attachToRoot) {
                        throw new InflateException("<merge /> can be used only with a valid "
                                + "ViewGroup root and attachToRoot=true");
                    }

                    rInflate(parser, root, inflaterContext, attrs, false);
                } else {
                    // Temp is the root view that was found in the xml
                    final View temp = createViewFromTag(root, name, inflaterContext, attrs);

                    ViewGroup.LayoutParams params = null;

                    if (root != null) {
                        if (DEBUG) {
                            System.out.println("Creating params from root: " +
                                    root);
                        }
                        // Create layout params that match root, if supplied
                        params = root.generateLayoutParams(attrs);
                        if (!attachToRoot) {
                            // Set the layout params for temp if we are not
                            // attaching. (If we are, we use addView, below)
                            temp.setLayoutParams(params);
                        }
                    }
                    ...
                    // Inflate all children under temp against its context.
                    rInflateChildren(parser, temp, attrs, true);
                    ...
                    // We are supposed to attach all the views we found (int temp)
                    // to root. Do that now.
                    if (root != null && attachToRoot) {
                        root.addView(temp, params);
                    }

                    // Decide whether to return the root that was passed in or the
                    // top view found in xml.
                    if (root == null || !attachToRoot) {
                        result = temp;
                    }
                }
            } 
            ...
            return result;
        }
    }
```

然后调用`rInflate()`进行处理：

```java
void rInflate(XmlPullParser parser, View parent, Context context,
            AttributeSet attrs, boolean finishInflate) throws XmlPullParserException, IOException {

        final int depth = parser.getDepth();
        int type;

        while (((type = parser.next()) != XmlPullParser.END_TAG ||
                parser.getDepth() > depth) && type != XmlPullParser.END_DOCUMENT) {

            if (type != XmlPullParser.START_TAG) {
                continue;
            }

            final String name = parser.getName();
            
            if (TAG_REQUEST_FOCUS.equals(name)) {
                parseRequestFocus(parser, parent);
            } else if (TAG_TAG.equals(name)) {
                parseViewTag(parser, parent, attrs);
            } else if (TAG_INCLUDE.equals(name)) {
                if (parser.getDepth() == 0) {
                    throw new InflateException("<include /> cannot be the root element");
                }
                parseInclude(parser, context, parent, attrs);
            } else if (TAG_MERGE.equals(name)) {
                throw new InflateException("<merge /> must be the root element");
            } else {
                final View view = createViewFromTag(parent, name, context, attrs);
                final ViewGroup viewGroup = (ViewGroup) parent;
                final ViewGroup.LayoutParams params = viewGroup.generateLayoutParams(attrs);
                rInflateChildren(parser, view, attrs, true);
                viewGroup.addView(view, params);
            }
        }

        if (finishInflate) {
            parent.onFinishInflate();
        }
    }
```

其中，`inflate()`和`rInflate`都调用了`rInflateChildren()`，之后再次调用`rInflate()`，通过这个循环过程来完成整个树形结构的绘制。


###3. View的绘制流程
在`contentView`创建完成之后，就正式进入到了`View`的绘制流程了。这个过程是由`ViewRoot`来负责的。从`requestLayout()`方法开始：

```java
 @Override
    public void requestLayout() {
        if (!mHandlingLayoutInLayoutRequest) {
            checkThread();
            mLayoutRequested = true;
            scheduleTraversals();
        }
    }
```

这一过程又是从`scheduleTraversals()`开始执行的：

```java
void scheduleTraversals() {
        if (!mTraversalScheduled) {
            mTraversalScheduled = true;
            mTraversalBarrier = mHandler.getLooper().getQueue().postSyncBarrier();
            mChoreographer.postCallback(
                    Choreographer.CALLBACK_TRAVERSAL, mTraversalRunnable, null);
            if (!mUnbufferedInputDispatch) {
                scheduleConsumeBatchedInput();
            }
            notifyRendererOfFramePending();
            pokeDrawLockIfNeeded();
        }
    }
```

该方法会向主线程发送消息，最终触发`performTraversals()`方法，该方法代码量十分巨大，因此就不在此贴出了。一旦触发该操作，就会从`decorView`开始进行`measure`，`Layout`，`draw`了。因此，`View`的绘制流程也主要分为以下三步：

1. measure: 判断是否需要重新计算View的大小，需要的话则计算
2. layout: 判断是否需要重新计算View的位置，需要的话则计算
3. draw: 判断是否需要重新绘制View，需要的话则重绘制

####3.1 `measure()`阶段
`measure()`过程为整个View树来计算实际的大小，即宽、高等属性。对于`measure()`方法，先来看看源码：

```java
public final void measure(int widthMeasureSpec, int heightMeasureSpec) {
        ...
        onMeasure(widthMeasureSpec, heightMeasureSpec);
        ...
    }
```

在注释中，源码说明了，每个View控件的实际宽和高都是由父视图和自身决定的。测量过程则在`onMeasure()`中执行。该方法主要实现以下功能：

1. 设置本View视图的最终大小，该功能的实现通过调用`setMeasuredDimension()`方法去设置实际的高(对应属性：`mMeasuredHeight()`和宽对应属性：`mMeasureWidth()`
2. 如果该View对象是个ViewGroup类型，需要重写该`onMeasure()`方法，对其子视图进行遍历的`measure()`过程。对每个子视图的`measure()`过程，是通过调用父类`ViewGroup.java`类里的`measureChildWithMargins()`方法去实现，该方法内部只是简单地调用了View对象的`measure()`方法。

####3.2 `layout()`阶段
layout阶段的基本思想也是由根View开始，递归地完成整个控件树的布局（layout）工作。该阶段主要对View各层级进行定位。首先使用`host.layout()`开始View树的布局，继而回调给View/ViewGroup类中的`layout()`方法。具体流程如下：

1. `layout`方法会设置该View视图位于父视图的坐标轴，即mLeft，mTop，mLeft，mBottom。接下来回调`onLayout()`方法如果该View是ViewGroup对象，需要实现该方法，对每个子视图进行布局
2. 如果该View是个`ViewGroup`类型，需要遍历每个子视图`chiildView`，调用该子视图的`layout()`方法去设置它的坐标值

####3.3 `draw()`阶段
这一步，正式对视图进行绘制。执行的起点就是`ViewRootImpl`中用于记录`ViewTree`根元素的成员变量`mView`的`draw`函数。这一步的主要步骤如下：

1. 绘制背景
2. 保存Canvas的Layers
3. 绘制内容区域
4. 绘制子对象
5. 绘制fading
6. 绘制decorations(滚动条等组件)

至此整个View便完成绘制了。本文内容仅仅是对Android界面绘制过程的一个开题，在下个月的学习中将会逐步完善整个界面的绘制过程内容。













