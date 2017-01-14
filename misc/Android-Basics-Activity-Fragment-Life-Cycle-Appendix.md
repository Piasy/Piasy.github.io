---
layout: page
title: 安卓基础：Activity/Fragment 生命周期（附录）
permalink: /2017/01/14/Android-Basics-Activity-Fragment-Life-Cycle-Appendix/
---

## 附录：完整 logcat

### 启动单个 Activity + Fragment

~~~ java
01-08 13:51:12.594  CompatAActivity.onCreate / →☐
01-08 13:51:12.610  CompatAActivity.onCreate / ☐→

01-08 13:51:12.645  CompatAFragment.onInflate / →☐
01-08 13:51:12.646  CompatAFragment.onInflate / ☐→
01-08 13:51:12.646  CompatAFragment.onAttach / →☐
01-08 13:51:12.646  CompatAFragment.onAttach / ☐→

01-08 13:51:12.646  CompatAActivity.onAttachFragment / →☐
01-08 13:51:12.646  CompatAActivity.onAttachFragment / ☐→

01-08 13:51:12.646  CompatAFragment.onCreate / →☐
01-08 13:51:12.647  CompatAFragment.onCreate / ☐→
01-08 13:51:12.649  CompatAFragment.onCreateView / →☐
01-08 13:51:12.649  CompatAFragment.onCreateView / ☐→
01-08 13:51:12.650  CompatAFragment.onViewCreated / →☐
01-08 13:51:12.650  CompatAFragment.onViewCreated / ☐→

01-08 13:51:12.653  CompatAActivity.onContentChanged / →☐
01-08 13:51:12.654  CompatAActivity.onContentChanged / ☐→

01-08 13:51:12.657  CompatAActivity.onStart / →☐
01-08 13:51:12.657  CompatAFragment.onActivityCreated / →☐
01-08 13:51:12.658  CompatAFragment.onActivityCreated / ☐→
01-08 13:51:12.658  CompatAFragment.onViewStateRestored / →☐
01-08 13:51:12.658  CompatAFragment.onViewStateRestored / ☐→
01-08 13:51:12.658  CompatAFragment.onStart / →☐
01-08 13:51:12.658  CompatAFragment.onStart / ☐→
01-08 13:51:12.659  CompatAActivity.onStart / ☐→

01-08 13:51:12.659  CompatAActivity.onPostCreate / →☐
01-08 13:51:12.659  CompatAActivity.onPostCreate / ☐→
01-08 13:51:12.661  CompatAActivity.onResume / →☐
01-08 13:51:12.662  CompatAActivity.onResume / ☐→

01-08 13:51:12.662  CompatAActivity.onPostResume / →☐
01-08 13:51:12.662  CompatAActivity.onResumeFragments / →☐
01-08 13:51:12.662  CompatAFragment.onResume / →☐
01-08 13:51:12.662  CompatAFragment.onResume / ☐→
01-08 13:51:12.662  CompatAActivity.onResumeFragments / ☐→
01-08 13:51:12.662  CompatAActivity.onPostResume / ☐→

01-08 13:51:12.690  CompatAActivity.onAttachedToWindow / →☐
01-08 13:51:12.690  CompatAActivity.onAttachedToWindow / ☐→
01-08 13:51:12.766  CompatAActivity.onCreateOptionsMenu / →☐
01-08 13:51:12.767  CompatAActivity.onCreateOptionsMenu / ☐→
01-08 13:51:12.768  CompatAFragment.onCreateOptionsMenu / →☐
01-08 13:51:12.768  CompatAFragment.onCreateOptionsMenu / ☐→
01-08 13:51:12.768  CompatAActivity.onPrepareOptionsMenu / →☐
01-08 13:51:12.768  CompatAActivity.onPrepareOptionsMenu / ☐→
01-08 13:51:12.768  CompatAFragment.onPrepareOptionsMenu / →☐
01-08 13:51:12.768  CompatAFragment.onPrepareOptionsMenu / ☐→
01-08 13:51:12.772  CompatAActivity.onPrepareOptionsMenu / →☐
01-08 13:51:12.772  CompatAActivity.onPrepareOptionsMenu / ☐→
01-08 13:51:12.772  CompatAFragment.onPrepareOptionsMenu / →☐
01-08 13:51:12.772  CompatAFragment.onPrepareOptionsMenu / ☐→

