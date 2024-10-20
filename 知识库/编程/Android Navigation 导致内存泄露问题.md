#Android #Bug #内存泄漏 #Navigation


Android Navigation 是 Google Jetpack 里面的一个组件，支持 Android 应用里面的页面导航。

Navigation 优点体会
在使用过程中，我们感受到如下的优点。

页面跳转性能更好，在单 Activity 的架构下，都是 fragment 的切换，每次 fragment 被压栈之后，View 被销毁，相比之前 Activity 跳转，更加轻量，需要的内存更少。

通过 Viewmodel 进行数据共享更便捷，不需要页面之间来回传数据。

统一的 Navigation API 来更精细的控制跳转逻辑。

所有坑的中心
Navigation 相关的坑，都有个中心。一般情况下，Fragment 就是一个 View，View 的生命周期就是 Fragment 的生命周期，但是在 Navigation 的架构下，Fragment 的生命周期和 View 的生命周期是不一样的。当 navigate 到新的 UI，被覆盖的 UI，View 被销毁，但是保留了 fragment 实例（未被 destroy），当这个 fragment 被 resume 的时候，View 会被重新创建。这是“罪恶”之源。

整理了 8 个坑，我们一个一个趟
先来个小坑感受一些 Navigation。

1. Databinding 需要 onDestroyView 设置为 Null。
现在大家都会使用 Jetpack 里面的 databinding 技术，这个确实可以帮助我们简化很多代码，其中的自感知的生命周期，可以帮我们只有在必要的时候来更新 UI。

一般会在 Fragment 的 onCreateView 模板函数中初始化 ViewDataBing，这样就会有 Fragment 持有对 View 的引用。但是 fragment 和 view 的生命周期是不一样的，当 view 被销毁的时候，fragment 并不一定被销毁，所以一定要在 fragment.onDestroyView 函数中把对 view 的引用变量设置为 null，不然会导致 view 回收不掉。上一段官方的代码来说明一下。

```
private var _binding: ResultProfileBinding? = null
// This property is only valid between onCreateView and
// onDestroyView.
private val binding get() = _binding!!
 
override fun onCreateView(
    inflater: LayoutInflater,
    container: ViewGroup?,
    savedInstanceState: Bundle?
): View? {
    _binding = ResultProfileBinding.inflate(inflater, container, false)
    val view = binding.root
    return view
}
 
override fun onDestroyView() {
    super.onDestroyView()
    _binding = null
}
```



2. 当 Databinding 遇到错的 lifecycle.
Databinding 确实很强大，能把数据和 UI 进行绑定，这里对 UI 就有个要求，UI 一定要知道自己的生命周期的，知道自己什么时候处于 Active 和 InActive 的状态。所以我们必须要给 databinding 设置一个正确的生命周期.

下面来看一段有问题的代码：
```
override fun onCreateView(
    inflater: LayoutInflater,
    container: ViewGroup?,
    savedInstanceState: Bundle?
): View {
    _binding = HomeFragmentBinding.inflate(inflater, container, false)
    binding.lifecycleOwner = this // 问题代码在这里！！！
    return binding.root
}
```


这段代码运行起来没有问题，看起来都是按照预期的在执行。甚至官方代码也是这么写的。连 LeakCanary 也检测不出来内存泄漏的问题，LeakCanary 只能检测出来一些 Activity，Fragment 和 View 等实例的内存泄漏，对于普通的类的实例是没有办法分析的。

问题就出现在 databinding 遇到了一个错的 lifecycle，在没有用 Navigation 框架的时候，View 的生命周期和 Fragment 的生命周期一致的，但是在 Navigation 框架下，两者的生命周期是不一致的。我们来看下 ViewDataBinding 设置 lifecycleOwner 的具体代码。

下面的代码中，往这个 lifecycleOwner 里面加入了一个 OnStartListener 实例，因为这个 lifecycleOwner 是 fragment 的，会在 fragment 销毁的时候反注册，但是并不会在 View 被销毁的时候被反注册。而 OnStartListener 有对这个 ViewDataBinding 有引用，会导致 View 被销毁的时候（跳到另外一个页面），这个引用会阻止系统回收这个 View。

