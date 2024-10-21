# 【架构设计】MVP（Model-View-Presenter）

MVP（Model-View-Presenter）是 MVC 的演化版本，MVP 的角色定义如下。 

• Model：主要提供数据的存取功能。Presenter 需要通过 Model 层来存储、获取数据。 

• View：负责处理用户事件和视图部分的展示。在 Android 中，它可能是 Activity、Fragment 类或者是某个 View 控件。 

• Presenter：作为 View 和 Model 之间沟通的桥梁，它从 Model 层检索数据后返回给 View 层，使得 View 和 Model 之间没有耦合。 

在 MVP 里（见图 10-2），Presenter 完全将 Model 和 View 进行了分离，主要的程序逻辑在 Presenter 里实现。而且，Presenter 与具体的 View 是没有直接关联的，而是通过定义好的接口进 行交互，从而使得在变更 View 时可以保持 Presenter 的不变，这点符合面向接口编程的特点。 View 只应该有简单的 Set/Get 方法，以及用户输入和设置界面显示的内容，除此之外就不应该 有更多的内容。绝不允许 View 直接访问 Model，这就是其与 MVC 的很大不同之处。

![image-20241021140202258](https://gitee.com/ForeverHamburger/picgo_imgs1/raw/master/202410211402866.png)

## 应用 MVP 模式

此前写过一篇博客，关于MVP设计模式以及MVC设计模式，故而在这里不多做赘述。

### 实现 Model 

首先我们要创建 Model 实体 IpInfo：

![image-20241021154520587](https://gitee.com/ForeverHamburger/picgo_imgs1/raw/master/202410211545652.png)

IpData 的部分代码如下所示：

![image-20241021154542352](https://gitee.com/ForeverHamburger/picgo_imgs1/raw/master/202410211545394.png)

随后定义获取网络数据的接口类：

![image-20241021154559850](https://gitee.com/ForeverHamburger/picgo_imgs1/raw/master/202410211545875.png)

这里有一个回调监听接口 LoadTasksCallBack ，它定义了网络访问回调的各种状态：

![image-20241021154611974](https://gitee.com/ForeverHamburger/picgo_imgs1/raw/master/202410211546012.png)

接下来我们编写 NetTask 的实现类以获取数据，如下所示：

```java
public class IpInfoTask implements NetTask＜String＞ {
 	private static IpInfoTask INSTANCE = null；
 	private static final String HOST = ＂http：//ip.taobao.com/service/getIpInfo.php＂；
	private LoadTasksCallBack loadTasksCallBack；
 	private IpInfoTask() {
 	}
 	public static IpInfoTask getInstance() {
 		if (INSTANCE == null) {
 			INSTANCE = new IpInfoTask()；
 		}
 		return INSTANCE；
 	}
 	@Override
 	public void execute(String ip, final LoadTasksCallBack loadTasksCallBack) { 
 		RequestParams requestParams = new RequestParams()；
 		requestParams.addFormDataPart(＂ip＂, ip)；
 		HttpRequest.post(HOST, requestParams, new BaseHttpRequestCallback＜IpInfo＞() {
 			@Override
 			public void onStart() {
 				super.onStart()；
 				loadTasksCallBack.onStart()；
 			}
 			@Override
 			protected void onSuccess(IpInfo ipInfo) {
 				super.onSuccess(ipInfo)；
 				loadTasksCallBack.onSuccess(ipInfo)；
 			}
		 	@Override
 			public void onFinish() {
 				super.onFinish()；
 				loadTasksCallBack.onFinish()；
 			}
 			@Override
 			public void onFailure(int errorCode, String msg) {
 				super.onFailure(errorCode, msg)；
 				loadTasksCallBack.onFailed()；
 			}
     	})；
    }
}
```

IpInfoTask 是一个单例类，在 execute 方法中通过 OkHttpFinal 来获取数据，同时在 OkHttpFinal 的回调函数中调用自己定义的回调函数 loadTasksCallBack。

### 实现 Presenter

首先定义一个契约接口 IpInfoContract，契约接口主要用来存放相同业务的 Presenter 和 View 的接口，便于查找和维护。其代码如下所示：

![image-20241021155001194](https://gitee.com/ForeverHamburger/picgo_imgs1/raw/master/202410211550228.png)

在此可以看到 Presenter 接口定义了获取数据的方法，而 View 定义了与界面交互的方法。 其中，isActive方法用于判断Fragment是否添加到了Activity中。另外，View接口继承自BaseView 接口，BaseView 接口如下所示：

```java
public interface BaseView＜T＞ {
 	void setPresenter(T presenter)；
}
```

BaseView 接口的目的就是给 View 绑定 Presenter，接着实现 Presenter 接口，如下所示：

```java
public class IpInfoPresenter implements IpInfoContract.Presenter, LoadTasksCallBack＜IpInfo＞ {
 	private NetTask netTask；
 	private IpInfoContract.View addTaskView；
 	public IpInfoPresenter(IpInfoContract.View addTaskView, NetTask netTask) {
 		this.netTask = netTask；
 		this.addTaskView=addTaskView；
 	}
 	@Override
 	public void getIpInfo(String ip) {
 		netTask.execute(ip,this)；//1
 	}
 	@Override
 	public void onSuccess(IpInfo ipInfo) {
 		if(addTaskView.isActive()){
 			addTaskView.setIpInfo(ipInfo)；
 		}
 	}
 	@Override
 	public void onStart() {
 		if(addTaskView.isActive()){
 			addTaskView.showLoading()；
 		}
 	}
 	@Override
 	public void onFailed() {
 		if(addTaskView.isActive()){
 			addTaskView.showError()；
 			addTaskView.hideLoading()；
 		}
 	}
 	@Override
 	public void onFinish() {
 		if(addTaskView.isActive()){
 			addTaskView.hideLoading()；
 		}
 	}
}
```

IpInfoPresenter 中含有 NetTask 和 IpInfoContract.View 的实例（后面会讲），并且实现了LoadTasksCallBack 接口。在上面代码注释 1 处，将自身传入 NetTask 的 execute 方法中来获取 数据，并回调给IpInfoPresenter，最后通过 addTaskView 来和 View 进行交互，并更改界面。这 回我们应该明白了：Presenter就是一个中间人的角色，其通过 NetTask，也就是 Model 层来获 得和保存数据，然后再通过 View 更新界面，这期间通过定义接口使得 View 和 Model 没有任何 交互。最后看看 View 层的实现。

### 实现 View

在上面的契约接口IpInfoContract中我们已经定义了View接口，实现它的是IpInfoFragment。 其代码如下所示：

```java
public class IpInfoFragment extends Fragment implements IpInfoContract.View {
 	private TextView tv_country；
 	private TextView tv_area；
 	private TextView tv_city；
 	private Button bt_ipinfo；
 	private Dialog mDialog；
 	private IpInfoContract.Presenter mPresenter；
 	public static IpInfoFragment newInstance() {
 		return new IpInfoFragment()；
 	}
    
 	@Override
 	public View onCreateView(LayoutInflater inflater, ViewGroup container,Bundle savedInstanceState) {
 		View root = inflater.inflate(R.layout.fragment_ipinfo, container,false)；
 		tv_country= (TextView) root.findViewById(R.id.tv_country)；
 		tv_area= (TextView) root.findViewById(R.id.tv_area)；
 		tv_city= (TextView) root.findViewById(R.id.tv_city)；
 		bt_ipinfo= (Button) root.findViewById(R.id.bt_ipinfo)；
 		return root；
 	}
    
 	@Override
 	public void onActivityCreated(Bundle savedInstanceState) {
 		super.onActivityCreated(savedInstanceState)；
 		mDialog=new ProgressDialog(getActivity())；
 		mDialog.setTitle(＂获取数据中＂)；
     	bt_ipinfo.setOnClickListener(new View.OnClickListener() {
 			@Override
 			public void onClick(View view) {
 				mPresenter.getIpInfo(＂39.155.184.147＂)；//2
 			}
 		})；
 	}
	@Override
	public void setPresenter(IpInfoContract.Presenter presenter) {//1
 		mPresenter=presenter；
 	}
 	@Override
 	public void setIpInfo(IpInfo ipInfo) {
 		if(ipInfo!=null&&ipInfo.getData()!=null){
 			IpData ipData=ipInfo.getData()；
 			tv_country.setText(ipData.getCountry())；
 			tv_area.setText(ipData.getArea())；
 			tv_city.setText(ipData.getCity())；
 		}
 	}
 	@Override
 	public void showLoading() {
 		mDialog.show()；
 	}
 	@Override
 	public void hideLoading() {
 		if(mDialog.isShowing()) {
 			mDialog.dismiss()；
 		}
 	}
 	@Override
 	public void showError() {
 		Toast.makeText(getActivity().getApplicationContext(),＂网络出错＂,
 		Toast.LENGTH_SHORT).show()；
 	}
 	@Override
 	public boolean isActive() {
 		return isAdded()；
 	}
}
```

在上面代码注释 1 处通过实现 setPresenter 方法来注入 IpInfoPresenter。在注释 2 处则调用IpInfoPresenter 的 getIpInfo 方法来获取 IP 地址的信息。另外，IpInfoFragment 实现了 View 接口， 用来接收 IpInfoPresenter 的回调并更新界面。那么 IpInfoFragment 是在哪里调用 setPresenter 来 注入 IpInfoPresenter 的呢？答案是在 IpInfoActivity 中，代码如下所示：

![image-20241021160009085](https://gitee.com/ForeverHamburger/picgo_imgs1/raw/master/202410211600141.png)

在这个例子中 IpInfoActivity 并不作为 View 层，而是作为 View、Model 和 Presenter 三层的 纽带。在上面代码注释 1 处的代码新建 IpInfoFragment，接着通过注释 2 处的代码来将 IpInfoFragment 添加到 IpInfoActivity 中。紧接着创建 IpInfoTask，并将它和 IpInfoFragment 作为 参数传入 IpInfoPresenter，并在注释 3 处将 IpInfoPresenter 注入到 IpInfoFragment 中。可以看到 IpInfoPresenter 和 IpInfoFragment 是互相注入的。注释 2 处的 ActivityUtils 的代码如下所示：

![image-20241021160022877](https://gitee.com/ForeverHamburger/picgo_imgs1/raw/master/202410211600928.png)

上面的代码负责提交事务，将 Fragment 添加到 Activity 中。

View 和 Model 之间没有联系，View 与 Presenter 通过接口进行交互， 并在 Activity 中进行相互注入。Model 层的 NetTask 在 Activity 中注入 Presenter，并等待 Presenter 调用。
