
# AssetBundle

## AB包的组成
一个AssetBundle分为两个部分，一个头部部分，一个数据部分。数据部分保存资源的二进制数据，头部部分用来记录AB包的标识符，压缩格式，和manifest等。manifest 包含一个查找表(在大多数的设备上是平衡二叉树，而在widnow和osx平台上是红黑树)。通过以Object名作为索引，可以找到Object在数据部分中的位置。AB包对于Unity来说也是一个资源，会为该AB包和AB包的manifest文件创建对应的.meta文件

## AB包的压缩格式
AB包有三种压缩方式，如下所示
- None，不压缩，AB包中数据部分直接保存资源的二进制数据。
- LZMA，该方式对AB包数据部分进行整体压缩。
- LZ4，该方式对AB包中各个资源分别进行压缩。

AB包三种压缩方式的特点
- None，包体最大，加载最快
- LZMA，包体最小，加载最慢，在使用前需要将整体进行解压。
- LZ4，包体大小和加载速度在上面两者之间，在使用前不需要对整体进行解压，可以分块解压使用。

----------------

## AssetBundle的打包

使用BuildPipeline来进行AssetBundle打包，Unity会根据指定的平台，和指定压缩方式来生成特定的AB包。一个AB包会作为其中所有Asset的数据源(LocalID,和GUID)，unity会根据当前项目中资源的ab标签来进行AssetBundle的打包。

### 相关API
==BuildPipeline.BuildAssetBundles==

```C#
public class AssetBundleHelper {
	[MenuItem("Assets/BuildAll AssetBundle")]
    private static void BuildAllAssetBundle()
    {
        string pathName = "Assets/AssetBundle";
        if(!Directory.Exists(pathName))
        {
            Directory.CreateDirectory(pathName);
        }
        BuildPipeline.BuildAssetBundles(pathName,
                                        BuildAssetBundleOptions.None,
                                        BuildTarget.StandaloneWindows);
    }
}
```

### 资源的重复打包
假设我们有一个Prefab，该Prefab引用了一个Material。但是这个Material并没有打到任何一个AB包中，那么此时Unity 的打包策略会将该Material拷贝，然后将副本和Prefab一起进行打包。当多个不同的AB包中都有Object引用到该Material的时候，每一个AB包内都会有该Material的一份副本。这就会增大AB包的体积。会造成如下影响。
- 影响包体大小
- 可能增大了APP的体积
- 可能增大了运行时的内存消耗
- 可能会影响Unity的合批(patching)，不同AB包中Material副本会被认为是不同的Material，即使他们都是相同的material的副本


## 加载AssetBundle

==AssetBundle.LoadFromFile== 

- 当加载的AB包是未经压缩或者是LZ4压缩的话只加载AB包的头部(当需要加载AB包中的数据的时候再从磁盘中加载)，如果是加载LZMA压缩的AB包的话，会将AB包解压然后加载到内存中。 


==AssetBundle.LoadFromFileAsync==
-  上面的异步版本

==AssetBundle.LoadFromMemory==

- 从一个二进制数据块中创建并返回一个AB包，一般从服务器下载加密后的AB包，对其解密后再转换为AB包。但是使用该方法内存消耗很大。内存消耗可能有三份或者有两份分别为：托管代码中的加密的AB包，栈中的解密后的AB副本，创建出来的AB三个。

==AssetBundle.LoadFromMemoryAsync==
- 上面的异步版本

