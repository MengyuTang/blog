# Android事件分发机制

##  事件传递过程：
    Activity->ViewGroup->View
    
##  主要方法：
    1.dispatchTouchEvent()
    2.onTouchEvent()
    3.onInterceptTouchEvent()
    
##  Activity中的事件分发流程
       
```
/**
     * Called to process touch screen events.  You can override this to
     * intercept all touch screen events before they are dispatched to the
     * window.  Be sure to call this implementation for touch screen events
     * that should be handled normally.
     *
     * @param ev The touch screen event.
     *
     * @return boolean Return true if this event was consumed.
     */
    public boolean dispatchTouchEvent(MotionEvent ev) {
        if (ev.getAction() == MotionEvent.ACTION_DOWN) {
            onUserInteraction();
        }
        if (getWindow().superDispatchTouchEvent(ev)) {
            return true;
        }
        return onTouchEvent(ev);
    }
    
/**
  * 当此activity在栈顶时，触屏点击按home，back，menu键等都会触发此方法
  */
    public void onUserInteraction() { 

      }

    public boolean superDispatchTouchEvent(MotionEvent event) {
        return mDecor.superDispatchTouchEvent(event);
    }

    public boolean superDispatchTouchEvent(MotionEvent event) {

        return super.dispatchTouchEvent(event);

    }
    
  @Override
  public boolean onTouchEvent(MotionEvent event) {

        if (mWindow.shouldCloseOnTouch(this, event)) {
            finish();
            return true;
        }
        
        return false;
    }
    
    public boolean shouldCloseOnTouch(Context context, MotionEvent event) {
    if (mCloseOnTouchOutside && event.getAction() == MotionEvent.ACTION_DOWN
            && isOutOfBounds(context, event) && peekDecorView() != null) {
        return true;
    }
    return false;
}
```
### dispatchTouchEvent()
  我们来看下它是怎么把事件传递到VieGroup中的。在dispatchTouchEvent()中，首先是处理downn事件，这个没什么好说的，接下来才是重点，我们看到getWindow().superDispatchTouchEvent(ev)，其中getWindow是获取PhoneWindow对象，然后调用PhoneWindow的superDispatchTouchEvent(),这就是我们要分析的第三步
### superDispatchTouchEvent
  它的内部是调用的mDecor.superDispatchTouchEvent(event)，这个mDecor是DecorView，DecorView是PhoneWindow的一个内部类，它继承自FrameLayout，而FrameLayout又是ViewGroup的子类，因此mDecor.superDispatchTouchEvent(event)其实调用的是ViewGroup的dispatchTouchEvent()方法，这样就实现是事件从Activity传到Viewgroup，在ViewGroup内处理完成之后，如果返回true，那说明Viewgroup消费了该事件，此次事件直接结束；如果返回false，那么事件就继续交给Activity来处理。所以最后就走到了第四步分析，Activity的onTouchEvent()方法。
### onTouchEvent()
  在OnTouchEvent()方法中一般是返回false，交给Activity的子类来继续处理，除非遇到当一个点击事件未被Activity下任何一个View接收 / 处理时，此时点击事件是否发生在Window边界外，如果是，则返回true，消费掉事件，并且执行finish(),否则，返回false。
  
  刚刚说到DispatchtouchEvent()，如果activity没有消费该次事件，则会传递到ViewGroup，那么我们来看下ViewGroup中是事件传递过程。
    
## ViewGroup中的事件分发流程
    ###  ViewGroup的dispatchTouchEvent()到底做了什么？
        
