---
layout:     post
title:      RxPermissions源码分析
date:       2017-04-17
summary:    关于RxPermissions源码分析以及其中的一些黑科技
categories: jekyll pixyll
---

Github:[https://github.com/twolight](https://github.com/twolight)

Blog:[https://twolight.github.io/](https://twolight.github.io/)

Email: [twolight88@gmail.com](twolight88@gmail.com)



[RxPermissions](https://github.com/tbruyelle/RxPermissions)是一个Android动态请求权限的开源库.



先来了解如何使用，Demo中有个例子



```java

RxPermissions rxPermissions = new RxPermissions(this);
rxPermissions.setLogging(true);

setContentView(R.layout.act_main);
surfaceView = (SurfaceView) findViewById(R.id.surfaceView);

RxView.clicks(findViewById(R.id.enableCamera))
	// Ask for permissions when button is clicked
	.compose(rxPermissions.ensureEach(Manifest.permission.CAMERA))
	.subscribe(new Action1<Permission>() {
		@Override
		public void call(Permission permission) {

		}
	},
	new Action1<Throwable>() {
		@Override
		public void call(Throwable t) {
			Log.e(TAG, "onError", t);
		}
	},
	new Action0() {
		@Override
		public void call() {
			Log.i(TAG, "OnComplete");
		}
});
    
```



Rxjava的用法这里不做说明，请了解Rxjava的相关文章。就是这么简单，RxPermissions相关代码也并没有出现在Activity的其他地方。`onRequestPermissionsResult`居然没有override，那请求权限的结果在哪里获取。这是我看到源码的第一反应。



难道在库里有一个activity override`onRequestPermissionsResult` 来获取结果？或者捕获了Activity的生命周期，动态插入代码？这是我的一些猜想。



然后进一步深入库源码。库里只有三个文件

`Permission.java`  

`RxPermissions.java`

`RxPermissionsFragment.java`



是的，你没看错，居然有一个`Fragment`，上面的初步猜想就不正确了。到这一步那就可以确定`Fragment`作为一个载体来获取权限申请结果，最后把结果通过Rxjava回调给使用者。

下面来分析一下这个三个类，来看看是如何实现的。

# Permission.java

比较简单，权限Model类，封装了权限，权限结果等

# RxPermissions.java



```java
public class RxPermissions {
  	  ....
          
      private RxPermissionsFragment getRxPermissionsFragment(Activity activity) {
              RxPermissionsFragment rxPermissionsFragment = 	findRxPermissionsFragment(activity);
              boolean isNewInstance = rxPermissionsFragment == null;
              if (isNewInstance) {
                  rxPermissionsFragment = new RxPermissionsFragment();
                  FragmentManager fragmentManager = activity.getFragmentManager();
                  fragmentManager
                          .beginTransaction()
                          .add(rxPermissionsFragment, TAG)
                          .commitAllowingStateLoss();
                  fragmentManager.executePendingTransactions();
              }
              return rxPermissionsFragment;
      }
      private RxPermissionsFragment findRxPermissionsFragment(Activity activity) {
              return (RxPermissionsFragment) 									activity.getFragmentManager().findFragmentByTag(TAG);
      }
  
  	  ....
  
}


```



`RxPermissionsFragment`只是add到Activity中，并没有show，所以看不到`Fragement`，使用`commitAllowingStateLoss`，将不保证UI状态的丢失，这正是我们需要的，因为就是要在`后台`默默的获取权限的结果。



# RxPermissionsFragment.java



```
public class RxPermissionsFragment extends Fragment {
	@Override
    public void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setRetainInstance(true);
    }
    
	@TargetApi(Build.VERSION_CODES.M)
    void requestPermissions(@NonNull String[] permissions) {
        requestPermissions(permissions, PERMISSIONS_REQUEST_CODE);
    }

    @TargetApi(Build.VERSION_CODES.M)
    public void onRequestPermissionsResult(int requestCode, @NonNull String permissions[], @NonNull int[] grantResults) {
        super.onRequestPermissionsResult(requestCode, permissions, grantResults);

        if (requestCode != PERMISSIONS_REQUEST_CODE) return;

        boolean[] shouldShowRequestPermissionRationale = new boolean[permissions.length];

        for (int i = 0; i < permissions.length; i++) {
            shouldShowRequestPermissionRationale[i] = shouldShowRequestPermissionRationale(permissions[i]);
        }

        onRequestPermissionsResult(permissions, grantResults, shouldShowRequestPermissionRationale);
    }
}
```



`setRetainInstance(true)` 增加引用，避免`Fragment`被Java内存回收器回收，在Java中没有被引用的对象会被回收。

调用 `Fragement`的`requestPermissions`方法 ，最终请求权限的结果会回调`onRequestPermissionsResult`方法。然后通过Rxjava把结果发送给订阅者。最终证明之前的猜想正确。















