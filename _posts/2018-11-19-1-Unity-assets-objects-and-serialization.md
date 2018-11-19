---
layout: post_layout
title: 3-Unity如何序列化对象[翻译]
time: 2018年11月19日 星期一
location: 深圳
pulished: true
excerpt_separator: "```"
---
#### 本文从[此](https://unity3d.com/cn/learn/tutorials/topics/best-practices/assets-objects-and-serialization )翻译    

## 1.1了解Asset和UnityEngine.Object的区别

Asset    
- Asset是存储在Unity工程的资源文件    
- 有些Asset包含Unity原生格式的数据(如Material)    
- 另外一些non-native Asset需要处理为Unity原生格式(如FBX)    

UnityEngine.Object   
- 一个集合的序列化数据共同描述资源的特定实例 (如mesh,Sprite,AudioClip,AnimationClip->都是Object的子类)    
- 大部分Object都是Unity自带的，其中有两种特殊类型：    

	- ScriptableObject
		- 让开发者定义自己的数据类型，可由Unity序列化和反序列化，可在Editor窗口进行操作。
	- MonoBehaviour    
		- 提供指向MonoScript的包装器（wrapper）。MonoScript是内部数据类型，用来保持特定程序集和命名空间中的特定脚本类的引用，MonoScript和MonoBehaviour本身不包含任何可执行代码。

Asset和UnityEngine.Object属于一对多的关系，一个Asset包含一个或多个Object

## 1.2对象间引用
- 所有的UnityEngine.Object都可以引用其他UnityEngine.Object
- 这些其他UnityEngine.Object有可能与其在同一个Asset文件中，也有可能是被导入的其他Asset文件
- 如一个Material对象通常会包含其他Texture引用，这些引用通常来自其他Asset文件
- 当序列化时，这些引用由两块独立的数据片段组成：File GUID和Local ID
- File GUID标识目标Asset
- 本地唯一的Local ID则标识Asset中每个Object，因为一个Asset有可能包含多个Object

File GUID 存在于Asset的meta文件
```
fileFormatVersion: 2
guid: 32c03844d7c5a214a9bbe7f67025a7a0
timeCreated: 1481311032
```
	
Local GUID则存在与Asset文件中（Debug模式下可以看）
```
%YAML 1.1
%TAG !u! tag:unity3d.com,2011:
--- !u!21 &2100000
```
## 1.3为什么使用File GUID和Local ID？
- 答案是其稳健性以及提供了灵活的平台无关的工作流程

File GUID提供了文件具体位置的抽象，文件在磁盘的位置变得不相关。
因此可以自由的移动文件，而无须更新引用该文件的所有对象。
所以meta文件如果丢失了，那么该Asset中所有对象的引用也将丢失。
Local Id则可以唯一标识Asset中的某个Object。

Unity Edior具有已知文件GUID的特定文件路径的映射。
每当加载或导入文件时，映射都会记录。
映射记录将Asset的特定路径链接到Asset的文件GUID。
因此在Unity Editor没关闭的情况下，只要Asset的路径没改变,生成出来的meta文件GUID还是和原来一样。

## 1.4复合资源（Composite Assets）与导入器
非原生资源的导入需要通过资源导入器（asset importer），
这些导入器都是在资源被导入时自动执行的，我们可以通过AssetImporter及其子类来修改导入的规则。
在导入过程中，会将Assset转换为适合在Unity Editor中选择的平台所适配的格式。
在导入过程中，可以进行多个重量级操作，例如纹理压缩。

所以每次打开Unity Editor都执行导入过程会十分低效，
因此Asset导入的结果会被缓存到Library文件。
具体来说，导入过程的结果存储在以资产文件GUID的前两位数字命名的文件夹中。

这实际上适用于所有资源，而不仅仅是非原生资源。但是原生资源不需要冗长的转换过程或重新序列化。

## 1.5序列化与实例
虽然File GUID和Local ID这套是健壮的， 但是GUID的比较很慢，而运行时需要更高性能的系统。
Unity内部进行缓存，将File GUID和Local ID转化为单个会话唯一的简单整型，这被称为Instance ID。
当有新的Object注册缓存的时候，只要简单的自增即可。
缓存维护了Instances ID 、 Object的源文件位置(File GUID 和 Local ID) 和 在内存中的实例对象（如果有的话）

这允许Object互相之间健壮的引用，解析Instance ID引用能快速返回由Instance ID表示的已加载对象，如果目标对象没有加载，则GUID和Local ID可以解析为对象的位置，让Unity及时加载。

启动时，场景中引用的所有对象，以及包含在Resources目录所有对象，当在运行时导入新Asset，以及从AssetBundle加载对象时，也会将其添加到缓存

只有当条目变得陈旧时，才会被移出内存。
当提供访问特定文件GUID和Local ID的AssetBundle卸载的时候才会发生这种情况。
如果被卸载的AssetBundle重新加载，将会有个新的Instance ID。

注意某些平台的某些事件会导致对象内存不足。
例如在iOS平台当应用被挂起时，图像资源会从图形存储器卸载，而如果这些对象是来自已经被卸载的AssetBundle，Unity无法为这些对象重新加载资源，这些对象的任何现有引用都会无效。

实施说明：在运行时，上面的控制流并不是十分准确，在运行时比较文件GUID和本地ID在大量加载操作期间不会表现得很好。当构建Unity项目时，File GUID和Local ID就被确定的映射成更简单的格式。然而，大体的概念还是保持相同，并且在运行时用其思考仍然是有用的。
这也是为什么Asset File不能在运行时通过File GUID和Local ID来查找的原因。

