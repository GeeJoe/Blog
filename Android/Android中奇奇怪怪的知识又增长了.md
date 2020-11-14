# Android 中奇奇怪怪的知识又增长了



### 1. EditText 被销毁了但是软键盘还没有消失

**原因：** window 中其他 view 获取了焦点

**解决办法：** 设置 `Window` 的 `softInputMode` 为 `SOFT_INPUT_STATE_ALWAYS_HIDDEN`，意为：当 `Window` 获取焦点时，不自动弹出软键盘

### 2. 当使用 PackageManager 的时候发生 TransactionTooLargeException 
**原因：** 当我们使用 `PackageManger` 获取数据的时候（比如 `getPackageInfo()`、`getInstalledPackages()` 等），可能会发生崩溃，原因是调用这些方法是跨进程通信的，而 Android 中的 Binder 通信机制是有 1MB 大小限制的，所以如果上述方法返回的数据超过 1MB 就会发生 `TransactionTooLargeException`； 

**解决办法：** 调用上述方法时，通过传入一些限制参数，比如 FLAG 来控制返回数据的大小，详见 [Stackoverflow](https://stackoverflow.com/questions/24253976/android-package-manager-has-died-with-transactiontoolargeexception)

### 3. Android 10 之后，在 xml 中，gradient 默认的 angle 是 270° (从上往下)
- xml 中 `<gradient>` 标签 angle = 0 --> 从左往右，angle = 90 --> 从下往上，逆时针以此类推
- Android 10 以下，在 xml 中写 `<gradient>` 标签的时候，渐变色的默认方向是从左往右（不指定 angle 的时候默认值是 0）
- Android 10 及以上，`<gradient>` 不指定 angle 默认值是 270 （从上往下）
> 官方都没有提到过这个变动，好坑

### 4. 