这个分析逻辑是对的，但是结果是不对的，系统还是会对这个 View 进行回收，因为 OnStartListener 的实例持有的是对这个 View 的弱引用，这个 View 还是会被回收。这就是 LeakCanary 没有报错的原因。但是这个 OnStartListener 的实例，就没这么幸运了，正是这个实例无法回收导致了内存泄漏。

```
@MainThread
public void setLifecycleOwner(@Nullable LifecycleOwner lifecycleOwner) {
    if (mLifecycleOwner == lifecycleOwner) {
        return;
    }
    if (mLifecycleOwner != null) {
        mLifecycleOwner.getLifecycle().removeObserver(mOnStartListener);
    }
    mLifecycleOwner = lifecycleOwner;
    if (lifecycleOwner != null) {
        if (mOnStartListener == null) {
            mOnStartListener = new OnStartListener(this);
            // 这个实例持有了ViewDataBinging的实例，虽然是弱引用。
        }
        lifecycleOwner.getLifecycle().addObserver(mOnStartListener);
        // 问题出现在这里，如果这个lifecycle是fragment的，View被销毁了，里面不会进行反注册。
    }
    for (WeakListener<?> weakListener : mLocalFieldObservers) {
        if (weakListener != null) {
            weakListener.setLifecycleOwner(lifecycleOwner);
        }
    }
}
```


正确的做法是需要给这个 ViewDataBinding 设置 viewLifecycleOwner.

binding.lifecycleOwner = viewLifecycleOwner
多说一句啊，这个问题是如何被发现的呢？我们有一套检查框架逻辑的代码，对于这个问题，我们会在 fragment 的 onStop 函数里面检查有多少实例在监听 fragment 的生命周期，我们发现，这个数字会一直涨，这个问题就暴露了，适当的时候，和大家分享一下这套框架。

3. Glide 自我管理的生命周期值得信赖吗？
不值得信赖了

Glide 是一个非常流行的图片加载框架，不得不说，Glide 的缓存这一块的设计非常的优秀，功能强大，可扩展强。还有它的生命周期的自我管理，通过创建一个 fragment 在当前的页面，通过这个 fragment 的生命周期，实现在 onStart 的时候进行图片加载，在 onStop 的时候，把还没有执行或者没有执行完的任务缓存下来，以便在 onStart 的再执行，当然是在没有 onDestory 的情况下。

一切都很完美，直到遇到了 Navigation。

glide.with(fragment).load(url).into(imageview).
呵呵，上面的这段，在 Navigation 的架构下，如果 Fragment 还在，但是执行了 onDestroyView，imageview 需要被销毁。这个情况下，如果图片加载任务没执行完，任务就会被缓存下来了。这个任务还有对需要被销毁的 imageview 有强引用，导致这个 imageview 销毁不了，从而内存泄漏。

如何 100%的重现这个问题呢，有个简单的方法，让大家可以验证一下这个问题。给这个任务，加一个图片的 transformation，这个 transformation 什么也不干，就是 sleep 3 秒钟，在这个 3 秒中之内，跳转到另一个页面。这会导致当前页面进行 View 的 destory，但是 fragment 并不会 destory，因为这个任务还没执行完，这个任务就会被 Glide 缓存，具体会被缓存位置为 RequestManager->RequestTracker->pendingRequests。

如何来解决这个问题呢？这个没有现成的解决方法，在 Glide 的官网有提类似的问题，但是 Glide 维护者听起来还没有意识到这个问题，没有后续的计划。当然，我们需要来解决这个问题，不然我们的代码就会存在这一点瑕疵了。

解决的方法：自己来管理 Glide 的生命周期，不要通过那个看不见的 fragment 的生命周期，因为那是靠不住的。我们自己写了一个 RequestManager，通过传入的 fragment 的 viewLifecycleOwner 来进行管理。使用也很方便，在调用的时候如下即可。

