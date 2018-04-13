## RecycleBin 机制
RecycleBin 是 ListView 能够实现加载成百上千条数据也不会发生 OOM 的机制。RecycleBin 是 AbsListView 的内部类，因此所有 AbsListView 
的子类都能使用 RecycleBin 机制，例如：ListView 、GridView 等。RecycleBin 的主要方法(代码不全)如下所示：
```java
    // 代码为 Android O
    class RecycleBin {
        private RecyclerListener mRecyclerListener;

        /**
         * The position of the first view stored in mActiveViews.
         */
        private int mFirstActivePosition;

        /**
         * Views that were on screen at the start of layout. This array is populated at the start of
         * layout, and at the end of layout all view in mActiveViews are moved to mScrapViews.
         * Views in mActiveViews represent a contiguous range of Views, with position of the first
         * view store in mFirstActivePosition.
         */
        private View[] mActiveViews = new View[0];

        /**
         * Unsorted views that can be used by the adapter as a convert view.
         */
        private ArrayList<View>[] mScrapViews;

        private int mViewTypeCount;

        private ArrayList<View> mCurrentScrap;

        private ArrayList<View> mSkippedScrap;

        private SparseArray<View> mTransientStateViews;
        private LongSparseArray<View> mTransientStateViewsById;

        public void setViewTypeCount(int viewTypeCount) {}

        public void markChildrenDirty() {}

        public boolean shouldRecycleViewType(int viewType) {return viewType >= 0;}

        /**
         * Clears the scrap heap.
         */
        void clear() {clearScrap(scrap);clearTransientStateViews();}

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

        View getTransientStateView(int position) {return null;}

        /**
         * Dumps and fully detaches any currently saved views with transient
         * state.
         */
        void clearTransientStateViews() {}

        /**
         * @return A view from the ScrapViews collection. These are unordered.
         */
        View getScrapView(int position) {}

        /**
         * Puts a view into the list of scrap views.
         * <p>
         * If the list data hasn't changed or the adapter has stable IDs, views
         * with transient state will be preserved for later retrieval.
         *
         * @param scrap The view to add
         * @param position The view's position within its parent
         */
        void addScrapView(View scrap, int position) {}

        private ArrayList<View> getSkippedScrap() {}

        /**
         * Finish the removal of any views that skipped the scrap heap.
         */
        void removeSkippedScrap() {}

        /**
         * Move all views remaining in mActiveViews to mScrapViews.
         */
        void scrapActiveViews() {pruneScrapViews();}

        /**
         * At the end of a layout pass, all temp detached views should either be re-attached or
         * completely detached. This method ensures that any remaining view in the scrap list is
         * fully detached.
         */
        void fullyDetachScrapViews() {}

        /**
         * Makes sure that the size of mScrapViews does not exceed the size of
         * mActiveViews, which can happen if an adapter does not recycle its
         * views. Removes cached transient state views that no longer have
         * transient state.
         */
        private void pruneScrapViews() {}

        /**
         * Puts all views in the scrap heap into the supplied list.
         */
        void reclaimScrapViews(List<View> views) {}

        /**
         * Updates the cache color hint of all known views.
         *
         * @param color The new cache color hint.
         */
        void setCacheColorHint(int color) {}

        private View retrieveFromScrap(ArrayList<View> scrapViews, int position) {return null;}

        private void clearScrap(final ArrayList<View> scrap) {}

        private void clearScrapForRebind(View view) {}

        private void removeDetachedView(View child, boolean animate) {}
    }
```
RecycleBin 中有 5 个主要的方法来实现 View 的回收与复用机制：

* __void fillActiveViews(int, int)__ 方法接收两个参数，第一个参数 childCount 表示要储存的 View 数量（屏幕上的可见 View ），
第二个参数 firstActivePosition 表示 ListView 中第一个可见 View 的 position，RecycleBin 使用 mActiveViews<View>[] 来存储 View，
调用这个方法后，会根据方法的参数将 View 保存到 mActiveViews<View>[] 相应的位置中。

* __View getActiveView(int)__ 方法与 fillActiveViews() 对应，接收一个 position 参数，在方法内部会
根据 position - firstActivePosition 计算出 index，根据 index 从 mActiveViews<View>[] 中获取 View。
注意：一旦从 mActiveViews<View>[] 获取到了 View，该 View 便在 mActiveViews<View>[] 被移除。

* __void addScrapView(View, int)__ 当某个 View(ScrapView) 确定需要回收时（不可见），方法将需要回收的 View 加入回收列表中。在 RecycleBin 中，回收的 View 保存在 mScrapViews 和 mCurrentScrap 中, 在 mScrapViews 内部元素为 `ArrarList<View>` ,同一类 View 被放到 相同的 `ArrarList<View>` 中，传进来的 View 会根据
 position 来计算 ViewType ，根据 ViewType 加入到 ArrarList<View> 中。

* __View getScrapView(int)__ 方法与 addScrapView() 对应，接收一个 position 参数，在方法内部会
根据 position 计算出 ViewType，根据 ViewType 来返回需要复用的 View。

* __setViewTypeCount(int)__ 方法是 addScrapView()，getScrapView() 能正常工作的基础。viewTypeCount 表示 ListView 中 ViewType 的数量。
这个方法会为每一个 ViewType 创建一个 `ArrayList<View>` 保存在 mScrapViews 中。在 Adapter 当中可以重写一个 getViewTypeCount() 来表示 ListView 中有几种类型的数据项，而 setViewTypeCount() 方法的作用就是为每种类型的数据项都单独启用一个 RecycleBin 缓存机制。

