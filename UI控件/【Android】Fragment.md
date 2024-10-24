# 【Android】Fragment

一、Fragment的简单用法
1、制作Fragment
1.1 新建一个布局文件left_fragment.xml
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:orientation="vertical"
    android:layout_width="match_parent"
    android:layout_height="match_parent">

    <Button
        android:id="@+id/Button"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_gravity="center_horizontal"
        android:text="Button"/>


</LinearLayout>
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
1.2 新建一个LeftFragment类继承Fragment
public class LeftFragment extends Fragment {

    LeftFragmentBinding binding;
    @Nullable
    @Override
    public View onCreateView(@NonNull LayoutInflater inflater, @Nullable ViewGroup container, @Nullable Bundle savedInstanceState) {
        /**
         * 普通写法：
         *  View view = inflater.inflate(R.layout.left_fragment,container,false);
         *  retuen view;
         *  不需要重写onDestroyView
         */
    
        binding = LeftFragmentBinding.inflate(inflater,container,false);
         return binding.getRoot();
    }
    
    public void f(){
        Log.d("F","SSS");
    }
    
    @Override
    public void onDestroyView() {
        super.onDestroyView();
        binding = null;
    }
}

1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
重写onCreateView方法。

因为要保持binding的生命周期在有效范围内，所以需要在onDestroy中让binding=null。

1.3 在布局文件activity_main.xml加载这个碎片
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools"
    android:orientation="horizontal"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    tools:context=".MainActivity">

    <fragment
        android:id="@+id/Leftfragment"
        android:name="com.example.fragment.LeftFragment"
        android:layout_width="0dp"
        android:layout_height="match_parent"
        android:layout_weight="1"/>

</LinearLayout>

1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
使用android:name指定加载的类，注意需要有完整的包名。

2、动态加载Fragment
2.1 新建一个用于替换的布局文件
新建一个用于替换的布局文件，another_left_fragment.xml

<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:orientation="vertical"
    android:background="#ffff00"
    android:layout_width="match_parent"
    android:layout_height="match_parent">

    <TextView
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:textSize="24sp"
        android:text="This is Fragment"/>

</LinearLayout>
1
2
3
4
5
6
7
8
9
10
11
12
13
14
2.2 新建一个AnotherLeftFragment类继Fragment
public class AnotherRightFragment  extends Fragment {
    @Nullable
    @Override
    public View onCreateView(@NonNull LayoutInflater inflater, @Nullable ViewGroup container, @Nullable Bundle savedInstanceState) {
        return inflater.inflate(R.layout.another_right_fragment,container,false);
    }
}
1
2
3
4
5
6
7
3.3 在MainActivity中动态替换
public class MainActivity extends AppCompatActivity implements View.OnClickListener{

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);


        //获取FragmentManager实例，通过向下转型为LeftFragment
        LeftFragment leftFragment = (LeftFragment) getSupportFragmentManager().findFragmentById(R.id.Leftfragment);
    
        //调用button按钮的监听器
        leftFragment.binding.Button.setOnClickListener(this);


    }


    @Override
    public void onClick(View view) {
        int Id = view.getId();
        if(Id == R.id.Button){
    
            /**
             * 匿名替换的方式：
             *  getSupportFragmentManager().beginTransaction().replace(R.id.Leftfragment,new AnotherRightFragment()).commit();
             */
    
            replaceFragment(new AnotherRightFragment());
        }
    }
    
    private void replaceFragment(Fragment fragment){
        //1、获取FragmentManager实例
        FragmentManager fragmentManager = getSupportFragmentManager();
    
        //2、开启一个事物
        FragmentTransaction transaction = fragmentManager.beginTransaction();
    
        //3、向指定名称Id的容器内替换碎片实例
        transaction.replace(R.id.Leftfragment,fragment);
    
        //4、提交事物
        transaction.commit();
    
    }
}

1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28
29
30
31
32
33
34
35
36
37
38
39
40
41
42
43
44
45
46
47


3、在碎片中模拟返回栈
我们成功实现了向活动中动态添加碎片的功能，不过尝试一下就会发现，通过点击按钮添加了一个碎片之后，这时按下Back键程序就会直接退出。如果这里我们想模仿类似于返回栈的效果，按下Bck键可以回到上一个碎片，该如何实现呢？其实很简单FragmentTransaction中提供了一个addToBackStack()方法，可以用于将一个事务添加到返回栈中，修改MainActivity中的代码，如下所示：

private void replaceFragment(Fragment fragment){
    //获取FragmentManager实例
    FragmentManager fragmentManager = getSupportFragmentManager();

    //开启一个事物
    FragmentTransaction transaction = fragmentManager.beginTransaction();
    
    //向指定名称Id的容器内替换碎片实例
    transaction.replace(R.id.Rightfragment,fragment);
    
    //添加返回栈
    transaction.addToBackStack(null);
    
    //提交事物
    transaction.commit();

}

1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
4、Fragment和活动间的通信
4.1 从Activit中获取Fragment
其实我们已经用过很多次了，就是先获取LeftFragment的实例，如何通过实例中的binding访问Button的行为，这就是获取一个碎片，如何调用碎片中的方法。

        //获取FragmentManager实例，通过向下转型为LeftFragment
        LeftFragment leftFragment = (LeftFragment) getSupportFragmentManager().findFragmentById(R.id.Leftfragment);
1
2
4.2 从Fragment中获得Activit
使用getActivit即可，因为获得的活动本身就是一个Context对象

MainActivit activit = (MainActivit) getActivit();
1
二、Fragment的生命周期
1、Fragment的四种状态和回调
状态：

运行状态：当一个碎片是可见的，并且它所关联的活动正处于运行状态时，该碎片也处于运行状态。
暂停状态：当一个活动进人暂停状态时（由于另一个未占满屏幕的活动被添加到了栈顶），与它相关联的可见碎片就会进人到暂停状态。
停止状态：当一个活动进入停止状态时，与它相关联的碎片就会进入到停止状态，或者通过调用FragmentTransaction的remove()、replace()方法将碎片从活动中移除，但如果在事务提交之前调用addToBackStack()方法，这时的碎片也会进入到停止状态。
销毁状态：碎片总是依附于活动而存在的，因此当活动被销毁时，与它相关联的碎片就会进人到销毁状态。或者通过调用FragmentTransaction的remove()、replace()方法将碎片从活动中移除，但在事务提交之前并没有调用addToBackStack()方法，这时的碎片也会进人到销毁状态。
回调方法：

onAttach()：当活动和碎片建立关联时使用
OnCreateView：当为碎片创建视图时使用
onActivityCreated()：确保与碎片相关联的活动一定已经创建完毕的时候调用。
onDestroyView()。当与碎片关联的视图被移除的时候调用。
onDetach()。当碎片和活动解除关联的时候调用。
2、Fragment的生命周期

旧版生命周期如下：


打开一个碎片生命周期的调用情况：


一般**onCreateView()**用于初始化Fragment的视图，**onViewCreated()一般用于初始化视图内各个控件，而onCreate()**用于初始化与Fragment视图无关的变量。

另外值得一提的是，在碎片中你也是可以通过onSaveInstanceState()方法来保存数据的，因为进入停止状态的碎片有可能在系统内存不足的时候被回收。保存下来的数据在onCreate()、onCreateview()和onActivityCreated()这3个方法中你都可以重新得到，它们都含有一个Bundle类型的savedInstanceState参数。
