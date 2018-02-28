---
title: 如何优化listView
category:
  - Android
  - Android_everyday
tags:
  - Android_everyday
  - Android
description: "结合ConvertView,ViewHolder和源码中的RecycleBin机制分析了ListView的优化，并给出了在滑动时不加载图片的代码实现。"
date: 2016-10-10 23:00:34
---

[如何优化ListView的性能？](http://www.jianshu.com/p/8fd5fa90ee6c)

## 重用ConvertView和使用ViewHolder模式
这两项是优化listview的重要方法，基本上已经是写自定义adapter的标配了。
```
static class ViewHolder {
	TextView text;
	ImageView icon;
}
public View getView(int position, View convertView, ViewGroup parent) 
{
	ViewHolder holder;
	if (convertView == null) {
	convertView = mInflater.inflate(R.layout.list_item_icon_text,parent, false);
	holder = new ViewHolder();
	holder.text = (TextView)	convertView.findViewById(R.id.text);
	holder.icon = (ImageView) convertView.findViewById(R.id.icon);
	convertView.setTag(holder);
} else {
	holder = (ViewHolder) convertView.getTag();
}

//set values to the viewHolder
holder.text.setText(DATA[position]);
holder.icon.setImageBitmap((position & 1) == 1 ? mIcon1 : 
mIcon2);
return convertView;
}
```
可以看到优化ListView的性能很多时候都是在getView中完成的。

先说ViewHolder:ViewHolder的使用，降低了进行findViewById的次数，当convertView不为空的情况下，可以直接执行set value的操作。

然后说convertView：convertView是getView函数的第二个参数。也就是说，getView对convertView的复用，是依靠调用它的函数传入一个convertView实现的。那么是哪里调用了getView函数呢？

## ListView的复用机制
ListView中有一个RecycleBin类复负回收不可见且可能被再次使用的ItemView，由ScrapView存储。

RecycleBin机制是ListView能够实现成百上千条数据都不会OOM最重要的一个原因。它是写在AbsListView中的一个内部类，所以所有继承自AbsListView的子类，也就是ListView和GridView，都可以使用这个机制。RecycleBin有几个重要的函数:
### fillActiveViews
```
/**
         * Fill ActiveViews with all of the children of the AbsListView.
         *
         * @param childCount The minimum number of views mActiveViews should hold
         * @param firstActivePosition The position of the first view that will be stored in
         *        mActiveViews
         */
        void fillActiveViews(int childCount, int firstActivePosition) {
            if (mActiveViews.length < childCount) {
                mActiveViews = new View[childCount];
            }
            mFirstActivePosition = firstActivePosition;

            //noinspection MismatchedReadAndWriteOfArray
            final View[] activeViews = mActiveViews;
            for (int i = 0; i < childCount; i++) {
                View child = getChildAt(i);
                AbsListView.LayoutParams lp = (AbsListView.LayoutParams) child.getLayoutParams();
                // Don't put header or footer views into the scrap heap
                if (lp != null && lp.viewType != ITEM_VIEW_TYPE_HEADER_OR_FOOTER) {
                    // Note:  We do place AdapterView.ITEM_VIEW_TYPE_IGNORE in active views.
                    //        However, we will NOT place them into scrap views.
                    activeViews[i] = child;
                    // Remember the position so that setupChild() doesn't reset state.
                    lp.scrappedFromPosition = firstActivePosition + i;
                }
            }
        }
```
这个方法接收两个参数，第一个参数表示要存储的view的数量，第二个参数表示ListView中第一个可见元素的position值。RecycleBin当中使用mActiveViews这个数组来存储View，调用这个方法后就会根据传入的参数来将ListView中的指定元素存储到mActiveViews数组当中。

### getActiveView
```

        /**
         * Get the view corresponding to the specified position. The view will be removed from
         * mActiveViews if it is found.
         *
         * @param position The position to look up in mActiveViews
         * @return The view if it is found, null otherwise
         */
        View getActiveView(int position) {
            int index = position - mFirstActivePosition;
            final View[] activeViews = mActiveViews;
            if (index >=0 && index < activeViews.length) {
                final View match = activeViews[index];
                activeViews[index] = null;
                return match;
            }
            return null;
        }
```
getActiveView() 这个方法和fillActiveViews()是对应的，用于从mActiveViews数组当中获取数据。该方法接收一个position参数，表示元素在ListView当中的位置，方法内部会自动将position值转换成mActiveViews数组对应的下标值。需要注意的是，mActiveViews当中所存储的View，一旦被获取了之后就会从mActiveViews当中移除，下次获取同样位置的View将会返回null，也就是说mActiveViews不能被重复利用。

### addScrapView()
```
/**
         * Puts a view into the list of scrap views.
         * <p>
         * If the list data hasn't changed or the adapter has stable IDs, views
         * with transient state will be preserved for later retrieval.
         *
         * @param scrap The view to add
         * @param position The view's position within its parent
         */
        void addScrapView(View scrap, int position) {
            final AbsListView.LayoutParams lp = (AbsListView.LayoutParams) scrap.getLayoutParams();
            if (lp == null) {
                // Can't recycle, but we don't know anything about the view.
                // Ignore it completely.
                return;
            }

            lp.scrappedFromPosition = position;

            // Remove but don't scrap header or footer views, or views that
            // should otherwise not be recycled.
            final int viewType = lp.viewType;
            if (!shouldRecycleViewType(viewType)) {
                // Can't recycle. If it's not a header or footer, which have
                // special handling and should be ignored, then skip the scrap
                // heap and we'll fully detach the view later.
                if (viewType != ITEM_VIEW_TYPE_HEADER_OR_FOOTER) {
                    getSkippedScrap().add(scrap);
                }
                return;
            }

            scrap.dispatchStartTemporaryDetach();

            // The the accessibility state of the view may change while temporary
            // detached and we do not allow detached views to fire accessibility
            // events. So we are announcing that the subtree changed giving a chance
            // to clients holding on to a view in this subtree to refresh it.
            notifyViewAccessibilityStateChangedIfNeeded(
                    AccessibilityEvent.CONTENT_CHANGE_TYPE_SUBTREE);

            // Don't scrap views that have transient state.
            final boolean scrapHasTransientState = scrap.hasTransientState();
            if (scrapHasTransientState) {
                if (mAdapter != null && mAdapterHasStableIds) {
                    // If the adapter has stable IDs, we can reuse the view for
                    // the same data.
                    if (mTransientStateViewsById == null) {
                        mTransientStateViewsById = new LongSparseArray<>();
                    }
                    mTransientStateViewsById.put(lp.itemId, scrap);
                } else if (!mDataChanged) {
                    // If the data hasn't changed, we can reuse the views at
                    // their old positions.
                    if (mTransientStateViews == null) {
                        mTransientStateViews = new SparseArray<>();
                    }
                    mTransientStateViews.put(position, scrap);
                } else {
                    // Otherwise, we'll have to remove the view and start over.
                    getSkippedScrap().add(scrap);
                }
            } else {
                if (mViewTypeCount == 1) {
                    mCurrentScrap.add(scrap);
                } else {
                    mScrapViews[viewType].add(scrap);
                }

                if (mRecyclerListener != null) {
                    mRecyclerListener.onMovedToScrapHeap(scrap);
                }
            }
        }
```
用于将一个废弃的View进行缓存，该方法接收一个View参数，当有某个View确定要废弃掉的时候(比如滚动出了屏幕)，就应该调用这个方法来对View进行缓存，RecycleBin当中使用mScrapViews和mCurrentScrap这两个List来存储废弃View。

### getScrapView 
```
        /**
         * @return A view from the ScrapViews collection. These are unordered.
         */
        View getScrapView(int position) {
            final int whichScrap = mAdapter.getItemViewType(position);
            if (whichScrap < 0) {
                return null;
            }
            if (mViewTypeCount == 1) {
                return retrieveFromScrap(mCurrentScrap, position);
            } else if (whichScrap < mScrapViews.length) {
                return retrieveFromScrap(mScrapViews[whichScrap], position);
            }
            return null;
        }
```
用于从废弃缓存中取出一个View，这些废弃缓存中的View是没有顺序可言的，因此getScrapView()方法中的算法也非常简单，就是直接从mCurrentScrap当中获取尾部的一个scrap view进行返回。

## ListView的绘制流程
上面讲到的getView和RecycleBin都只是提供了缓存数据，优化ListView的方法，如何调用这些方法来实现优化的目的呢？要看ListView的绘制过程中，对这些方法的使用了
ListView也是继承自View，所以，他的执行流程也会经过三步：onMeasure()用于测量View的大小，onLayout()用于确定View的布局，onDraw()用于将View绘制到界面上。在ListView当中，onMeasure()并没有什么特殊的地方，因为它终归是一个View，占用的空间最多并且通常也就是整个屏幕。onDraw()在ListView当中也没有什么意义，因为ListView本身并不负责绘制，而是由ListView当中的子元素来进行绘制的。那么ListView大部分的功能其实都是在onLayout()方法中进行。但其实，ListView的onLayout是在其父类AbsListView中实现的
```
    /**
     * Subclasses should NOT override this method but
     *  {@link #layoutChildren()} instead.
     */
    @Override
    protected void onLayout(boolean changed, int l, int t, int r, int b) {
        super.onLayout(changed, l, t, r, b);

        mInLayout = true;

        final int childCount = getChildCount();
        if (changed) {
            for (int i = 0; i < childCount; i++) {
                getChildAt(i).forceLayout();
            }
            mRecycler.markChildrenDirty();
        }

        layoutChildren();
        mInLayout = false;

        mOverscrollMax = (b - t) / OVERSCROLL_LIMIT_DIVISOR;

        // TODO: Move somewhere sane. This doesn't belong in onLayout().
        if (mFastScroll != null) {
            mFastScroll.onItemCountChanged(getChildCount(), mItemCount);
        }
    }
```
这个函数，前面主要判断了一下ListView的大小或者位置有没有发生变化，如果变化了，就强制重绘。然后就是核心的一行：layoutChildren();然而这个函数的实现，又在ListView中。

```
    protected void layoutChildren() {
        final boolean blockLayoutRequests = mBlockLayoutRequests;
        if (blockLayoutRequests) {
            return;
        }

        mBlockLayoutRequests = true;

        try {
            super.layoutChildren();

            invalidate();

            if (mAdapter == null) {
                resetList();
                invokeOnItemScrollListener();
                return;
            }

            final int childrenTop = mListPadding.top;
            final int childrenBottom = mBottom - mTop - mListPadding.bottom;
            final int childCount = getChildCount();

            int index = 0;
            int delta = 0;

            View sel;
            View oldSel = null;
            View oldFirst = null;
            View newSel = null;

            // Remember stuff we will need down below
            switch (mLayoutMode) {
            case LAYOUT_SET_SELECTION:
                index = mNextSelectedPosition - mFirstPosition;
                if (index >= 0 && index < childCount) {
                    newSel = getChildAt(index);
                }
                break;
            case LAYOUT_FORCE_TOP:
            case LAYOUT_FORCE_BOTTOM:
            case LAYOUT_SPECIFIC:
            case LAYOUT_SYNC:
                break;
            case LAYOUT_MOVE_SELECTION:
            default:
                // Remember the previously selected view
                index = mSelectedPosition - mFirstPosition;
                if (index >= 0 && index < childCount) {
                    oldSel = getChildAt(index);
                }

                // Remember the previous first child
                oldFirst = getChildAt(0);

                if (mNextSelectedPosition >= 0) {
                    delta = mNextSelectedPosition - mSelectedPosition;
                }

                // Caution: newSel might be null
                newSel = getChildAt(index + delta);
            }


            boolean dataChanged = mDataChanged;
            if (dataChanged) {
                handleDataChanged();
            }

            // Handle the empty set by removing all views that are visible
            // and calling it a day
            if (mItemCount == 0) {
                resetList();
                invokeOnItemScrollListener();
                return;
            } else if (mItemCount != mAdapter.getCount()) {
                throw new IllegalStateException("The content of the adapter has changed but "
                        + "ListView did not receive a notification. Make sure the content of "
                        + "your adapter is not modified from a background thread, but only from "
                        + "the UI thread. Make sure your adapter calls notifyDataSetChanged() "
                        + "when its content changes. [in ListView(" + getId() + ", " + getClass()
                        + ") with Adapter(" + mAdapter.getClass() + ")]");
            }

            setSelectedPositionInt(mNextSelectedPosition);

            AccessibilityNodeInfo accessibilityFocusLayoutRestoreNode = null;
            View accessibilityFocusLayoutRestoreView = null;
            int accessibilityFocusPosition = INVALID_POSITION;

            // Remember which child, if any, had accessibility focus. This must
            // occur before recycling any views, since that will clear
            // accessibility focus.
            final ViewRootImpl viewRootImpl = getViewRootImpl();
            if (viewRootImpl != null) {
                final View focusHost = viewRootImpl.getAccessibilityFocusedHost();
                if (focusHost != null) {
                    final View focusChild = getAccessibilityFocusedChild(focusHost);
                    if (focusChild != null) {
                        if (!dataChanged || isDirectChildHeaderOrFooter(focusChild)
                                || focusChild.hasTransientState() || mAdapterHasStableIds) {
                            // The views won't be changing, so try to maintain
                            // focus on the current host and virtual view.
                            accessibilityFocusLayoutRestoreView = focusHost;
                            accessibilityFocusLayoutRestoreNode = viewRootImpl
                                    .getAccessibilityFocusedVirtualView();
                        }

                        // If all else fails, maintain focus at the same
                        // position.
                        accessibilityFocusPosition = getPositionForView(focusChild);
                    }
                }
            }

            View focusLayoutRestoreDirectChild = null;
            View focusLayoutRestoreView = null;

            // Take focus back to us temporarily to avoid the eventual call to
            // clear focus when removing the focused child below from messing
            // things up when ViewAncestor assigns focus back to someone else.
            final View focusedChild = getFocusedChild();
            if (focusedChild != null) {
                // TODO: in some cases focusedChild.getParent() == null

                // We can remember the focused view to restore after re-layout
                // if the data hasn't changed, or if the focused position is a
                // header or footer.
                if (!dataChanged || isDirectChildHeaderOrFooter(focusedChild)
                        || focusedChild.hasTransientState() || mAdapterHasStableIds) {
                    focusLayoutRestoreDirectChild = focusedChild;
                    // Remember the specific view that had focus.
                    focusLayoutRestoreView = findFocus();
                    if (focusLayoutRestoreView != null) {
                        // Tell it we are going to mess with it.
                        focusLayoutRestoreView.onStartTemporaryDetach();
                    }
                }
                requestFocus();
            }

            // Pull all children into the RecycleBin.
            // These views will be reused if possible
            final int firstPosition = mFirstPosition;
            final RecycleBin recycleBin = mRecycler;
            if (dataChanged) {
                for (int i = 0; i < childCount; i++) {
                    recycleBin.addScrapView(getChildAt(i), firstPosition+i);
                }
            } else {
                recycleBin.fillActiveViews(childCount, firstPosition);
            }

            // Clear out old views
            detachAllViewsFromParent();
            recycleBin.removeSkippedScrap();

            switch (mLayoutMode) {
            case LAYOUT_SET_SELECTION:
                if (newSel != null) {
                    sel = fillFromSelection(newSel.getTop(), childrenTop, childrenBottom);
                } else {
                    sel = fillFromMiddle(childrenTop, childrenBottom);
                }
                break;
            case LAYOUT_SYNC:
                sel = fillSpecific(mSyncPosition, mSpecificTop);
                break;
            case LAYOUT_FORCE_BOTTOM:
                sel = fillUp(mItemCount - 1, childrenBottom);
                adjustViewsUpOrDown();
                break;
            case LAYOUT_FORCE_TOP:
                mFirstPosition = 0;
                sel = fillFromTop(childrenTop);
                adjustViewsUpOrDown();
                break;
            case LAYOUT_SPECIFIC:
                sel = fillSpecific(reconcileSelectedPosition(), mSpecificTop);
                break;
            case LAYOUT_MOVE_SELECTION:
                sel = moveSelection(oldSel, newSel, delta, childrenTop, childrenBottom);
                break;
            default:
                if (childCount == 0) {
                    if (!mStackFromBottom) {
                        final int position = lookForSelectablePosition(0, true);
                        setSelectedPositionInt(position);
                        sel = fillFromTop(childrenTop);
                    } else {
                        final int position = lookForSelectablePosition(mItemCount - 1, false);
                        setSelectedPositionInt(position);
                        sel = fillUp(mItemCount - 1, childrenBottom);
                    }
                } else {
                    if (mSelectedPosition >= 0 && mSelectedPosition < mItemCount) {
                        sel = fillSpecific(mSelectedPosition,
                                oldSel == null ? childrenTop : oldSel.getTop());
                    } else if (mFirstPosition < mItemCount) {
                        sel = fillSpecific(mFirstPosition,
                                oldFirst == null ? childrenTop : oldFirst.getTop());
                    } else {
                        sel = fillSpecific(0, childrenTop);
                    }
                }
                break;
            }

            // Flush any cached views that did not get reused above
            recycleBin.scrapActiveViews();

            if (sel != null) {
                // The current selected item should get focus if items are
                // focusable.
                if (mItemsCanFocus && hasFocus() && !sel.hasFocus()) {
                    final boolean focusWasTaken = (sel == focusLayoutRestoreDirectChild &&
                            focusLayoutRestoreView != null &&
                            focusLayoutRestoreView.requestFocus()) || sel.requestFocus();
                    if (!focusWasTaken) {
                        // Selected item didn't take focus, but we still want to
                        // make sure something else outside of the selected view
                        // has focus.
                        final View focused = getFocusedChild();
                        if (focused != null) {
                            focused.clearFocus();
                        }
                        positionSelector(INVALID_POSITION, sel);
                    } else {
                        sel.setSelected(false);
                        mSelectorRect.setEmpty();
                    }
                } else {
                    positionSelector(INVALID_POSITION, sel);
                }
                mSelectedTop = sel.getTop();
            } else {
                final boolean inTouchMode = mTouchMode == TOUCH_MODE_TAP
                        || mTouchMode == TOUCH_MODE_DONE_WAITING;
                if (inTouchMode) {
                    // If the user's finger is down, select the motion position.
                    final View child = getChildAt(mMotionPosition - mFirstPosition);
                    if (child != null) {
                        positionSelector(mMotionPosition, child);
                    }
                } else if (mSelectorPosition != INVALID_POSITION) {
                    // If we had previously positioned the selector somewhere,
                    // put it back there. It might not match up with the data,
                    // but it's transitioning out so it's not a big deal.
                    final View child = getChildAt(mSelectorPosition - mFirstPosition);
                    if (child != null) {
                        positionSelector(mSelectorPosition, child);
                    }
                } else {
                    // Otherwise, clear selection.
                    mSelectedTop = 0;
                    mSelectorRect.setEmpty();
                }

                // Even if there is not selected position, we may need to
                // restore focus (i.e. something focusable in touch mode).
                if (hasFocus() && focusLayoutRestoreView != null) {
                    focusLayoutRestoreView.requestFocus();
                }
            }

            // Attempt to restore accessibility focus, if necessary.
            if (viewRootImpl != null) {
                final View newAccessibilityFocusedView = viewRootImpl.getAccessibilityFocusedHost();
                if (newAccessibilityFocusedView == null) {
                    if (accessibilityFocusLayoutRestoreView != null
                            && accessibilityFocusLayoutRestoreView.isAttachedToWindow()) {
                        final AccessibilityNodeProvider provider =
                                accessibilityFocusLayoutRestoreView.getAccessibilityNodeProvider();
                        if (accessibilityFocusLayoutRestoreNode != null && provider != null) {
                            final int virtualViewId = AccessibilityNodeInfo.getVirtualDescendantId(
                                    accessibilityFocusLayoutRestoreNode.getSourceNodeId());
                            provider.performAction(virtualViewId,
                                    AccessibilityNodeInfo.ACTION_ACCESSIBILITY_FOCUS, null);
                        } else {
                            accessibilityFocusLayoutRestoreView.requestAccessibilityFocus();
                        }
                    } else if (accessibilityFocusPosition != INVALID_POSITION) {
                        // Bound the position within the visible children.
                        final int position = MathUtils.constrain(
                                accessibilityFocusPosition - mFirstPosition, 0,
                                getChildCount() - 1);
                        final View restoreView = getChildAt(position);
                        if (restoreView != null) {
                            restoreView.requestAccessibilityFocus();
                        }
                    }
                }
            }

            // Tell focus view we are done mucking with it, if it is still in
            // our view hierarchy.
            if (focusLayoutRestoreView != null
                    && focusLayoutRestoreView.getWindowToken() != null) {
                focusLayoutRestoreView.onFinishTemporaryDetach();
            }
            
            mLayoutMode = LAYOUT_NORMAL;
            mDataChanged = false;
            if (mPositionScrollAfterLayout != null) {
                post(mPositionScrollAfterLayout);
                mPositionScrollAfterLayout = null;
            }
            mNeedSync = false;
            setNextSelectedPositionInt(mSelectedPosition);

            updateScrollIndicators();

            if (mItemCount > 0) {
                checkSelectionChanged();
            }

            invokeOnItemScrollListener();
        } finally {
            if (!blockLayoutRequests) {
                mBlockLayoutRequests = false;
            }
        }
    }
```
太长了，将第一次layout时用到的重要部分单独提取出来
```
  protected void layoutChildren() {
			...
            final int childCount = getChildCount();
			

			...
            boolean dataChanged = mDataChanged;
			...
            if (dataChanged) {
				...
            } else {
                recycleBin.fillActiveViews(childCount, firstPosition);
            }

			...
            switch (mLayoutMode) {
            case LAYOUT_SET_SELECTION:
                ...
            case LAYOUT_SYNC:
                ...
            case LAYOUT_FORCE_BOTTOM:
                ...
            case LAYOUT_FORCE_TOP:
                ...
            case LAYOUT_SPECIFIC:
                ...
            case LAYOUT_MOVE_SELECTION:
                ...
            default:
                if (childCount == 0) {
                    if (!mStackFromBottom) {
                        ...
                        sel = fillFromTop(childrenTop);
                    } else {
                        ...
                    }
                } else {
                    ...
                }
                break;
            }
    }
```
ListView当中目前还没有任何子View，数据都还是由Adapter管理的，并没有展示到界面上，因此getChildCount()方法得到的值肯定是0。接着会根据dataChanged这个布尔型的值来判断执行逻辑，dataChanged只有在数据源发生改变的情况下才会变成true，其它情况都是false，因此这里调用RecycleBin的fillActiveViews()方法。按理来说，调用fillActiveViews()方法是为了将ListView的子View进行缓存的，可是目前ListView中还没有任何的子View，因此这一行暂时还起不了任何作用。
接下来根据mLayoutMode的值来决定布局模式，默认情况下都是普通模式LAYOUT_NORMAL，因此会进入到default语句当中。而下面又会紧接着进行两次if判断，childCount目前是等于0的，并且默认的布局顺序是从上往下，因此会进入到fillFromTop()方法中
```
    private View fillFromTop(int nextTop) {
        mFirstPosition = Math.min(mFirstPosition, mSelectedPosition);
        mFirstPosition = Math.min(mFirstPosition, mItemCount - 1);
        if (mFirstPosition < 0) {
            mFirstPosition = 0;
        }
        return fillDown(mFirstPosition, nextTop);
    }
```
基本上，只是调用了fillDown
```
    private View fillDown(int pos, int nextTop) {
        View selectedView = null;

        int end = (mBottom - mTop);
        if ((mGroupFlags & CLIP_TO_PADDING_MASK) == CLIP_TO_PADDING_MASK) {
            end -= mListPadding.bottom;
        }

        while (nextTop < end && pos < mItemCount) {
            // is this the selected item?
            boolean selected = pos == mSelectedPosition;
            View child = makeAndAddView(pos, nextTop, true, mListPadding.left, selected);

            nextTop = child.getBottom() + mDividerHeight;
            if (selected) {
                selectedView = child;
            }
            pos++;
        }

        setVisibleRangeHint(mFirstPosition, mFirstPosition + getChildCount() - 1);
        return selectedView;
    }
```
这里使用了一个while循环来执行重复逻辑，一开始nextTop的值是第一个子元素顶部距离整个ListView顶部的像素值，pos则是刚刚传入的mFirstPosition的值，而end是ListView底部减去顶部所得的像素值，mItemCount则是Adapter中的元素数量。因此一开始的情况下nextTop必定是小于end值的，并且pos也是小于mItemCount值的。那么每执行一次while循环，pos的值都会加1，并且nextTop也会增加，当nextTop大于等于end时，也就是子元素已经超出当前屏幕了，或者pos大于等于mItemCount时，也就是所有Adapter中的元素都被遍历结束了，就会跳出while循环。
while循环中，主要调用了makeAndAddView()方法
```
    private View makeAndAddView(int position, int y, boolean flow, int childrenLeft,
            boolean selected) {
        View child;


        if (!mDataChanged) {
            // Try to use an existing view for this position
            child = mRecycler.getActiveView(position);
            if (child != null) {
                // Found it -- we're using an existing child
                // This just needs to be positioned
                setupChild(child, position, y, flow, childrenLeft, selected, true);

                return child;
            }
        }

        // Make a new view for this position, or convert an unused view if possible
        child = obtainView(position, mIsScrap);

        // This needs to be positioned and measured
        setupChild(child, position, y, flow, childrenLeft, selected, mIsScrap[0]);

        return child;
    }
```
首先尝试从RecycleBin当中快速获取一个active view，不过很遗憾的是目前RecycleBin当中还没有缓存任何的View，所以这里得到的值肯定是null。那么取得了null之后就会继续向下运行，会调用obtainView()方法来再次尝试获取一个View，这次的obtainView()方法是可以保证一定返回一个View的，于是下面立刻将获取到的View传入到了setupChild()方法当中。这个方法在AbsListView.java中
```
    View obtainView(int position, boolean[] isScrap) {
		...
        final View scrapView = mRecycler.getScrapView(position);
        final View child = mAdapter.getView(position, scrapView, this);
        if (scrapView != null) {
            if (child != scrapView) {
                // Failed to re-bind the data, return scrap to the heap.
                mRecycler.addScrapView(scrapView, position);
            } else {
                isScrap[0] = true;

                // Finish the temporary detach started in addScrapView().
                child.dispatchFinishTemporaryDetach();
            }
        }
		...

        return child;
    }
```
调用了RecycleBin的getScrapView()方法来尝试获取一个废弃缓存中的View，赋值给scrapView,同样的道理，这里肯定是获取不到的，getScrapView()方法会返回一个null。然后调用mAdapter的getView()方法来去获取一个View，赋值给child。如果scrapView不为空，且不等于child，就更新recyclerBin。在首次layout的时候， 必定是每行都需要去调用mAdapter的getView()方法来去获取View，也就是调用到了前面章节介绍到的getView方法。这个View会赋值给child并作为函数结果返回，并最终传入到setupChild()方法当中。其实也就是说，第一次layout过程当中，所有的子View都是调用LayoutInflater的inflate()方法加载出来的，这样就会相对比较耗时，但是不用担心，后面就不会再有这种情况了.接着看setupChild方法。
```
    private void setupChild(View child, int position, int y, boolean flowDown, int childrenLeft,
            boolean selected, boolean recycled) {
        ...
        if ((recycled && !p.forceAdd) || (p.recycledHeaderFooter
                && p.viewType == AdapterView.ITEM_VIEW_TYPE_HEADER_OR_FOOTER)) {
            ...
        } else {
            ...
            addViewInLayout(child, flowDown ? -1 : 0, p, true);
        }
        ...
    }
```
刚才调用obtainView()方法获取到的子元素View，这里调用了addViewInLayout()方法将它添加到了ListView当中。那么根据fillDown()方法中的while循环，会让子元素View将整个ListView控件填满然后就跳出，也就是说即使我们的Adapter中有一千条数据，ListView也只会加载第一屏的数据，剩下的数据反正目前在屏幕上也看不到，所以不会去做多余的加载工作，这样就可以保证ListView中的内容能够迅速展示到屏幕上。
那么到此为止，第一次Layout过程结束。

### 第二次layout
View，在展示到界面上之前都会经历至少两次onMeasure()和两次onLayout()的过程。

还是从layoutChildren()方法开始：
```
protected void layoutChildren() {
        	...
            final int childCount = getChildCount();

            ...
            if (dataChanged) {
                ...
            } else {
                recycleBin.fillActiveViews(childCount, firstPosition);
            }

            // Clear out old views
            detachAllViewsFromParent();
            recycleBin.removeSkippedScrap();

            switch (mLayoutMode) {
            case LAYOUT_SET_SELECTION:
                ...
            case LAYOUT_SYNC:
                ...
            case LAYOUT_FORCE_BOTTOM:
                ...
            case LAYOUT_FORCE_TOP:
                ...
            case LAYOUT_SPECIFIC:
                ...
            case LAYOUT_MOVE_SELECTION:
                ...
            default:
                if (childCount == 0) {
                    ...
                } else {
                    if (mSelectedPosition >= 0 && mSelectedPosition < mItemCount) {
                        sel = fillSpecific(mSelectedPosition,
                                oldSel == null ? childrenTop : oldSel.getTop());
                    } else if (mFirstPosition < mItemCount) {
                        sel = fillSpecific(mFirstPosition,
                                oldFirst == null ? childrenTop : oldFirst.getTop());
                    } else {
                        sel = fillSpecific(0, childrenTop);
                    }
                }
                break;
            }

            ...
    }
```
同样还是调用getChildCount()方法来获取子View的数量，只不过现在得到的值不会再是0了，而是ListView中一屏可以显示的子View数量，因为我们刚刚在第一次Layout过程当中向ListView添加了这么多的子View。下面调用了RecycleBin的fillActiveViews()方法，这次效果可就不一样了，因为目前ListView中已经有子View了，这样所有的子View都会被缓存到RecycleBin的mActiveViews数组当中，后面将会用到它们。
接下来将会是非常非常重要的一个操作，调用了detachAllViewsFromParent()方法。这个方法会将所有ListView当中的子View全部清除掉，从而保证第二次Layout过程不会产生一份重复的数据。这样把已经加载好的View又清除掉，待会还要再重新加载一遍，这不是严重影响效率吗？不用担心，还记得我们刚刚调用了RecycleBin的fillActiveViews()方法来缓存子View吗，待会儿将会直接使用这些缓存好的View来进行加载，而并不会重新执行一遍inflate过程，因此效率方面并不会有什么明显的影响。
在接下来的判断逻辑当中，由于不再等于0了，因此会进入到else语句当中。而else语句中又有三个逻辑判断，第一个逻辑判断不成立，因为默认情况下我们没有选中任何子元素，mSelectedPosition应该等于-1。第二个逻辑判断通常是成立的，因为mFirstPosition的值一开始是等于0的，只要adapter中的数据大于0条件就成立。那么进入到fillSpecific()方法当中，代码如下所示：
```
    private View fillSpecific(int position, int top) {
        boolean tempIsSelected = position == mSelectedPosition;
        View temp = makeAndAddView(position, top, true, mListPadding.left, tempIsSelected);
        // Possibly changed again in fillUp if we add rows above this one.
        mFirstPosition = position;

        View above;
        View below;

        final int dividerHeight = mDividerHeight;
        if (!mStackFromBottom) {
            above = fillUp(position - 1, temp.getTop() - dividerHeight);
            // This will correct for the top of the first view not touching the top of the list
            adjustViewsUpOrDown();
            below = fillDown(position + 1, temp.getBottom() + dividerHeight);
            int childCount = getChildCount();
            if (childCount > 0) {
                correctTooHigh(childCount);
            }
        } else {
            below = fillDown(position + 1, temp.getBottom() + dividerHeight);
            // This will correct for the bottom of the last view not touching the bottom of the list
            adjustViewsUpOrDown();
            above = fillUp(position - 1, temp.getTop() - dividerHeight);
            int childCount = getChildCount();
            if (childCount > 0) {
                 correctTooLow(childCount);
            }
        }

        if (tempIsSelected) {
            return temp;
        } else if (above != null) {
            return above;
        } else {
            return below;
        }
    }
```
fillSpecific()这算是一个新方法了，不过其实它和fillUp()、fillDown()方法功能也是差不多的，主要的区别在于，fillSpecific()方法会优先将指定位置的子View先加载到屏幕上，然后再加载该子View往上以及往下的其它子View。那么由于这里我们传入的position就是第一个子View的位置，于是fillSpecific()方法的作用就基本上和fillDown()方法是差不多的了，核心函数依然是makeAndAddView





[详细分析](http://m.blog.csdn.net/article/details?id=44996879)


## 滑动时不加载图片
[示例](http://www.aichengxu.com/view/507689)