01-08 13:51:13.151  CompatAActivity.onWindowFocusChanged / →☐
01-08 13:51:13.151  CompatAActivity.onWindowFocusChanged / ☐→
~~~

### AActivity 启动 BActivity

~~~ java
// 启动 B
01-08 14:02:21.237  CompatAActivity.onUserInteraction / →☐
01-08 14:02:21.237  CompatAActivity.onUserInteraction / ☐→
01-08 14:02:21.431  CompatAActivity.onUserInteraction / →☐
01-08 14:02:21.431  CompatAActivity.onUserInteraction / ☐→
01-08 14:02:21.431  CompatAActivity.onUserLeaveHint / →☐
01-08 14:02:21.432  CompatAActivity.onUserLeaveHint / ☐→

01-08 14:02:21.432  CompatAActivity.onPause / →☐
01-08 14:02:21.432  CompatAFragment.onPause / →☐
01-08 14:02:21.432  CompatAFragment.onPause / ☐→
01-08 14:02:21.432  CompatAActivity.onPause / ☐→

01-08 14:02:21.436  CompatAActivity.onWindowFocusChanged / →☐
01-08 14:02:21.436  CompatAActivity.onWindowFocusChanged / ☐→

01-08 14:02:21.446  CompatBActivity.onCreate / →☐
01-08 14:02:21.447  CompatBActivity.onCreate / ☐→

01-08 14:02:21.470  CompatBFragment.onInflate / →☐
01-08 14:02:21.470  CompatBFragment.onInflate / ☐→
01-08 14:02:21.471  CompatBFragment.onAttach / →☐
01-08 14:02:21.472  CompatBFragment.onAttach / ☐→

01-08 14:02:21.473  CompatBActivity.onAttachFragment / →☐
01-08 14:02:21.473  CompatBActivity.onAttachFragment / ☐→

01-08 14:02:21.473  CompatBFragment.onCreate / →☐
01-08 14:02:21.473  CompatBFragment.onCreate / ☐→
01-08 14:02:21.474  CompatBFragment.onCreateView / →☐
01-08 14:02:21.477  CompatBFragment.onCreateView / ☐→
01-08 14:02:21.477  CompatBFragment.onViewCreated / →☐
01-08 14:02:21.477  CompatBFragment.onViewCreated / ☐→

01-08 14:02:21.478  CompatBActivity.onContentChanged / →☐
01-08 14:02:21.478  CompatBActivity.onContentChanged / ☐→

01-08 14:02:21.479  CompatBActivity.onStart / →☐
01-08 14:02:21.479  CompatBFragment.onActivityCreated / →☐
01-08 14:02:21.479  CompatBFragment.onActivityCreated / ☐→
01-08 14:02:21.479  CompatBFragment.onViewStateRestored / →☐
01-08 14:02:21.479  CompatBFragment.onViewStateRestored / ☐→
01-08 14:02:21.480  CompatBFragment.onStart / →☐
01-08 14:02:21.480  CompatBFragment.onStart / ☐→
01-08 14:02:21.480  CompatBActivity.onStart / ☐→

01-08 14:02:21.481  CompatBActivity.onPostCreate / →☐
01-08 14:02:21.481  CompatBActivity.onPostCreate / ☐→

01-08 14:02:21.483  CompatBActivity.onResume / →☐
01-08 14:02:21.484  CompatBActivity.onResume / ☐→

01-08 14:02:21.485  CompatBActivity.onPostResume / →☐
01-08 14:02:21.487  CompatBActivity.onResumeFragments / →☐
01-08 14:02:21.488  CompatBFragment.onResume / →☐
01-08 14:02:21.489  CompatBFragment.onResume / ☐→
01-08 14:02:21.489  CompatBActivity.onResumeFragments / ☐→
01-08 14:02:21.489  CompatBActivity.onPostResume / ☐→

01-08 14:02:21.577  CompatBActivity.onAttachedToWindow / →☐
01-08 14:02:21.577  CompatBActivity.onAttachedToWindow / ☐→