```
    @Override
    public boolean dispatchTouchEvent(MotionEvent ev) { 

        ……//省略无关代码

            if (disallowIntercept || !onInterceptTouchEvent(ev)) {  
            // disallowIntercept：disallowIntercept = 是否禁用事件拦截的功能(默认是false)，可通过调用requestDisallowInterceptTouchEvent（）修改
            // onInterceptTouchEvent： !onInterceptTouchEvent(ev) = 对onInterceptTouchEvent()返回值取反
                ev.setAction(MotionEvent.ACTION_DOWN);  
                final int scrolledXInt = (int) scrolledXFloat;  
                final int scrolledYInt = (int) scrolledYFloat;  
                final View[] children = mChildren;  
                final int count = mChildrenCount;  

            // 通过for循环，遍历了当前ViewGroup下的所有子View，找到被点击的子View
            for (int i = count - 1; i >= 0; i--) {  
                final View child = children[i];  
                if ((child.mViewFlags & VISIBILITY_MASK) == VISIBLE  
                        || child.getAnimation() != null) {  
                    child.getHitRect(frame);  
                    if (frame.contains(scrolledXInt, scrolledYInt)) {  
                        final float xc = scrolledXFloat - child.mLeft;  
                        final float yc = scrolledYFloat - child.mTop;  
                        ev.setLocation(xc, yc);  
                        child.mPrivateFlags &= ~CANCEL_NEXT_UP_EVENT;  

                        //通过调用子View的dispatchTouchEvent()将事件传递给子View并获取处理结果
                        if (child.dispatchTouchEvent(ev))  { 
                            mMotionTarget = child;  
                            return true; 
                                }  
                            }  
                        }  
                    }  
                }  
            }  
            boolean isUpOrCancel = (action == MotionEvent.ACTION_UP) ||  
                    (action == MotionEvent.ACTION_CANCEL);  
            if (isUpOrCancel) {  
                mGroupFlags &= ~FLAG_DISALLOW_INTERCEPT;  
            }  
            final View target = mMotionTarget;  
        //如果所有的子View都没有被点击，则点击的是空白处，此时调用ViewGroup父类的ispatchTouchEvent()方法
        if (target == null) {  
            ev.setLocation(xf, yf);  
            if ((mPrivateFlags & CANCEL_NEXT_UP_EVENT) != 0) {  
                ev.setAction(MotionEvent.ACTION_CANCEL);  
                mPrivateFlags &= ~CANCEL_NEXT_UP_EVENT;  
            }  
            
            return super.dispatchTouchEvent(ev);
        } 

        ... 

}
/**
  * 作用：是否拦截事件
  */
  public boolean onInterceptTouchEvent(MotionEvent ev) {  
    
    return false;
    
  } 

```
### dispatchTouchEvent()
  首先根据disallowIntercept判断是否禁用了拦截功能，然后通过onInterceptTouchEvent()方法判断是否拦截了事件，如果没有拦截或者没有禁用拦截功能，则进入循环，遍历ViewGroup所有的子View，找出被点击的View，如果找到了，则将事件传递到子View的dispatchTouchEvent()方法，交给子View处理，并返回处理结果，如果返回true，则说明在子View中消费了该次事件，ViewGroup的dispatchTouchEvent()也是直接返回true，拦截掉该次事件。我们在Activity的事件分发流程中说过，如果ViewGroup的dispatchTouchEvent()返回true，则事件被ViewGroup消费，事件直接结束。 
### 那么如果子View的dispatchTouchEvent()方法返回false呢？ 
  如果子View没有消费此次事件，则调用super.dispatchTouchEvent()方法，即交由ViewGroup的父类来处理，而View是ViewGroup的父类，因此会调用View的onTouch()->onTouchEvent()->performClick()->onClick()，自己处理该次事件。
    
  需要注意的是，这里的View与前文提到的子View不是一个东西，这里的View是ViewGroup的父类，其实还是相当于ViewGroup在执行操作。
      
## View的事件分发流程
  在ViewGroup的dispaatchTouchEvent()方法中，如果找到了被点击的子View，就会将事件交由子View处理。这时候就进入了子View的事件分发流程。
### 这里涉及到的核心方法有：
        1.dispatchTouchEvent()
        2.onTouch()
        3.onTouchEvent()
        4.performClick()
        4.onClick()
  那么接下来我们就循着这些方法依次来看。
### dispatchTouchEvent()
```
  public boolean dispatchTouchEvent(MotionEvent event) {  

        if (mOnTouchListener != null && (mViewFlags & ENABLED_MASK) == ENABLED &&  
                mOnTouchListener.onTouch(this, event)) {  
            return true;  
        } 
        return onTouchEvent(event);  
  }
```
在dispatchTouchEvent()方法中，有三个条件，只有以下3个条件都为真，dispatchTouchEvent()才返回true；否则执行onTouchEvent()
       1. mOnTouchListener != null
       2. (mViewFlags & ENABLED_MASK) == ENABLED
       3. mOnTouchListener.onTouch(this, event)