KGlide.with(fragment).load(url).into(imageview).
源码精简了一下，贴在这里，请指正。

```
import com.bumptech.glide.manager.Lifecycle as GlideLifecycle
 
class KGlide {
 
    companion object {
        private val lifecycleMap = ArrayMap<LifecycleOwner, RequestManager>()
 
        @MainThread
        fun with(fragment: Fragment): RequestManager {
            Util.assertMainThread()
 
            val lifecycleOwner = fragment.viewLifecycleOwner
            if (lifecycleOwner.lifecycle.currentState == Lifecycle.State.DESTROYED) {
                throw IllegalStateException("View is already destroyed.")
            }
 
            if (lifecycleMap[lifecycleOwner] == null) {
                val appContext = fragment.requireContext().applicationContext
                lifecycleMap[lifecycleOwner] = RequestManager(
                    Glide.get(appContext),
                    KLifecycle(lifecycleOwner.lifecycle),
                    KEmptyRequestManagerTreeNode(), appContext
                )
            }
            return lifecycleMap[lifecycleOwner]!!
        }
    }
 
    class KEmptyRequestManagerTreeNode : RequestManagerTreeNode {
        override fun getDescendants(): Set<RequestManager> {
            return emptySet()
        }
    }
 
    class KLifecycle(private val lifecycle: Lifecycle) : GlideLifecycle {
        private val lifecycleListeners =
            Collections.newSetFromMap(WeakHashMap<LifecycleListener, Boolean>())
 
        private val lifecycleObserver = object : DefaultLifecycleObserver {
            override fun onStart(owner: LifecycleOwner) {
                val listeners = Util.getSnapshot(lifecycleListeners)
                for (listener in listeners) {
                    listener.onStart()
                }
            }
 
            override fun onStop(owner: LifecycleOwner) {
                val listeners = Util.getSnapshot(lifecycleListeners)
                for (listener in listeners) {
                    listener.onStop()
                }
            }
 
            override fun onDestroy(owner: LifecycleOwner) {
                val listeners = Util.getSnapshot(lifecycleListeners)
                for (listener in listeners) {
                    listener.onDestroy()
                }
 
                lifecycleMap.remove(owner)
                lifecycleListeners.clear()
                lifecycle.removeObserver(this)
            }
        }
 
        init {
            lifecycle.addObserver(lifecycleObserver)
        }
 
        override fun addListener(listener: LifecycleListener) {
            lifecycleListeners.add(listener)
            when (lifecycle.currentState) {
                Lifecycle.State.STARTED, Lifecycle.State.RESUMED -> listener.onStart()
                Lifecycle.State.DESTROYED -> listener.onDestroy()
                else -> listener.onStop()
            }
        }
 
        override fun removeListener(listener: LifecycleListener) {
            lifecycleListeners.remove(listener)
        }
    }
}

```


4. Android 组件的生命周期自我管理值得信任吗？
不值得，信任需要我们对 Android 生命周期的管理细节足够的了解。没有足够的了解，哪里来的信任，也就是盲目的信任。

我们在 Android 官方文档里面应该看到过 LiveData 的介绍，下面摘录一段。

Livedata is an observable data holder class. Unlike a regular observable, LiveData is lifecycle-aware, meaning it respects the lifecycle of other app components, such as activities, fragments, or services. This awareness ensures LiveData only updates app component observers that are in an active lifecycle state.

然后还向我们说明 Livedata 的不会导致内存泄漏。

This is especially useful for activities and fragments because they can safely observe LiveData objects and not worry about leaks—activities and fragments are instantly unsubscribed when their lifecycles are destroyed.

写的很清楚，言之昭昭啊。如果你相信了官方文档的介绍，就 too young，too simple 了。LiveData 未必会在 lifecycleOwner 销毁的时候进行反注册，内存泄漏还是会发生。我们看一段 LiveData 会产生内存泄漏的代码。

