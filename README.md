## 前言
> 最近项目中需要用到涂鸦的功能，在 Github 上搜了一圈也没找到适合的库，索性就自己撸一个出来，正好复习一下自定义 View 的知识。写完之后怎么可以自己藏着呢，当然得写篇博客分享给大家。

在开始本文的内容之前，先展示一波最终的效果

![DoodleView](http://upload-images.jianshu.io/upload_images/4334738-43c36fbde1f6b882.gif?imageMogr2/auto-orient/strip)

可以看到这个这个自定义 View 的功能还是很丰富的，无论是设置画笔的形状、颜色、粗细，还是进行重置和保存，该有的 API，基本都已经实现了。有需要的读者直接 [点击这里](https://github.com/developerHaoz/DoodleView) ，希望帮忙点个 star，哈哈哈。

## 一、定义画笔的行为类
-----
这里所说的「行为」指的就是我们刚才看到的画笔的形状，无论是路径、直线、还是圆形，这些东西说到底都是画笔的行为。

所以我们先定义一个公共的父类，以便进行管理，减少代码量。
```
abstract class Action {
    public int color;

    Action() {
        color = Color.BLACK;
    }

    Action(int color) {
        this.color = color;
    }

    public abstract void draw(Canvas canvas);

    public abstract void move(float mx, float my);
}
```

可以看到这个类被定义成抽象类，里面有 draw() 和 move() 两个抽象方法，这两个方法就是留给子类进行继承和拓展的，子类只要实现这两个方法，确定好他们各自的行为，就能让画笔显示出各种各样的效果。

接下来举几个具体的子类来说明一下用法：
```
// 自由曲线
class MyPath extends Action {
    private Path path;
    private int size;

    MyPath() {
        path = new Path();
        size = 1;
    }

    MyPath(float x, float y, int size, int color) {
        super(color);
        path = new Path();
        this.size = size;
        path.moveTo(x, y);
        path.lineTo(x, y);
    }

    public void draw(Canvas canvas) {
        Paint paint = new Paint();
        paint.setAntiAlias(true);
        paint.setDither(true);
        paint.setColor(color); // 设置画笔颜色
        paint.setStrokeWidth(size); // 设置画笔粗细
        paint.setStyle(Paint.Style.STROKE);
        canvas.drawPath(path, paint);
    }

    public void move(float mx, float my) {
        path.lineTo(mx, my);
    }
}

// 直线
class MyLine extends Action {
    private float startX;
    private float startY;
    private float stopX;
    private float stopY;
    private int size;

    MyLine() {
        startX = 0;
        startY = 0;
        stopX = 0;
        stopY = 0;
    }

    MyLine(float x, float y, int size, int color) {
        super(color);
        startX = x;
        startY = y;
        stopX = x;
        stopY = y;
        this.size = size;
    }

    public void draw(Canvas canvas) {
        Paint paint = new Paint();
        paint.setAntiAlias(true);
        paint.setStyle(Paint.Style.STROKE);
        paint.setColor(color);
        paint.setStrokeWidth(size);
        canvas.drawLine(startX, startY, stopX, stopY, paint);
    }

    public void move(float mx, float my) {
        stopX = mx;
        stopY = my;
    }
}
```

就拿最常见的自由曲线来作为例子讲一下。我们定义 MyPath 这个类，继承自 BaseAction，然后添加了 Path 和 size 两个成员变量。其中的 size 是用来设置画笔的粗细。Path 是用来确定自由曲线的轨迹。

在 MyPath 的 draw() 方法中我们创建了一个 Paint 用于图形的描绘。最后将 path 和 paint 传给 canvas，实现图形的最终绘制。

```
    public void draw(Canvas canvas) {
        Paint paint = new Paint();
        paint.setAntiAlias(true);
        paint.setDither(true);
        paint.setColor(color); // 设置画笔颜色
        paint.setStrokeWidth(size); // 设置画笔粗细
        paint.setStyle(Paint.Style.STROKE);
        canvas.drawPath(path, paint);
    }
```

其他子类都是按照这种思路来实现，具体的实现可以参考下 Github 上的源码 [DoodleView](https://github.com/developerHaoz/DoodleView)。

## 二、实现自定义的 DoodleView
-----
这个 DoodleView 是直接继承 SurfaceView 的。本来想继承 View 来写，后来仔细想了下最后还是用 SurfaceView 来进行实现。

这里简单说一下 View 和 SurfaceView 的区别。
- View 在主线程中对页面进行刷新，而 SurfaceView 则是另外开了一个子线程对当前页面进行刷新。

- View 适合用于主动更新的情况，而 SurfaceView 则适用于被动更新的情况，比如频繁刷新界面。

因为我们这个涂鸦的 View，是频繁进行刷新的，每次触摸屏幕都会进行相应的界面刷新，所以用 SurfaceView 来实现就比较合理了。

这里我直接结合代码来讲一下 DoodleView 的实现思路，因为我是继承自 SurfaceView 来写的，对于 SurfaceView 不是很了解的朋友，可以先看一下这篇文章 [Android中的SurfaceView详解](http://www.jianshu.com/p/b037249e6d31)

```

public class DoodleView extends SurfaceView implements SurfaceHolder.Callback {

    private SurfaceHolder mSurfaceHolder = null;

    // 当前所选画笔的形状
    private BaseAction curAction = null;
    // 默认画笔为黑色
    private int currentColor = Color.BLACK;
    // 画笔的粗细
    private int currentSize = 5;

    private Paint mPaint;

    private List<BaseAction> mBaseActions;

    private Bitmap mBitmap;

    private ActionType mActionType = ActionType.Path;

    public DoodleView(Context context, AttributeSet attrs, int defStyle) {
        super(context, attrs, defStyle);
        init();
    }

    private void init() {
        mSurfaceHolder = this.getHolder();
        mSurfaceHolder.addCallback(this);
        this.setFocusable(true);

        mPaint = new Paint();
        mPaint.setColor(Color.WHITE);
        mPaint.setStrokeWidth(currentSize);
    }

    @Override
    public void surfaceCreated(SurfaceHolder holder) {
        Canvas canvas = mSurfaceHolder.lockCanvas();
        canvas.drawColor(Color.WHITE);
        mSurfaceHolder.unlockCanvasAndPost(canvas);
        mBaseActions = new ArrayList<>();
    }

    @Override
    public boolean onTouchEvent(MotionEvent event) {
        int action = event.getAction();
        if (action == MotionEvent.ACTION_CANCEL) {
            return false;
        }

        float touchX = event.getRawX();
        float touchY = event.getRawY();

        switch (action) {
            case MotionEvent.ACTION_DOWN:
                setCurAction(touchX, touchY);
                break;
            case MotionEvent.ACTION_MOVE:
                Canvas canvas = mSurfaceHolder.lockCanvas();
                canvas.drawColor(Color.WHITE);
                for (BaseAction baseAction : mBaseActions) {
                    baseAction.draw(canvas);
                }
                curAction.move(touchX, touchY);
                curAction.draw(canvas);
                mSurfaceHolder.unlockCanvasAndPost(canvas);
                break;
            case MotionEvent.ACTION_UP:
                mBaseActions.add(curAction);
                curAction = null;
                break;

            default:
                break;
        }
        return super.onTouchEvent(event);
    }

    /**
     * 得到当前画笔的类型，并进行实例化
     *
     * @param x
     * @param y
     */
    private void setCurAction(float x, float y) {
        switch (mActionType) {
            case Path:
                curAction = new MyPath(x, y, currentSize, currentColor);
                break;
            case Line:
                curAction = new MyLine(x, y, currentSize, currentColor);
                break;
            default:
                break;
        }
    }

    /**
     * 设置画笔的颜色
     *
     * @param color 颜色
     */
    public void setColor(String color) {
        this.currentColor = Color.parseColor(color);
    }

    /**
     * 设置画笔的粗细
     *
     * @param size 画笔的粗细
     */
    public void setSize(int size) {
        this.currentSize = size;
    }

    /**
     * 设置画笔的形状
     *
     * @param type 画笔的形状
     */
    public void setType(ActionType type) {
        this.mActionType = type;
    }

    /**
     * 将当前的画布转换成一个 Bitmap
     *
     * @return Bitmap
     */
    public Bitmap getBitmap() {
        mBitmap = Bitmap.createBitmap(getWidth(), getHeight(), Bitmap.Config.ARGB_8888);
        Canvas canvas = new Canvas(mBitmap);
        doDraw(canvas);
        return mBitmap;
    }

    /**
     * 保存涂鸦后的图片
     *
     * @param doodleView
     * @return 图片的保存路径
     */
    public String saveBitmap(DoodleView doodleView) {
        String path = Environment.getExternalStorageDirectory().getAbsolutePath()
                + "/doodleview/" + System.currentTimeMillis() + ".png";
        if (!new File(path).exists()) {
            new File(path).getParentFile().mkdir();
        }
        savePicByPNG(doodleView.getBitmap(), path);
        return path;
    }

    /**
     * 将一个 Bitmap 保存在一个指定的路径中
     *
     * @param bitmap
     * @param filePath
     */
    public static void savePicByPNG(Bitmap bitmap, String filePath) {
        FileOutputStream fileOutputStream;
        try {
            fileOutputStream = new FileOutputStream(filePath);
            if (null != fileOutputStream) {
                bitmap.compress(Bitmap.CompressFormat.PNG, 90, fileOutputStream);
                fileOutputStream.flush();
                fileOutputStream.close();
            }
        } catch (FileNotFoundException e) {
            e.printStackTrace();
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    /**
     * 开始进行绘画
     *
     * @param canvas
     */
    private void doDraw(Canvas canvas) {
        canvas.drawColor(Color.TRANSPARENT);
        for (BaseAction action : mBaseActions) {
            action.draw(canvas);
        }
        canvas.drawBitmap(mBitmap, 0, 0, mPaint);
    }


    /**
     * 回退
     *
     * @return 是否已经回退成功
     */
    public boolean back(){
        if(mBaseActions != null && mBaseActions.size() > 0){
            mBaseActions.remove(mBaseActions.size() -1);
            Canvas canvas = mSurfaceHolder.lockCanvas();
            canvas.drawColor(Color.WHITE);
            for (BaseAction action : mBaseActions) {
                action.draw(canvas);
            }
            mSurfaceHolder.unlockCanvasAndPost(canvas);
            return true;
        }
        return false;
    }

    /**
     * 重置签名
     */
    public void reset(){
        if(mBaseActions != null && mBaseActions.size() > 0){
            mBaseActions.clear();
            Canvas canvas = mSurfaceHolder.lockCanvas();
            canvas.drawColor(Color.WHITE);
            for (BaseAction action : mBaseActions) {
                action.draw(canvas);
            }
            mSurfaceHolder.unlockCanvasAndPost(canvas);
        }
    }

    enum ActionType {
        Path, Line
    }
}
```

可以看到，我们先定义了一个枚举类，用于区分各种画笔的形状，为了让代码看起来更简洁，我这里只放了 Path 和 Line 两种类型的，如果你还想实现其他类型的形状，直接加进去就行了。

在类的一开始我们定义了一些必要的成员变量，如画笔的颜色、形状、粗细，以及保存画笔行为的 List<BaseAction>，以及需要用到的画笔 Paint

准备工作搞定了之后就开始进行核心代码的实现了。

#### 1、构造函数
```
    public DoodleView(Context context, AttributeSet attrs, int defStyle) {
        super(context, attrs, defStyle);
        init();
    }

    private void init() {
        mSurfaceHolder = this.getHolder();
        mSurfaceHolder.addCallback(this);
        this.setFocusable(true);

        mPaint = new Paint();
        mPaint.setColor(Color.WHITE);
        mPaint.setStrokeWidth(currentSize);
    }
```
可以看到我们在构造函数中先进行了 SurfaceHolder 的一些设置，以及对 Paint 进行了必要的设置。

然后在 surfaceCreated(SurfaceHolder holder) 方法中对 Canas 进行了创建和提交，以及初始化了 List<BaseAction> 

#### 2、触摸事件的处理
这个方法的实现可以说是这个 DoodleView 的核心了

```
    @Override
    public boolean onTouchEvent(MotionEvent event) {
        int action = event.getAction();
        if (action == MotionEvent.ACTION_CANCEL) {
            return false;
        }

        float touchX = event.getRawX();
        float touchY = event.getRawY();

        switch (action) {
            case MotionEvent.ACTION_DOWN:
                setCurAction(touchX, touchY);
                break;
            case MotionEvent.ACTION_MOVE:
                Canvas canvas = mSurfaceHolder.lockCanvas();
                canvas.drawColor(Color.WHITE);
                for (BaseAction baseAction : mBaseActions) {
                    baseAction.draw(canvas);
                }
                curAction.move(touchX, touchY);
                curAction.draw(canvas);
                mSurfaceHolder.unlockCanvasAndPost(canvas);
                break;
            case MotionEvent.ACTION_UP:
                mBaseActions.add(curAction);
                curAction = null;
                break;

            default:
                break;
        }
        return super.onTouchEvent(event);
    }
```
我们先拿到触摸的横坐标和纵坐标，然后根据手势来进行相应的处理
- ACTION_DOWN：当刚开始出触摸屏幕的时候，先设置画笔的形状

- ACTION_MOVE：手开始移动的时候，调用 move() 和 draw() 对 Canvas 进行绘制，最后将 Canvas 的内容进行提交。

- ACTION_UP：将手抬起来的时候，将当前画笔的形状添加到 List<BaseAction> 中，并将 curAction（当前的画笔形状）设为 null.

#### 3、其他的 API
除了一些核心方法的实现，为了拓展这个 DoodleView 的功能，我还添加了一些实用的 API。

##### 保存涂鸦后的图片
```
    public String saveBitmap(DoodleView doodleView) {
        String path = Environment.getExternalStorageDirectory().getAbsolutePath()
                + "/doodleview/" + System.currentTimeMillis() + ".png";
        if (!new File(path).exists()) {
            new File(path).getParentFile().mkdir();
        }
        savePicByPNG(doodleView.getBitmap(), path);
        return path;
    }

    public static void savePicByPNG(Bitmap bitmap, String filePath) {
        FileOutputStream fileOutputStream;
        try {
            fileOutputStream = new FileOutputStream(filePath);
            if (null != fileOutputStream) {
                bitmap.compress(Bitmap.CompressFormat.PNG, 90, fileOutputStream);
                fileOutputStream.flush();
                fileOutputStream.close();
            }
        } catch (FileNotFoundException e) {
            e.printStackTrace();
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
```
先创建一个用于保存图片的路径，判断路径是否存在，如果不存在的话，就创建一下。否则通过这个路径拿到对应的文件流，并将当前图片转换成 Bitmap 之后放进去。

##### 重置涂鸦的界面
我们进行涂鸦，难免会出现手误，这时候进行重置就显得相当重要了。
```
    public void reset(){
        if(mBaseActions != null && mBaseActions.size() > 0){
            mBaseActions.clear();
            Canvas canvas = mSurfaceHolder.lockCanvas();
            canvas.drawColor(Color.WHITE);
            for (BaseAction action : mBaseActions) {
                action.draw(canvas);
            }
            mSurfaceHolder.unlockCanvasAndPost(canvas);
        }
    }
```
这里直接获取 Canvas，然后将 List<BaseAction> 进行 clear，因为 List<BaseAction> 里面没有内容，Canvas 上自然也就没有任何东西，最后将 Canvas 进行提交。

以上便是本文的全部内容，有兴趣的同学可以 [点击这里](https://github.com/developerHaoz/DoodleView) 看一下具体实现，麻烦点个 star，谢谢了。

-----
### 猜你喜欢
- [Android 一起来看看知乎开源的图片选择库](http://www.jianshu.com/p/382346bf0aa9)
- [Android 能让你少走弯路的干货整理](http://www.jianshu.com/p/514656c383a2)
- [Android 撸起袖子，自己封装 DialogFragment](http://www.jianshu.com/p/c9f20ec7277a)
- [手把手教你从零开始做一个好看的 APP](http://www.jianshu.com/p/8d2d74d6046f)
