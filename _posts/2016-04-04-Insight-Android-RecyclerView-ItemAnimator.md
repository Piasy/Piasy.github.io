---
layout: post
title: 深入理解 RecyclerView 系列之二：ItemAnimator
tags:
    - 安卓开发
    - RecyclerView
---

本文继上篇 [ItemDecoration](/2016/03/26/Insight-Android-RecyclerView-ItemDecoration/index.html){:target="_blank"} 之后，是深入理解 RecyclerView 系列的第二篇，关注于 ItemAnimator，主要是分析 [RecyclerView Animators](https://github.com/wasabeef/recyclerview-animators){:target="_blank"} 这个库的原理，然后总结如何自己编写自定义的 ItemAnimator。本文涉及到的完整代码可以在[ Github 获取](https://github.com/Piasy/AndroidPlayground/blob/master/effect/RecyclerViewAdvancedDemo){:target="_blank"}。

## 先看看类结构
+ `DefaultItemAnimator` extends `SimpleItemAnimator` extends `RecyclerView.ItemAnimator`
+ `FadeInAnimator` extends `BaseItemAnimator` extends `SimpleItemAnimator` extends `RecyclerView.ItemAnimator`
+ `RecyclerView.ItemAnimator` 定义了一系列 API 用于开发 item view 的动效
  + `animateDisappearance`, `animateAppearance`, `animatePersistence`, `animateChange` 这4个 API 用来对 item view 进行动画显示
  + `recordPreLayoutInformation`, `recordPostLayoutInformation` 这2个 API 用来记录 item view 在 layout 前后的状态信息，这些信息封装在 `ItemHolderInfo` 或者其子类中，并将传递给上述4个动画API中，以便进行动画展示
  + `runPendingAnimations` 可以用来延迟动画到下一帧，此时就需要在上述4个 API 的实现中返回 `true`，并且自行记录延迟的动画信息，以便在下一帧时执行
  + `dispatchAnimationStarted` 和 `dispatchAnimationFinished` 是用来进行状态同步和事件通知的，子类必须在动画开始时调用 dispatchAnimationStarted，结束时调用 dispatchAnimationFinished，当然如果不展示动画，那就只需要直接调用 dispatchAnimationFinished
+ `SimpleItemAnimator` 则对 `RecyclerView.ItemAnimator` 的 API 进行了一次封装
  + 把父类定义的4个动画 API 转换为了 `animateRemove`, `animateAdd`, `animateMove`, `animateChange` 这4个，为什么这样？这一次封装就把对 preLayoutInfo 和 postLayoutInfo 的处理的公共代码封装了起来，把 ItemHolderInfo 转换为了 left, top, x, y 这样的位置信息，这样，大部分动画只需要根据位置变化信息的实现，专注实现自己的动画逻辑即可，一方面复用了代码，另一方面也更好的践行了单一职责原则
  + 但是如果位置信息对于动画的展示不够，那就需要自己重写 `RecyclerView.ItemAnimator` 的相应动画 API 了
  + 同时也定义了一系列 `dispatch***` 和 `on***` API，用于进行事件回调
+ `DefaultItemAnimator` 是 RecyclerView 包中的一个默认实现，而 `BaseItemAnimator` 则是 RecyclerView Animators 库中 animator 的基类，它们都继承自 `SimpleItemAnimator`，两者具有很大相似性，只分析后者

## BaseItemAnimator
BaseItemAnimator 实现了父类的 `animateRemove`, `animateAdd`, `animateMove`, `animateChange` 这4个 API，而实现方式都是把参数包装一下，放入相应的 animation 列表中，并返回 true，然后在 runPendingAnimations 函数中集中显示动画。为什么要这样呢？因为 recycler view 的变化是随时都可能发生的，而这样的处理就可以把动画的显示按帧对其，即两帧之间的变化，都在下一帧开始时一起处理。但是这样做有什么优势呢？暂时不得而知，DefaultItemAnimator 就是这样处理的。

例如 animateRemove 的实现如下：

``` java
@Override
public boolean animateRemove(final ViewHolder holder) {
    endAnimation(holder);
    preAnimateRemove(holder);
    mPendingRemovals.add(holder);
    return true;
}
```

那么下面重点看看 runPendingAnimations。

``` java
  @Override 
  public void runPendingAnimations() {
    boolean removalsPending = !mPendingRemovals.isEmpty();
    boolean movesPending = !mPendingMoves.isEmpty();
    boolean changesPending = !mPendingChanges.isEmpty();
    boolean additionsPending = !mPendingAdditions.isEmpty();
    if (!removalsPending && !movesPending && !additionsPending && !changesPending) {
      // nothing to animate
      return;
    }
    // First, remove stuff
    for (ViewHolder holder : mPendingRemovals) {
      doAnimateRemove(holder);
    }
    mPendingRemovals.clear();
    // Next, move stuff
    if (movesPending) {
      final ArrayList<MoveInfo> moves = new ArrayList<MoveInfo>();
      moves.addAll(mPendingMoves);
      mMovesList.add(moves);
      mPendingMoves.clear();
      Runnable mover = new Runnable() {
        @Override public void run() {
          for (MoveInfo moveInfo : moves) {
            animateMoveImpl(moveInfo.holder, moveInfo.fromX, moveInfo.fromY, moveInfo.toX,
                moveInfo.toY);
          }
          moves.clear();
          mMovesList.remove(moves);
        }
      };
      if (removalsPending) {
        View view = moves.get(0).holder.itemView;
        ViewCompat.postOnAnimationDelayed(view, mover, getRemoveDuration());
      } else {
        mover.run();
      }
    }
    // Next, change stuff, to run in parallel with move animations
    if (changesPending) {
      final ArrayList<ChangeInfo> changes = new ArrayList<ChangeInfo>();
      changes.addAll(mPendingChanges);
      mChangesList.add(changes);
      mPendingChanges.clear();
      Runnable changer = new Runnable() {
        @Override public void run() {
          for (ChangeInfo change : changes) {
            animateChangeImpl(change);
          }
          changes.clear();
          mChangesList.remove(changes);
        }
      };
      if (removalsPending) {
        ViewHolder holder = changes.get(0).oldHolder;
        ViewCompat.postOnAnimationDelayed(holder.itemView, changer, getRemoveDuration());
      } else {
        changer.run();
      }
    }
    // Next, add stuff
    if (additionsPending) {
      final ArrayList<ViewHolder> additions = new ArrayList<ViewHolder>();
      additions.addAll(mPendingAdditions);
      mAdditionsList.add(additions);
      mPendingAdditions.clear();
      Runnable adder = new Runnable() {
        public void run() {
          for (ViewHolder holder : additions) {
            doAnimateAdd(holder);
          }
          additions.clear();
          mAdditionsList.remove(additions);
        }
      };
      if (removalsPending || movesPending || changesPending) {
        long removeDuration = removalsPending ? getRemoveDuration() : 0;
        long moveDuration = movesPending ? getMoveDuration() : 0;
        long changeDuration = changesPending ? getChangeDuration() : 0;
        long totalDelay = removeDuration + Math.max(moveDuration, changeDuration);
        View view = additions.get(0).itemView;
        ViewCompat.postOnAnimationDelayed(view, adder, totalDelay);
      } else {
        adder.run();
      }
    }
  }
```

在这里，remove 最先执行，remove 执行完毕后，再同时开始 move 和 change，而它俩都结束后，最后再执行 add。BaseItemAnimator 对 add 和 remove 这两个动画的播放进行了再一次的封装，定义了 `animateAddImpl` 和 `animateRemoveImpl` 这两个 API，以及 `preAnimateAddImpl` 和 `preAnimateRemoveImpl` 供动画开始前进行需要的操作，而这个库内置的多种 animator，都只是在这两个 API 中实现了不同的出现和消失的逻辑。add 和 remove 这两个动画的进一步封装，再次简化了编写 animator 的代码，具体的 animator 只需要专注于自己的动画显示逻辑即可。而 move 和 change 这两类动画，则是直接使用了 DefaultItemAnimator 的代码，move 就是通过 TranslationX 和 TranslationY 把 item view 从老位置移动到新位置，change 就是通过 TranslationX, setTranslationY 和 alpha 来完成内容的改变效果。

## 自定义的 move 和 change 实现
这部分有一篇不错的文章：[InstaMaterial - RecyclerView animations done right ](http://frogermcs.github.io/instamaterial-recyclerview-animations-done-right){:target="_blank"}。

基本原理还是 RecyclerView.ItemAnimator 提供的 API，canReuseUpdatedViewHolder 控制动效时是否创建新的 ViewHolder，recordPreLayoutInformation/recordPostLayoutInformation 用来在 layout 之前/后记录需要的信息，animateChange/animateMove 来实现具体的动画逻辑，而这时可能会需要 layout 前后记录的信息。

在这篇文章中，作者就是在 recordPreLayoutInformation 中把需要的信息记录在了自定义的 ItemHolderInfo 中，并且在 animateChange 使用记录的信息进行动画的显示。这个过程并没有什么难点，主要还是动画的设计，以及实现的效率和稳定性，例如避免反复创建不必要的对象，避免出现闪退等。具体的例子可以看这篇文章。

## Talk is cheap, show me the code
好了，说了这么多，还是需要一个完整的 demo 才接地气。结合自家产品的需求，这个 demo 中将要实现这样的效果：列表可滑动，新数据加入时，如果正在滑动，则不自动滚到最新（最底部），如果超过5秒不滑动，则自动滚动到最新，如果本来就在最新，则动效塞入新数据（fade in），列表中的数据15秒自动消失，fade out 消失动效，每个 item 点击之后有一个“桃心放大”的效果，模仿的是上一节中那篇文章的效果。关于滑动检测、自动滑动的内容，将在下一篇中展开，本篇聚焦于 ItemAnimator，所以这一版本包含的是增加、移除、点击的动效。

<img src="/img/201604/recycler-animator-demo.gif" alt="整体效果" style="height: 400px;">

增加和移除的动效，直接继承自 RecyclerView Animators 库的 FadeInAnimator 就可以实现了，而点击动效，则直接借用了 InstaMaterial 的部分代码。在这个过程中，还发现了 RecyclerView Animators 的一个 bug，它的所有内置 animator 的实现，change 动效都不起作用，而且还会影响其他类型的动效展示，原因比较简单，重载了 animateChange，但是既没有调用 super 的实现，也没有调用 dispatchAnimationFinished，具体可以查看[这个 issue 下的评论](https://github.com/wasabeef/recyclerview-animators/issues/35#issuecomment-205198654){:target="_blank"}。

从最后的效果图中我们可以看到，如果 item view 快要消失时，我们点击了，播放点击动效之后，item 的消失会有闪烁的问题，这个问题本篇先暂且放下，后续的文章中会进行分析和解决。