```
class HomeFragment : Fragment() {
    private val model: NavigationViewModel by viewModels()
 
    override fun onCreateView(
            inflater: LayoutInflater,
            container: ViewGroup?,
            savedInstanceState: Bundle?
    ): View? {
        return inflater.inflate(R.layout.home_fragment, container, false)
    }
 
    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)
        model.getTextValue().observe(viewLifecycleOwner){
            view.findViewById<Button>(R.id.text).text = it
        }
 
        if (isXXX()) {
            findNavController().navigate(R.id.next_action)
        }
    }
}

```


当你进入某个页面，发现需要导航到另一个页面，这个时候就需要很小心。如果像上面这样的写法，就会导致内存泄漏。

这个 Case 里，在 Fragment.onViewCreated()的模板方法，监听了一个 LiveData，这会导致这个 LiveData 持有外面对象的引用。理想情况下，这个 LivaData 会在 LifecycleOwner 在 onDestory 的时候进行反注册，但是在一些情况下，这个反注册就不会进行。

如上代码的情况下，如果这个页面马上跳到 next_action 的页面，之前订阅的 LiveData 就不会进行反注册。原因出在当跳出这个页面的时候，页面还处于生命周期的状态 INITIALIZED，但是反注册的条件是这个页面的生命周期状态至少是 CREATED.


```
void performDestroyView() {
    mChildFragmentManager.dispatchDestroyView();
    if (mView != null && mViewLifecycleOwner.getLifecycle().getCurrentState()
                    .isAtLeast(Lifecycle.State.CREATED)) {
        mViewLifecycleOwner.handleLifecycleEvent(Lifecycle.Event.ON_DESTROY);
    }
    ......
    }
 
```
其实 Android 的生命周期管理还是值得信任的，前提是我们得彻底搞清楚状态流转的细节。

5. 当 ViewPager2 遇到 Navigation
ViewPager 是在应用开发的过程中，高频的用到的组件。Android 的官网有对基本的使用有详细的介绍。

一直都很美好，直到遇到 Navigation。

让我们来看官方例子里面 ViewPager2 的 Adapter 的类的声明。

```
class DemoCollectionAdapter(fragment: Fragment) : FragmentStateAdapter(fragment) {
 
    override fun getItemCount(): Int = 100
 
    override fun createFragment(position: Int): Fragment {
        // Return a NEW fragment instance in createFragment(int)
        val fragment = DemoObjectFragment()
        fragment.arguments = Bundle().apply {
            // Our object is just an integer :-P
            putInt(ARG_OBJECT, position + 1)
        }
        return fragment
    }
}
```


不避讳的说，我们实际项目中的代码，也犯了同样的问题。不是说官网的写法有问题，而是在 Navigation 的框架下，才会导致的内存泄漏问题。这个泄漏是如何发生的呢？我们来看一下 FragmentStateAdapter 的构造函数。

```
/**
 * @param fragment if the {@link ViewPager2} lives directly in a {@link Fragment} subclass.
 *
 * @see FragmentStateAdapter#FragmentStateAdapter(FragmentActivity)
 * @see FragmentStateAdapter#FragmentStateAdapter(FragmentManager, Lifecycle)
 */
public FragmentStateAdapter(@NonNull Fragment fragment) {
    this(fragment.getChildFragmentManager(), fragment.getLifecycle());
}
/**
 * @param fragmentManager of {@link ViewPager2}'s host
 * @param lifecycle of {@link ViewPager2}'s host
 *
 * @see FragmentStateAdapter#FragmentStateAdapter(FragmentActivity)
 * @see FragmentStateAdapter#FragmentStateAdapter(Fragment)
 */
public FragmentStateAdapter(@NonNull FragmentManager fragmentManager,
        @NonNull Lifecycle lifecycle) {
    mFragmentManager = fragmentManager;
    mLifecycle = lifecycle;
    super.setHasStableIds(true);
}

```