01-08 14:02:21.642  CompatBActivity.onCreateOptionsMenu / →☐
01-08 14:02:21.643  CompatBActivity.onCreateOptionsMenu / ☐→
01-08 14:02:21.643  CompatBFragment.onCreateOptionsMenu / →☐
01-08 14:02:21.643  CompatBFragment.onCreateOptionsMenu / ☐→
01-08 14:02:21.644  CompatBActivity.onPrepareOptionsMenu / →☐
01-08 14:02:21.644  CompatBActivity.onPrepareOptionsMenu / ☐→
01-08 14:02:21.645  CompatBFragment.onPrepareOptionsMenu / →☐
01-08 14:02:21.645  CompatBFragment.onPrepareOptionsMenu / ☐→
01-08 14:02:21.646  CompatBActivity.onPrepareOptionsMenu / →☐
01-08 14:02:21.646  CompatBActivity.onPrepareOptionsMenu / ☐→
01-08 14:02:21.646  CompatBFragment.onPrepareOptionsMenu / →☐
01-08 14:02:21.646  CompatBFragment.onPrepareOptionsMenu / ☐→

01-08 14:02:21.646  CompatBActivity.onWindowFocusChanged / →☐
01-08 14:02:21.646  CompatBActivity.onWindowFocusChanged / ☐→

01-08 14:02:22.101  CompatAActivity.onSaveInstanceState / →☐
01-08 14:02:22.104  CompatAFragment.onSaveInstanceState / →☐
01-08 14:02:22.104  CompatAFragment.onSaveInstanceState / ☐→
01-08 14:02:22.104  CompatAActivity.onSaveInstanceState / ☐→

01-08 14:02:22.109  CompatAActivity.onStop / →☐
01-08 14:02:22.110  CompatAFragment.onStop / →☐
01-08 14:02:22.110  CompatAFragment.onStop / ☐→
01-08 14:02:22.110  CompatAActivity.onStop / ☐→

// 返回
01-08 14:06:06.037  CompatBActivity.onUserInteraction / →☐
01-08 14:06:06.038  CompatBActivity.onUserInteraction / ☐→
01-08 14:06:06.121  CompatBActivity.onUserInteraction / →☐
01-08 14:06:06.121  CompatBActivity.onUserInteraction / ☐→

01-08 14:06:06.126  CompatBActivity.onPause / →☐
01-08 14:06:06.127  CompatBFragment.onPause / →☐
01-08 14:06:06.127  CompatBFragment.onPause / ☐→
01-08 14:06:06.127  CompatBActivity.onPause / ☐→

01-08 14:06:06.131  CompatAActivity.onRestart / →☐
01-08 14:06:06.131  CompatAActivity.onRestart / ☐→

01-08 14:06:06.133  CompatAActivity.onStart / →☐
01-08 14:06:06.133  CompatAFragment.onStart / →☐
01-08 14:06:06.133  CompatAFragment.onStart / ☐→
01-08 14:06:06.134  CompatAActivity.onStart / ☐→

01-08 14:06:06.134  CompatAActivity.onResume / →☐
01-08 14:06:06.135  CompatAActivity.onResume / ☐→

01-08 14:06:06.135  CompatAActivity.onPostResume / →☐
01-08 14:06:06.136  CompatAActivity.onResumeFragments / →☐
01-08 14:06:06.136  CompatAFragment.onResume / →☐
01-08 14:06:06.136  CompatAFragment.onResume / ☐→
01-08 14:06:06.136  CompatAActivity.onResumeFragments / ☐→
01-08 14:06:06.136  CompatAActivity.onPostResume / ☐→

01-08 14:06:06.184  CompatAActivity.onWindowFocusChanged / →☐
01-08 14:06:06.184  CompatAActivity.onWindowFocusChanged / ☐→

01-08 14:06:06.241  CompatBActivity.onWindowFocusChanged / →☐
01-08 14:06:06.242  CompatBActivity.onWindowFocusChanged / ☐→

01-08 14:06:06.519  CompatBActivity.onStop / →☐
01-08 14:06:06.522  CompatBFragment.onStop / →☐
01-08 14:06:06.522  CompatBFragment.onStop / ☐→
01-08 14:06:06.522  CompatBActivity.onStop / ☐→

