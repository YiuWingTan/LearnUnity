# Unity 的资源管理

- [资源的导入过程](# unity 中的资源的导入过程)
- [资源的实例化](# 资源的实例化)
- [资源加载和卸载相关API](#  资源加载和卸载相关API)

## 资源的导入过程
- 当我们导入一个资源的时候，Unity 会调用AssetImport来导入资源, Unity 还会将其进行一个转换，转换为Unity 自身可以识别的格式，并将转换后的文件保存在Unity工程目录下的Library文件夹中，放在 Library/metadata/[XX]/ 下，[XX]是GUID的头两位。当我们修改一个资源的源文件（或者它所依赖的资源）之后，Unity 会重新导入这个文件，并且更新已导入的版本。

- 当我们导入了资源之后，Unity 会为这个资源创建一个.meta文件，该文件里面记录有资源的GUID ，还会为该资源中引用到的对象创建一个localID用来表示资源中的引用关系，Unity 内部会有一张表来记录资源的GUID和资源所在路径的对应关系。Unity内部的资源管理系统可以通过GUID+LocalID来引用到一个资源中的任意一个对象(Unity Object)。

ps：当我们使用版本控制的时候，最好ignore掉Library文件夹。

--------------
## 资源的实例化
因为在运行时比较GUID和LocalID十分缓慢，Unity 维护一个Instance ID的缓存，这个缓存维护着实例ID(Instance ID)、对象的源文件(用文件GUID、本地ID来标识)和内存中的对象(如果有的话)三者之间的关联关系。Unity 这个内部缓存的名字就叫做**PersistentManager**

这就让UnityEngine.Object能够健壮地维护彼此之间的关联，通过解析实例ID(Instance ID)就能快速返回载入在内存中的对象，如果对象还没有被加载，那么 文件GUID+本地ID 能被快速定位对象的源文件，Unity就能及时进行加载。

### Object加载
当Unity的应用程序启动的时候，PersistentManager的缓存系统会对项目立刻要用到的数据（比如启动场景里的物体和它的依赖项），以及所有包含在Resources 目录的Objects进行初始化生成Instance ID。如果在运行时导入了Asset和使用脚本创建资源，或者从AssetBundles加载Object都会在**PersistentManager** 产生新的项。

另外Object在满足下列条件的情况时会自动加载：
1、某个Object被引用了。
2、Object当前没有被加载进内存。
3、可以定位到Object的源位置（File GUID 和 Local ID）。

另外，如果File GUID和LocalID没有Instance ID，或者有Instance ID，但是对应的Objects已经被卸载了，且这个Instance ID引用了无效的File GUID和LocalID，那么这个Objects的引用会被保留，但是实际Objects不会被加载。在Unity的编辑器里会显示为：“(Missing)”引用，而在运行时，根据Objects类型不一样，有可能会是空指针，有可能会丢失网格或者纹理贴图导致场景或者物体显示粉红色。

### Object卸载
除了加载之外，Objects会在一些特定情况下被卸载。

1、一般在切场景或者手动调用了Resources.UnloadUnusedAssets的 API时候，一些没有被使用的资源就会被卸载。但是这个过程只会卸载那些没有任何引用的Objects。但是一些**HideFlags**被标记为** HideFlags.DontUnloadUnusedAsset**或者是** HideFlags.HideAndDontSave**的则不会被卸载。

2、从Resources目录下加载的Objects可以通过调用Resources.UnloadAsset API进行显式的卸载。但这些Objects的 Instance ID会保持有效，并且仍然会包含有效的File GUID 和 LocalID 。当任何Mono的变量或者其他Objects持有了被卸载的Objects的引用之后，这个Object在被直接或者间接引用之前被加载。

3、从AssetBundles里得到的Objects在执行了AssetBundle.Unload(true) API之后，会立刻被卸载。并且这会立刻让这些Objects的File GUID 、 Local ID以及Instance ID立刻失效。但如果调用的是AssetBundle.Unload(false)API的话，那么生命周期内的Objects不会随着AssetBundle一起被销毁，但是Unity会中断 File GUID 、Local ID和对应Object的Instance IDs之间的联系(因为AssetBundle已经被卸载掉了)，也就是说，如果这些Objects在未来的某些时候被销毁了，那么当再次对这些Objects进行引用的时候，是没法再自动进行重加载的。

另外，如果Objects中断了它和源AssetBundle的联系之后，那么再次加载相同Asset的时候，Unity也不会复用先前InstanceID和内存中的Asset，而是会重新创建 Instance ID和Asset，也就是说内存里会有多份冗余的资源。

在IOS中如果一个APP挂起，那么该APP加载到显存中的数据就会被卸载掉。

-------------------------------



-------------------
## 资源加载和卸载相关API

**Resources 一些常用的API，具体功能查文档**
```C#
Resources.Load //从Resources 文件夹下加载资源
Resources.LoadAll //加载路径下的所有Asset
Resources.LoadAsync //从Resources 文件夹下异步加载资源
Resources.UnloadAsset //卸载一个资源，该函数只能用在卸载Resources文件夹中的资源
Resources.UnloadUnusedAssets //异步的卸载资源,并不能释放AB包中东西，只能释放从AB包中加载出来的资源，也可以释放场景中的资源，其它不是从AB包加载来的资源。

```

## 使用Resources来管理资源
实践
-  只将项目中不经常改变，而且全局使用没有平台差异的东西保存到Resources中。
-  在进行原型开发的时候可以使用Resources。

缺点
- 放入过多的东西会增加应用程序的启动时间。
- 放入过多的东西会增加应用程序的体积。
- Resources会让内存管理变的困难，因为其中的资源大小不一，难以实现颗粒化的内存管理。
- 使用Resources 无法热更。

优点
- 有利于项目的开发阶段，减少了AB打包所带来的的额外等待时间。 

--------------------
## 使用AssetBundle来管理资源

缺点
- 使用异步加载的时候，需要写比较繁琐的回调处理
- 使用之前需要花费时间进行打包，尤其是在开发时候，调整资源频繁，如果忘记打包可能导致表现BUG。
- 调试的时候，无法通过Hierarchy直接定位到资源。

----------------------
## AssetBundle 打包策略
https://www.jianshu.com/p/5d4145cd900c




----------------------
GUI不能再运行时获取吗？(待验证)
Asset File GUIDs cannot be queried at runtime.

不仅在IOS在安卓上，一个app挂起后，显存中的数据会被卸载吗？(待验证)
The most common case where Objects are removed from memory at runtime without being unloaded occurs when Unity loses control of its graphics context. This may occur when a mobile app is suspended and the app is forced into the background. In this case, the mobile OS usually evicts all graphical resources from GPU memory. When the app returns to the foreground, Unity must reload all needed Textures, Shaders and Meshes to the GPU before scene rendering can resume