可以看到 FragmentStateAdapter 最终会走两个参数的构造函数。第一个是 fragmentManager of ViewPager2's Host，第二个参数是 lifecycle of ViewPager2's host。如果你看懂了之前的问题，你就会知道这个问题出在哪里了。在 Navigation 下面，fragment 和 view 的生命周期是不一致的，如果我们在 FragmentStateAdapter 的构造函数中，只传入 fragment 的实例的话，第二个参数 lifecycle 用的是第一个参数 fragment 的 lifecycle。但是很显然，viewpager2's host 的 lifecycleOwner 是 fragment 的 viewlifecycleOwner，而不是其本身。

具体导致的问题是，在 ViewPager2 实例被销毁的时候，对应的 FragmentStateAdapter 并不会被销毁，因为如果只传一个参数的话，使用的是 Fragment 的生命周期，只有在 fragment 退出的时候，才会被销毁。

这里多说一句啊，FragmentStateAdapter 实例不能被设置到多个 ViewPager2 的对象，所以当 ViewPager2 被重建的时候，这个 Adapter 不能被重用。

这些问题其实很难被发现，LeakCanary 也不能发现。幸好我们有个工具，在每个需要被检查的类的构造函数里面进行记录，然后在类的 finalize 方法对这个记录进行处理，如果发现某个类一直被构造，但是不执行 finalize 方法，这个类就需要被好好关照了。

6. ViewPager2 设置 Adapter 导致的 Fragment 重建问题
先来看以下的代码片段：

```
Line1：val viewPager2: ViewPager2 = ......
Line2：val adapter: FragmentStateAdapter = ......
Line3：viewPager2.adapter = adapter
Line4：model.getContentList.observe(viewLifecycleOwner) {
Line5：    adapter.data = it
Line6：    adapter.notifyDataSetChanged()
Line7：}
```

大家应该看不出来这段代码的问题所在的吧，这个是非常常规的写法。当然这段代码在非 Navigation 的架构下面是没有问题的。但是如果在 Navigation 的架构下，就会有比较严重的问题了。

说明一下问题出现的场景，如果用户先进入这个页面，执行上面代码，viewpager 正常显示。然后注意，重要的步骤来了，在这个页面上，导航到另外一个页面。那当前的这个页面会执行 fragment 的 onStop，注意并不会执行 onDestory。但是会执行 onDestoryView，也就是说 viewPager 将会被销毁，但是 fragment 被保留了。

那如果重新回到这个页面会发生什么事情呢，之前 onStop 的 fragment 会执行 onStart，包括 Adatper 里面生成的 fragment 也会进行重建，并创建 View。

出人意料的事情发生了，Adatper 里面的 fragment 在重建完成之后，立刻又被销毁掉了，这里的销毁是真正的销毁，执行了 onDestory 方法。然一个新的 fragment 被重新创建出来，这就是 fragment 重建问题。是什么导致了这个问题呢？

具体执行销毁 Fragment 的代码如下，在 FragmentStateAdapter 的 gcFragments 的方法。

```
void gcFragments() {
    if (!mHasStaleFragments || shouldDelayFragmentTransactions()) {
        return;
    }
 
    // Remove Fragments for items that are no longer part of the data-set
    Set<Long> toRemove = new ArraySet<>();
    for (int ix = 0; ix < mFragments.size(); ix++) {
        long itemId = mFragments.keyAt(ix);
        if (!containsItem(itemId)) {
            toRemove.add(itemId);
            mItemIdToViewHolder.remove(itemId); // in case they're still bound
        }
    }
 
    // Remove Fragments that are not bound anywhere -- pending a grace period
    if (!mIsInGracePeriod) {
        mHasStaleFragments = false; // we've executed all GC checks
 
        for (int ix = 0; ix < mFragments.size(); ix++) {
            long itemId = mFragments.keyAt(ix);
            if (!isFragmentViewBound(itemId)) {
                toRemove.add(itemId);
            }
        }
    }
 
    for (Long itemId : toRemove) {
        removeFragment(itemId);
    }
}
```