01-08 14:06:06.522  CompatBActivity.onDestroy / →☐
01-08 14:06:06.523  CompatBFragment.onDestroyView / →☐
01-08 14:06:06.523  CompatBFragment.onDestroyView / ☐→
01-08 14:06:06.524  CompatBFragment.onDestroy / →☐
01-08 14:06:06.525  CompatBFragment.onDestroy / ☐→
01-08 14:06:06.526  CompatBFragment.onDetach / →☐
01-08 14:06:06.526  CompatBFragment.onDetach / ☐→
01-08 14:06:06.526  CompatBActivity.onDestroy / ☐→

01-08 14:06:06.526  CompatBActivity.onDetachedFromWindow / →☐
01-08 14:06:06.527  CompatBActivity.onDetachedFromWindow / ☐→
~~~

### singleTop 测试

~~~ java
// 启动 Main
01-07 23:42:39.743 3246-3246/com.github.piasy.taskdemo D/TaskDemo: Main onCreate
01-07 23:42:39.757 3246-3246/com.github.piasy.taskdemo D/TaskDemo: Main onStart
01-07 23:42:39.758 3246-3246/com.github.piasy.taskdemo D/TaskDemo: Main onResume
// 启动 SingleTop
01-07 23:42:43.965 3246-3246/com.github.piasy.taskdemo D/TaskDemo: Main onPause
01-07 23:42:43.982 3246-3246/com.github.piasy.taskdemo D/TaskDemo: SingleTop onCreate
01-07 23:42:43.990 3246-3246/com.github.piasy.taskdemo D/TaskDemo: SingleTop onStart
01-07 23:42:43.991 3246-3246/com.github.piasy.taskdemo D/TaskDemo: SingleTop onResume
01-07 23:42:44.484 3246-3246/com.github.piasy.taskdemo D/TaskDemo: Main onStop
// 再次启动 SingleTop
01-07 23:42:49.532 3246-3246/com.github.piasy.taskdemo D/TaskDemo: SingleTop onPause
01-07 23:42:49.532 3246-3246/com.github.piasy.taskdemo D/TaskDemo: SingleTop onNewIntent
01-07 23:42:49.532 3246-3246/com.github.piasy.taskdemo D/TaskDemo: SingleTop onResume
// 返回
01-07 23:42:54.369 3246-3246/com.github.piasy.taskdemo D/TaskDemo: SingleTop onPause
01-07 23:42:54.374 3246-3246/com.github.piasy.taskdemo D/TaskDemo: Main onStart
01-07 23:42:54.375 3246-3246/com.github.piasy.taskdemo D/TaskDemo: Main onResume
01-07 23:42:54.767 3246-3246/com.github.piasy.taskdemo D/TaskDemo: SingleTop onStop
01-07 23:42:54.767 3246-3246/com.github.piasy.taskdemo D/TaskDemo: SingleTop onDestroy
// Home 键
01-07 23:43:00.034 3246-3246/com.github.piasy.taskdemo D/TaskDemo: Main onPause
01-07 23:43:00.429 3246-3246/com.github.piasy.taskdemo D/TaskDemo: Main onStop
~~~

### 非 support 组件的测试

~~~ java
// 启动 A
01-08 15:22:03.079  AActivity.onCreate / →☐
01-08 15:22:03.079  AActivity.onCreate / ☐→

01-08 15:22:03.082  AFragment.onInflate / →☐
01-08 15:22:03.083  AFragment.onInflate / ☐→
01-08 15:22:03.083  AFragment.onAttach / →☐
01-08 15:22:03.083  AFragment.onAttach / ☐→

01-08 15:22:03.083  AActivity.onAttachFragment / →☐
01-08 15:22:03.083  AActivity.onAttachFragment / ☐→

01-08 15:22:03.084  AFragment.onCreate / →☐
01-08 15:22:03.084  AFragment.onCreate / ☐→
01-08 15:22:03.084  AFragment.onCreateView / →☐
01-08 15:22:03.084  AFragment.onCreateView / ☐→
01-08 15:22:03.085  AFragment.onViewCreated / →☐
01-08 15:22:03.085  AFragment.onViewCreated / ☐→