## 1.6MonoScripts
MonoBehaviours拥有MonoScript的引用，MonoScript只是简单包含定位特定脚本类所需要信息。
这两种类型都不包含任何可执行代码。
MonoScript包含3个字符串：assembly名字，类名，命名空间
在构建项目时，Unity会收集所有在Asset目录下的松散脚本将其编译成Mono assemblies。
具体来说，Unity为Asset文件夹中使用的每种不同语言，以及Asset/Plugins文件夹中包含的脚本的单独assemblies。
Plugins子文件夹之外的C＃脚本放在Assembly-CSharp.dll中。
Plugins子文件夹中的脚本放在Assembly-CSharp-firstpass.dll中，依此类推。
这些程序集（加上预编译的程序集DLL）包含在Unity应用程序的最终构建中。
它们也是MonoScript引用的程序集。
与其他资源不同，Unity应用程序中包含的所有程序集都会在应用程序首次启动时加载。
这也是为什么AssetBundle(场景、预设)中不包含可执行脚本。
这允许不同MonoBehaviours引用特定共享类，即使MonoBehaviours在不同的AssetBundles。

## 1.7资源生命周期
有两种方式加载UnityEngine.Object:自动或者显式。

每当映射到该对象的实例ID被间接引用，而该对象当前未加载到内存中，并且可以定位对象的源数据时，将自动加载对象。对象也可以通过创建它们或通过调用资源加载API在脚本中显式加载。

加载对象时，Unity会尝试通过将每个引用的文件GUID和本地ID转换为实例ID来解析任何引用。
（An Object will be loaded on-demand the first time its Instance ID is dereferenced 
if two criteria are true:）
对象将被加载，第一次请求Instance ID，发现被间接引用，如果两个条件为真：
1. 实例ID引用当前未加载的对象 
1. 实例ID具有在缓存中注册的有效文件GUID和本地ID
这通常在引用本身加载和解析之后很快发生。

如果有文件GUID和Local ID但没有Instance ID或者Instance ID具有未加载对象及无效的File GUID和Local ID
那么将保留引用，但不会加载实际的对象。
这将在Unity Editor中显示为"（Missing）"，在运行中的应用，或者在场景中对象将以不正确的方式显示。

对象会在3种情景中卸载：
1. Object将会自动卸载，当发生未使用的Asset清理。这个过程将会在场景被破坏性改变时自动触发(例如调用Application.LoadLevel API)，或者脚本调用 Resources.UnloadUnusedAssets 接口。这个过程只会卸载那些未引用的对象：如果没有Mono变量保存对象的引用，并且没有其他活对象保存对对象的引用，那么对象将被卸载。	
1. Object来自Resources目录可以被显式卸载，调用Resources.UnloadAsset 接口。这些对象的Instance ID仍然有效，并且仍包含有效的File GUID和Local ID。如果任何Mono变量或其他Object持有对使用Resources.UnloadAsset卸载的对象的引用，那么一旦任何活引用被间接引用，该对象将被重新加载。
1. Object来自AssetBundle的话，调用AssetBundle.Unload(true)接口将会自动且马上的卸载。Instance ID和File GUID和Local ID 以及任何活引用都将变成("Missing")。从C#脚本尝试访问卸载对象上的方法或属性将产生NullReferenceException。

如果调用AssetBundle.Unload(false),活对象来自卸载的AssetBundle将不会被摧毁，但是Unity会无效其File GUID和Local ID以及Instance ID。Unity将不可能重新加载这些对象，如果后来对应资源从内存中卸载，并且对未加载的对象的活引用仍然存在。

## 1.8加载大层次结构
当序列化Unity GameObject时（如序列化Prefab时），要记住整个层次机构将被完全序列化。
也就是说，层次结构中的每个GameObject和GetComponent都将在序列化数据中单独表示。这对加载和实例化GameObject的层次结构有着有趣的影响。

创建GameObject层次结构时,CPU时间花费在不同的几个方面：
1. 读取源文件(从存储空间，从其他GameObject等)
1. 在新Transform间设置父子关系
1. 实例化GameObject及组件
1. 唤醒GameObject及组件
后三个时间成本通常是不变的，无论层次结构是从现有层次结构中克隆还是从存储加载。
然而，读取源数据的时间随着序列化到层次结构中的组件和GameObject的数量线性增加，并且乘以数据源的速度。

加载操作的成本由存储I/O时间支配。

当序列化单个prefab,每个GameObject和组件数据都是分开序列。所以UI有30个相同元素将会序列化30次，这将产生巨大的二进制数据，直到Unity支持嵌套prefab的时候（通过将重用元素移动到单独的prefab并且在运行时实例化，来显著减少其大型prefab加载时间），而不仅仅依赖于Unity的序列化和prefab系统。

此外，一旦构建了预制或GameObject层级，克隆现有层级比从存储加载新副本更快。

## 5.4note:
Unity 5.4改变了内存中Transform的表示。 每个根变换的整个子层次结构存储在紧凑，连续的存储器区域中。 当实例化将被立即重新排列到另一个层次结构中的新GameObject时，考虑使用接受父参数的新的GameObject.Instantiate重载。使用此重载可避免为新GameObject分配根变换层次结构。在测试中，这将实例化操作所需的时间加速约5-10％。