因为这个函数判断，之前 adapter 里面产生的 fragment 需要被回收，依据就是当前的 adatper.containsItem(id)的方法返回 false 了。再提供一个信息，这个函数会在 viewpager2 设置 adatper 的时候被调用。到现在为止，答案已经出来了。因为在 viewpager2 设置 adatper 的时候，adatper 里面什么数据也没有的啊，containsItem 函数必然返回为空的啊，真相大白了。

所以逻辑正确的代码应该如下：

```
val viewPager2: ViewPager2 = ......
model.getContentList.observe(viewLifecycleOwner) {
    if（viewPager2.adapter == null）{
        val adapter: FragmentStateAdapter = ......
        adapter.data = it
        viewPager2.adapter = adapter
    } else {
        viewPager2.adapter.data = it
    }
 
    adapter.notifyDataSetChanged()
}
```


这段代码解决问题就是，在 viewpager2 设置 adatper 之前，先把 adapter 填充进去数据，然后再进行设置。这样就可以解决在 gcFragments 里面因为 containsItem()函数返回 false，导致 fragment 被销毁的问题，其实这个 fragment 是可以被重用的。

7. 在 Navigation 的框架下，手动进行 Fragment 管理需要注意什么？
刚开始使用 Navigation，代码里还是会有一部分 Fragment 是手动管理的，通过 FragmentManager 的 Add/Replace/Remove 等操作。其实 Navigation 设计的一部分初衷，就是要用统一的导航操作来替代手动操作，虽然 Navigation 底层的操作也是通过 FragmentManager 来实现的。

如果代码里面还是有手动管理的代码，需要特别注意就是手动操作的时机和方式。原因还是在 Navigation 的框架下，Fragment 和 View 的生命周期不一致导致的。如果把操作 Fragment 的时机放在 ViewLifeCycle 的里面，就可能会造成一些意想不到的结果。

假设 stack top-most 的页面返回之后，新处于 Stack 顶的页面，原先处于 Stop 的 Fragment 就会走 onStart，而整个 View 将会被重建。如果在 ViewLifeCycle 的生命周期里面去 Add or Replace 一个 fragment，就必须要判断，需要操作的 fragment 是否已经存在，如果这个 fragment 已经存在，又进行了一次 Add or Replace 操作，这个 fragment 将会被重建，原来的 fragment 将会被销毁，新的 fragment 会被创建，而 view 的生命周期将会走两次，导致不必要的性能损失。不仅仅是性能的损失，还会导致之前谈到的第 4 条问题，导致内存泄漏的问题。

即使是注意到了这些问题，如果手动来判断情况，也会造成代码不必要的复杂，所以还是建议使用 Navigation 的框架来导航，而不是手动通过 FragmentManager 来进行操作。

8. Navigation 的主持下，Fragment 和 View 分家了，家产怎么分？
最后其实是个设计问题。View 是依附于 Fragment，从 Fragment 的 create 到 destory 的一生中，可能会伴随着多个 View 的实例，从 create 走到 destory。

从而，我们需要考虑，哪些变量应该是属于 fragment 的，哪些变量应该属于 view 的。

举一个例子，如果一个页面有一个列表，使用 recycleView 来实现，毫无疑问，recycleView 是属于 View 的，但是这个 recycleView 的 adapter 呢？如果这个 adapter 属于 view，那 adapter 的实例将会随着这个 recycleview 的创建而创建，消亡而消亡。如果把这个 adapter 放到 fragment，不管有多少个 View 实例将会被创建，用的就是 fragment 里面的 adapter，这样的设计，是不是会更好，更少的代价来实现需求。

所以这个家产如何分，第一优先顺位是 fragment, 第二个顺位才是 view。放在 fragment 里面才会最大程度的重用对象，达到性能的最大化。N
原文链接：https://blog.csdn.net/ByteDanceTech/article/details/120052166