==[UnityWebRequest.GetAssetBundle，AssetBundleDownloadHandler](https://docs.unity3d.com/2017.4/Documentation/ScriptReference/Networking.UnityWebRequest.GetAssetBundle.html)==
- 从网络中下载AB包，内部使用一个环形缓冲区来进行数据的下载缓存。
- [LZMA格式的AB包在下载时会进行解压，并使用LZ4压缩来进行缓存(可以通过**Caching.CompressionEnable**来设置开启和关闭这一行为，默认开启)](https://learn.unity.com/tutorial/assets-resources-and-assetbundles#5c7f8528edbc2a002053b5a8)
- 如果一个AB包已经被下载缓存了，该方法不会再次下载该AB包可以立刻获取AB包。
- **加载AB包后记得要进行Dispose**


==AssetBundle.LoadFromStream==
- 适用于从流式数据中加载AssetBundle
- 使用LZMA压缩的AssetBundle会被解压到内存中
- 使用LZ4压缩或未压缩的AssetBundle会直接从流数据中读取
- 在AssetBundle.Unload之后再Dispose流式对象

==AssetBundle.LoadFromStreamAsync==
- 上面的异步版本

-------------------

## 从AssetBundle 中加载资源

### 相关API
**一下API都有异步版本，,以下所有的加载API同步版本都比异步的版本都要快至少一帧**

==AssetBundle.LoadAllAsset==
- 当需要从AB包中加载所有资源或者是大部分资源的时候，使用该API比分批次的使用**AssetBundle.LoadAssets**更高效。

==AssetBundle.LoadAssetWithSubAssets==
- 当加载一些复合资源的时候使用该API，例如Sprite Atlas和一些带有动画的Fbx模型等等。

==AssetBundle.LoadAsset==
- 当不符合上述两个API的使用条件的时候使用该API进行资源加载。

### AssetBundle资源加载的重复问题
假设从AB包中加载了一个资源M，那么在使用AssetBundle.unLoad(false)卸载AB之后，再从中加载资源M，那么内存中就会有两份资源M。Unity并不会识别出之前的资源M 。而是重新加载一份新的。这就造成了内存中资源重复。


## AssetBundle包依赖
当一个AB包中的一个或多个UnityEngine.Object，引用到另外一个AB包中的一个或者多个UnityEngine.Object时被称为AB包依赖。
在加载对象本身之前，重要的是加载包含对象依赖关系的所有AssetBundles。Unity不会尝试在加载AssetBundle时自动加载任何依赖的AssetBundle。

### 获取AB包的依赖
当我们第一次创建一个AB包后，Unity会在AB包所在的文件夹中创建一个和文件夹同名的AB包。这个AB包是用来管理该文件夹下所有AB包的信息的，所有AB包的依赖信息都保存在该AB包对应的manifest中。
当我们想要获取某一个AB包的依赖信息的时候，可以加载和文件夹同名的AB包的manifest文件来获取。Unity使用**AssetBundleManifest**类来表示一个manifest文件。使用下面的API可以获取AB包信息。

==[AssetBundleManifest.GetAllAssetBundles](http://docs.unity3d.com/ScriptReference/AssetBundleManifest.GetAllAssetBundles.html?_ga=2.120397960.1268009268.1597715295-264823416.1593599415)==
- 获取文件夹中所有的AB包名称

==[AssetBundleManifest.GetAllDependencies](http://docs.unity3d.com/ScriptReference/AssetBundleManifest.GetAllDependencies.html?_ga=2.120397960.1268009268.1597715295-264823416.1593599415)==
- 获取指定AB包的所有依赖

==[AssetBundleManifest.GetDirectDependencies](http://docs.unity3d.com/ScriptReference/AssetBundleManifest.GetDirectDependencies.html?_ga=2.120397960.1268009268.1597715295-264823416.1593599415)==

- 获取指定AB包的直接依赖

**这三个API都返回一个字符串数组，注意不要在性能敏感的时候使用**


## AssetBundle的卸载

### 相关API
==[public void Unload(bool unloadAllLoadedObjects)](https://docs.unity3d.com/ScriptReference/AssetBundle.Unload.html)==

- 当unloadAllLoadedObjects为true时，会将AB包和它所加载的资源全部从内存中卸载。不管其是否还在被使用 
- 当unloadAllLoadedObjects为false时，只将AB包从内存中卸载。已加载的资源保留在内存中

### 卸载AssetBundle 加载的资源
==[Resources.UnloadUnusedAssets](https://docs.unity3d.com/ScriptReference/Resources.UnloadUnusedAssets.html)==
一个AB包卸载后，当其遗留的资源无引用的时候可以调用该方法来卸载。

==[Resources.UnloadAsset](https://docs.unity3d.com/ScriptReference/Resources.UnloadAsset.html)==
一个AB包卸载后，我们可以使用该API来卸载指定的残留资源(已验证)



## 补充
对于异步的加载操作，Unity 分为两步一步是在后台线程进行对象的加载和反序列化，另外一步是在主线程进行对象集成，我们可以使用[Application.backgroundLoadingPriority](https://docs.unity3d.com/2017.4/Documentation/ScriptReference/Application-backgroundLoadingPriority.html)来修改加载操作在主线程进行对象集成的时间限制，来避免卡顿。

[Unity 官方教程：Assets，Resources，AssetBundle](https://learn.unity.com/tutorial/assets-resources-and-assetbundles#5c7f8528edbc2a002053b5a8)
[Unity 的memory-management](https://learn.unity.com/tutorial/memory-management-in-unity?language=en#5c7f8528edbc2a002053b599)
[Unity 的AssetBundle教程](https://learn.unity.com/tutorial/introduction-to-asset-bundles?language=en#5ce589b4edbc2a106aa7b47a)










