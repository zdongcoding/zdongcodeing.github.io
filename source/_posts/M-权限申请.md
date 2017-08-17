---
title: Android M-权限基础
date: 2017-08-16 11:18:11
tags: android
---
背景
> 从Android 6.0开始, 用户需要在运行时请求权限, 本文对运行时权限的申请和处理进行介绍, 并讨论了使用运行时权限时新老版本的一些处理.当然只有当项目中的` targetSdkVersion>=23 ` 才需要去申请运行时权限

当targetSdkVersion<23时 运行在Android M系统以上的机器上时google官方默认都是允许的（第三方系统除外，有些第三方还是会提示您是否允许权限）

<!-- more -->
### 1.检查权限
```java
ContextCompat.java
public static int checkSelfPermission(@NonNull Context context, @NonNull String permission) 
    //return  
    PackageManager.PERMISSION_GRANTED  //同意
    PackageManager.PERMISSION_DENIED   // 拒绝

PermissionChecker.java
 public static int checkSelfPermission(@NonNull Context context,
            @NonNull String permission)
    //return  
    public static final int PERMISSION_GRANTED =  PackageManager.PERMISSION_GRANTED;//同意
    public static final int PERMISSION_DENIED =  PackageManager.PERMISSION_DENIED; // 拒绝
    public static final int PERMISSION_DENIED_APP_OP =  PackageManager.PERMISSION_DENIED  - 1; // 当targetSdkVersion<23时 用户去设置界面关闭权限 返回值
```
第一次安装时，当targetSdkVersion<23时 上面两个方法 ` return  PERMISSION_GRANTED `
,否则  ` return  PERMISSION_DENIED `

 两个的区别 ？？？？？？ 先卖个关子！（看注释大概就知道区别了）

### 2.申请权限
```java
ActivityCompat.java
public static void requestPermissions(final @NonNull Activity activity,
            final @NonNull String[] permissions, final @IntRange(from = 0) int requestCode) 
   // 类似startActivityOnResult 方法  有回调方法
void onRequestPermissionsResult(int requestCode, @NonNull String[] permissions,
                @NonNull int[] grantResults)
``` 
只要调用了 requestPermissions  它都会回调 ` onRequestPermissionsResult ` 如果` targetSdkVersion<23 `时 调用ActivityCompat.requestPermissions默认也会回调onRequestPermissionsResult  ` permissions.length=0 ,grantResults.length=0 `
如果` targetSdkVersion>=23 `时**如果允许权限了也还是会回调的**

### 3.禁止询问对话框
```java
ActivityCompat.java
 public static boolean shouldShowRequestPermissionRationale(@NonNull Activity activity,@NonNull String permission) {
        if (Build.VERSION.SDK_INT >= 23) {
            return ActivityCompatApi23.shouldShowRequestPermissionRationale(activity, permission);
        }
        return false;
    }
```
看上面源码` targetSdkVersion<23 `直接返回 false,` targetSdkVersion>=23 ` 默认false 当用户拒绝并且勾选不再询问返回false ,当用户只拒绝返回 true

####  问题一： ContextCompat.checkSelfPermission 和PermissionChecker.checkSelfPermission 的区别？？？
区别在于 ` targetSdkVersion<23 `**用户手动去设置中拒绝权限**情况下 ContextCompat.checkSelfPermission 返回的还是PERMISSION_GRANTED，而PermissionChecker.checkSelfPermission 发回PERMISSION_DENIED_APP_OP。

####  问题二：申请多个权限 多次调用 requestPermissions问题
效果肯定不是你想想的！！！ requestPermissions的调用到onRequestPermissionsResult回调这算一个完整的流程，当多次调用requestPermissions在onRequestPermissionsResult回调之前 只处理第一次请求。


