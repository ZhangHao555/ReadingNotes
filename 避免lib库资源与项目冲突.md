## 一 明确lib库资源的作用域
android 官网介绍 https://developer.android.google.cn/studio/projects/android-library#PrivateResources

库中的所有资源在默认情况下均处于公开状态。要将所有资源隐式设为私有，您必须至少将一个特定属性定义为公开。资源包括您项目的 res/ 目录中的所有文件，例如图片。要防止库的用户访问仅供内部使用的资源，您应该通过声明一个或多个公开资源的方式来使用这种自动私有标识机制。或者，您也可以通过添加空的 <public /> 标记将所有资源设为私有，此标记不会将任何资源设为公开，而是会将一切（所有资源）都设为私有。

要声明公开资源，请向库的 public.xml 文件添加 <public> 声明。如果之前未添加过公开资源，您需要在库的 res/values/ 目录中创建 public.xml 文件。

以下示例代码会创建两个名称分别为 mylib_app_name 和 mylib_public_string 的公开字符串资源：

```
  <resources>
        <public name="mylib_app_name" type="string"/>
        <public name="mylib_public_string" type="string"/>
    </resources>
```
如果想要让任何资源对使用您的库的开发者可见，您应该将这些资源设为公开。

通过将属性隐式设为私有，您不仅可以防止库的用户从内部库资源获得代码完成建议，还可以重命名或移除私有资源，而不会破坏库的客户端。系统会将代码完成从私有资源中过滤出去，并且 Lint 会在您尝试引用私有资源时发出警告。  

**不会得到代码提示**
![](https://user-gold-cdn.xitu.io/2019/9/16/16d3847be481b118?w=707&h=204&f=png&s=26702)

**强行使用会导致lint警告**
![](https://user-gold-cdn.xitu.io/2019/9/16/16d38472648f18bf?w=758&h=249&f=png&s=27244)


## 为私有化的资源添加前缀或者后缀


![](https://user-gold-cdn.xitu.io/2019/9/16/16d384a6dfcffec5?w=727&h=368&f=png&s=62937)