__setViewTypeCount(int)方法__
```java
public void setViewTypeCount(int viewTypeCount) {
            //要在 ListView 中显示数据，至少会有一种类型的 ViewType
            if (viewTypeCount < 1) {
                throw new IllegalArgumentException("Can't have a viewTypeCount < 1");
            }
            //noinspection unchecked
            //初始化一个ArrayList<View>[]，数组的 size = viewTypeCount，数组中每个
            //element 为 ArrayList<View>，即：为每一种 ViewType维护一个缓存列表
            ArrayList<View>[] scrapViews = new ArrayList[viewTypeCount];
            for (int i = 0; i < viewTypeCount; i++) {
                scrapViews[i] = new ArrayList<View>();
            }
            mViewTypeCount = viewTypeCount;
            // 刚滑出屏幕的 View(ScrapView) 会根据 ViewType 放进相应的缓存列表
            // mCurrentScrap 对应这个 View(ScrapView) 缓存的列表
            mCurrentScrap = scrapViews[0];// 初始化
            mScrapViews = scrapViews;
        }
```
__addScrapView(View,int)方法__
```java
void addScrapView(View scrap, int position) {
            final AbsListView.LayoutParams lp = (AbsListView.LayoutParams) scrap.getLayoutParams();
            if (lp == null) {
                // Can't recycle, but we don't know anything about the view.
                // Ignore it completely.
                return;
            }

            // The position （the view was removed from） when pulled out of the scrap heap.
            // 当从 View 缓存列表中取出复用时，可以知道 View 被移除时的 position
            lp.scrappedFromPosition = position;

            // Remove but don't scrap header or footer views, or views that
            // should otherwise not be recycled.
            final int viewType = lp.viewType;
            if (!shouldRecycleViewType(viewType)) {
                // 传进来的 View 不需要回收
                // Can't recycle. If it's not a header or footer, which have
                // special handling and should be ignored, then skip the scrap
                // heap and we'll fully detach the view later.
                if (viewType != ITEM_VIEW_TYPE_HEADER_OR_FOOTER) {
                    getSkippedScrap().add(scrap);
                }
                return;
            }

            // 分发onDetachedFromWindow()事件，这句很关键
            // 使用 ListView 的时候，有时需要判断 View 是否从 ListView 移出。
            // 在普通的Layout 中，例如LinearLayou中，会回掉 View 的 onDetachedFromWindow()方法。
            // 但是在低版本 SDK 中使用 ListView，View 移除时，onDetachedFromWindow() 方法不会被调用
            // 在 Android N 及以上版本不存在此问题
            scrap.dispatchStartTemporaryDetach();

            // The the accessibility state of the view may change while temporary
            // detached and we do not allow detached views to fire accessibility
            // events. So we are announcing that the subtree changed giving a chance
            // to clients holding on to a view in this subtree to refresh it.
            // View 状态改变，辅助功能相关的事件通知
            notifyViewAccessibilityStateChangedIfNeeded(
                    AccessibilityEvent.CONTENT_CHANGE_TYPE_SUBTREE);

            // Don't scrap views that have transient state.
            // 不缓存处在过渡状态的 View
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
                    clearScrapForRebind(scrap);
                    getSkippedScrap().add(scrap);
                }
            } else {
                clearScrapForRebind(scrap);// 清除辅助功能相关状态
                if (mViewTypeCount == 1) {
                    mCurrentScrap.add(scrap);// View 加入缓存
                } else {
                    mScrapViews[viewType].add(scrap); // View 加入缓存
                }

                if (mRecyclerListener != null) {
                    // 回调监听
                    mRecyclerListener.onMovedToScrapHeap(scrap);
                }
            }
        }
```

__getScrapView(int)__
```java
View getScrapView(int position) {
            //根据 position 得到 ViewType
            final int whichScrap = mAdapter.getItemViewType(position);
            if (whichScrap < 0) {
                return null;
            }
            //从缓存列表中获取 View
            if (mViewTypeCount == 1) {
                return retrieveFromScrap(mCurrentScrap, position);
            } else if (whichScrap < mScrapViews.length) {
                return retrieveFromScrap(mScrapViews[whichScrap], position);
            }
            return null;
        }
```
__fillActiveViews(int,int)__
```java
void fillActiveViews(int childCount, int firstActivePosition) {
            if (mActiveViews.length < childCount) {
                // 需要重建 mActiveViews[]
                mActiveViews = new View[childCount];
            }
            mFirstActivePosition = firstActivePosition;

            //noinspection MismatchedReadAndWriteOfArray
            final View[] activeViews = mActiveViews;
            for (int i = 0; i < childCount; i++) {
                // i 为 View inflate 的顺序，暂时这样理解
                View child = getChildAt(i); //所有可见 View
                AbsListView.LayoutParams lp = (AbsListView.LayoutParams) child.getLayoutParams();
                // Don't put header or footer views into the scrap heap
                if (lp != null && lp.viewType != ITEM_VIEW_TYPE_HEADER_OR_FOOTER) {
                    // Note:  We do place AdapterView.ITEM_VIEW_TYPE_IGNORE in active views.
                    //        However, we will NOT place them into scrap views.
                    activeViews[i] = child;
                    // Remember the position so that setupChild() doesn't reset state.
                    // The position （the view was removed from） when pulled out of the scrap heap.
                    lp.scrappedFromPosition = firstActivePosition + i; //记录 View 的回收位置
                }
            }
        }
```