01-08 15:22:03.086  AActivity.onContentChanged / →☐
01-08 15:22:03.087  AActivity.onContentChanged / ☐→
01-08 15:22:03.087  AFragment.onActivityCreated / →☐
01-08 15:22:03.087  AFragment.onActivityCreated / ☐→
01-08 15:22:03.087  AFragment.onViewStateRestored / →☐
01-08 15:22:03.087  AFragment.onViewStateRestored / ☐→

01-08 15:22:03.089  AActivity.onStart / →☐
01-08 15:22:03.089  AActivity.onStart / ☐→

01-08 15:22:03.089  AFragment.onStart / →☐
01-08 15:22:03.089  AFragment.onStart / ☐→

01-08 15:22:03.089  AActivity.onPostCreate / →☐
01-08 15:22:03.089  AActivity.onPostCreate / ☐→

01-08 15:22:03.090  AActivity.onResume / →☐
01-08 15:22:03.091  AActivity.onResume / ☐→

01-08 15:22:03.092  AFragment.onResume / →☐
01-08 15:22:03.092  AFragment.onResume / ☐→

01-08 15:22:03.092  AActivity.onPostResume / →☐
01-08 15:22:03.092  AActivity.onPostResume / ☐→

01-08 15:22:03.107  AActivity.onAttachedToWindow / →☐
01-08 15:22:03.107  AActivity.onAttachedToWindow / ☐→
01-08 15:22:04.697  AActivity.onWindowFocusChanged / →☐
01-08 15:22:04.697  AActivity.onWindowFocusChanged / ☐→

// 启动 B
01-08 15:23:26.722  AActivity.onUserInteraction / →☐
01-08 15:23:26.722  AActivity.onUserInteraction / ☐→
01-08 15:23:26.833  AActivity.onUserInteraction / →☐
01-08 15:23:26.833  AActivity.onUserInteraction / ☐→
01-08 15:23:26.834  AActivity.onUserLeaveHint / →☐
01-08 15:23:26.834  AActivity.onUserLeaveHint / ☐→

01-08 15:23:26.834  AFragment.onPause / →☐
01-08 15:23:26.834  AFragment.onPause / ☐→

01-08 15:23:26.834  AActivity.onPause / →☐
01-08 15:23:26.834  AActivity.onPause / ☐→

01-08 15:23:26.841  AActivity.onWindowFocusChanged / →☐
01-08 15:23:26.841  AActivity.onWindowFocusChanged / ☐→

01-08 15:23:26.854  BActivity.onCreate / →☐
01-08 15:23:26.854  BActivity.onCreate / ☐→

01-08 15:23:26.858  BFragment.onInflate / →☐
01-08 15:23:26.858  BFragment.onInflate / ☐→
01-08 15:23:26.859  BFragment.onAttach / →☐
01-08 15:23:26.859  BFragment.onAttach / ☐→

01-08 15:23:26.860  BActivity.onAttachFragment / →☐
01-08 15:23:26.860  BActivity.onAttachFragment / ☐→

01-08 15:23:26.860  BFragment.onCreate / →☐
01-08 15:23:26.861  BFragment.onCreate / ☐→
01-08 15:23:26.861  BFragment.onCreateView / →☐
01-08 15:23:26.861  BFragment.onCreateView / ☐→
01-08 15:23:26.861  BFragment.onViewCreated / →☐
01-08 15:23:26.862  BFragment.onViewCreated / ☐→

01-08 15:23:26.862  BActivity.onContentChanged / →☐
01-08 15:23:26.862  BActivity.onContentChanged / ☐→
01-08 15:23:26.862  BFragment.onActivityCreated / →☐
01-08 15:23:26.862  BFragment.onActivityCreated / ☐→
01-08 15:23:26.862  BFragment.onViewStateRestored / →☐
01-08 15:23:26.862  BFragment.onViewStateRestored / ☐→

01-08 15:23:26.864  BActivity.onStart / →☐
01-08 15:23:26.864  BActivity.onStart / ☐→

01-08 15:23:26.864  BFragment.onStart / →☐
01-08 15:23:26.865  BFragment.onStart / ☐→