那么这三个条件分别是什么意思呢？
  
```
/**
  * 条件1：mOnTouchListener != null
  */
  public void setOnTouchListener(OnTouchListener l) { 
    mOnTouchListener = l;  
} 

  条件2：(mViewFlags & ENABLED_MASK) == ENABLED
  该条件是判断当前点击的控件是否enable，大多数View默认enable
 
 /**
  * 条件3：mOnTouchListener.onTouch(this, event)
  * 若在onTouch（）返回true，从而使得View.dispatchTouchEvent（）直接返回true，事件分发结束
  * 否则，执行onTouchEvent(event) 
  */
    button.setOnTouchListener(new OnTouchListener() {  
        @Override  
        public boolean onTouch(View v, MotionEvent event) {  
            return false;  
        }  
    });
```
搞懂了这三个条件，接下来就来看看onTouchEvent()方法
  
```
  public boolean onTouchEvent(MotionEvent event) {  
    final int viewFlags = mViewFlags;  

    if ((viewFlags & ENABLED_MASK) == DISABLED) {  
         
        return (((viewFlags & CLICKABLE) == CLICKABLE ||  
                (viewFlags & LONG_CLICKABLE) == LONG_CLICKABLE));  
    }  
    if (mTouchDelegate != null) {  
        if (mTouchDelegate.onTouchEvent(event)) {  
            return true;  
        }  
    }  

    // 若该控件可点击，则继续走下去
    if (((viewFlags & CLICKABLE) == CLICKABLE ||  
            (viewFlags & LONG_CLICKABLE) == LONG_CLICKABLE)) {  

                switch (event.getAction()) { 
                    case MotionEvent.ACTION_UP:  
                        boolean prepressed = (mPrivateFlags & PREPRESSED) != 0;  
                            ……//省略无关代码
                            performClick();  //抬起时执行performClick()
                            break;  

                    case MotionEvent.ACTION_DOWN:  
                        if (mPendingCheckForTap == null) {  
                            mPendingCheckForTap = new CheckForTap();  
                        }  
                        mPrivateFlags |= PREPRESSED;  
                        mHasPerformedLongPress = false;  
                        postDelayed(mPendingCheckForTap, ViewConfiguration.getTapTimeout());  
                        break;  

                    case MotionEvent.ACTION_CANCEL:  
                        mPrivateFlags &= ~PRESSED;  
                        refreshDrawableState();  
                        removeTapCallback();  
                        break;

                    case MotionEvent.ACTION_MOVE:  
                        final int x = (int) event.getX();  
                        final int y = (int) event.getY();  
        
                        int slop = mTouchSlop;  
                        if ((x < 0 - slop) || (x >= getWidth() + slop) ||  
                                (y < 0 - slop) || (y >= getHeight() + slop)) {  
                            // Outside button  
                            removeTapCallback();  
                            if ((mPrivateFlags & PRESSED) != 0) {  
                                removeLongPressCallback();  
                                mPrivateFlags &= ~PRESSED;  
                                refreshDrawableState();  
                            }  
                        }  
                        break;  
                }  
                return true;  
            }  
            return false;  
        }

    public boolean performClick() {  

        if (mOnClickListener != null) {  
            playSoundEffect(SoundEffectConstants.CLICK);  
            mOnClickListener.onClick(this);  
            return true;  
        }  
        return false;  
    }  
```
### onTouchEvent()
  通过以上源码可以看到，在发生抬起动作（ACTION_UP）时，经过一系列条件判断，会执行performClick()方法。
### performClick()
  在这个方法中，我们看到最关键的代码是 mOnClickListener.onClick(this);前提是 mOnClickListener != null,即我们为控件设置了OnClickListener,并实现了onClick()方法。到这里，点击事件基本上走完了，后面的就不用讲了吧。