01-08 15:23:26.865  BActivity.onPostCreate / →☐
01-08 15:23:26.865  BActivity.onPostCreate / ☐→

01-08 15:23:26.868  BActivity.onResume / →☐
01-08 15:23:26.870  BActivity.onResume / ☐→

01-08 15:23:26.870  BFragment.onResume / →☐
01-08 15:23:26.870  BFragment.onResume / ☐→

01-08 15:23:26.871  BActivity.onPostResume / →☐
01-08 15:23:26.871  BActivity.onPostResume / ☐→

01-08 15:23:26.894  BActivity.onAttachedToWindow / →☐
01-08 15:23:26.894  BActivity.onAttachedToWindow / ☐→
01-08 15:23:27.058  BActivity.onWindowFocusChanged / →☐
01-08 15:23:27.058  BActivity.onWindowFocusChanged / ☐→
01-08 15:23:27.430  AActivity.onSaveInstanceState / →☐
01-08 15:23:27.432  AFragment.onSaveInstanceState / →☐
01-08 15:23:27.433  AFragment.onSaveInstanceState / ☐→
01-08 15:23:27.434  AActivity.onSaveInstanceState / ☐→

01-08 15:23:27.438  AFragment.onStop / →☐
01-08 15:23:27.438  AFragment.onStop / ☐→

01-08 15:23:27.439  AActivity.onStop / →☐
01-08 15:23:27.439  AActivity.onStop / ☐→

// B 返回
01-08 15:25:18.988  BActivity.onUserInteraction / →☐
01-08 15:25:18.989  BActivity.onUserInteraction / ☐→
01-08 15:25:19.058  BActivity.onUserInteraction / →☐
01-08 15:25:19.058  BActivity.onUserInteraction / ☐→

01-08 15:25:19.067  BFragment.onPause / →☐
01-08 15:25:19.067  BFragment.onPause / ☐→

01-08 15:25:19.068  BActivity.onPause / →☐
01-08 15:25:19.068  BActivity.onPause / ☐→

01-08 15:25:19.077  AActivity.onRestart / →☐
01-08 15:25:19.077  AActivity.onRestart / ☐→

01-08 15:25:19.078  AActivity.onStart / →☐
01-08 15:25:19.078  AActivity.onStart / ☐→

01-08 15:25:19.079  AFragment.onStart / →☐
01-08 15:25:19.079  AFragment.onStart / ☐→

01-08 15:25:19.081  AActivity.onResume / →☐
01-08 15:25:19.083  AActivity.onResume / ☐→

01-08 15:25:19.084  AFragment.onResume / →☐
01-08 15:25:19.086  AFragment.onResume / ☐→

01-08 15:25:19.086  AActivity.onPostResume / →☐
01-08 15:25:19.086  AActivity.onPostResume / ☐→

01-08 15:25:19.128  AActivity.onWindowFocusChanged / →☐
01-08 15:25:19.129  AActivity.onWindowFocusChanged / ☐→
01-08 15:25:19.184  BActivity.onWindowFocusChanged / →☐
01-08 15:25:19.184  BActivity.onWindowFocusChanged / ☐→

01-08 15:25:19.456  BFragment.onStop / →☐
01-08 15:25:19.456  BFragment.onStop / ☐→

01-08 15:25:19.457  BActivity.onStop / →☐
01-08 15:25:19.457  BActivity.onStop / ☐→

01-08 15:25:19.457  BFragment.onDestroyView / →☐
01-08 15:25:19.458  BFragment.onDestroyView / ☐→
01-08 15:25:19.458  BFragment.onDestroy / →☐
01-08 15:25:19.459  BFragment.onDestroy / ☐→
01-08 15:25:19.459  BFragment.onDetach / →☐
01-08 15:25:19.459  BFragment.onDetach / ☐→

01-08 15:25:19.460  BActivity.onDestroy / →☐
01-08 15:25:19.460  BActivity.onDestroy / ☐→

01-08 15:25:19.460  BActivity.onDetachedFromWindow / →☐
01-08 15:25:19.461  BActivity.onDetachedFromWindow / ☐→
~~~
