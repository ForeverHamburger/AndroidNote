# 【Android】Fragment

## Fragment的动态创建

1. 创建一个Fragment
2. 布局代码中用一个容器来承接，但不直接绑定
3. 在代码中用FragmentManager、FragmentTransaction添加Fragment到容器中

## 在Activity中设置一个容器

![img](https://gitee.com/ForeverHamburger/picgo_imgs1/raw/master/202410252119768.png)

## 在Activity的代码中，设置动态创建Fragment的代码

![img](https://gitee.com/ForeverHamburger/picgo_imgs1/raw/master/202410252120389.png)



```java
public class DynamicFragmentActivity extends AppCompatActivity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        EdgeToEdge.enable(this);
        setContentView(R.layout.activity_dynamic_fragment);
        ViewCompat.setOnApplyWindowInsetsListener(findViewById(R.id.main), (v, insets) -> {
            Insets systemBars = insets.getInsets(WindowInsetsCompat.Type.systemBars());
            v.setPadding(systemBars.left, systemBars.top, systemBars.right, systemBars.bottom);
            return insets;
        });

        if(savedInstanceState == null){
            FragmentManager supportFragmentManager = getSupportFragmentManager();
            FragmentTransaction fragmentTransaction = supportFragmentManager.beginTransaction();
            fragmentTransaction.add(R.id.fragment_container_view_tag, ExampleFragment1.class,null)
                    .setReorderingAllowed(true)
                    .addToBackStack(null)
                    .commit();
        }
    }
}
```

1. `if(savedInstanceState == null){`：
   1. 这是一个条件判断语句。`savedInstanceState`是一个Bundle对象，通常用于保存Fragment的实例状态。如果`savedInstanceState`为null，说明Fragment是首次创建，而不是从保存的状态中恢复。
2. `FragmentManager supportFragmentManager = getSupportFragmentManager();`：
   1. 获取当前Activity的FragmentManager实例。FragmentManager用于管理Fragment的生命周期和事务。
3. `FragmentTransaction fragmentTransaction = supportFragmentManager.beginTransaction();`：
   1. 创建一个Fragment事务（FragmentTransaction），用于执行添加、替换、删除Fragment等操作。
4. `fragmentTransaction.add(R.id.fragment_container_view_tag, ExampleFragment1.class,null)`：
   1. 向FragmentManager中添加一个新的Fragment。`R.id.fragment_container_view_tag`是Fragment要添加到的容器视图的ID。`ExampleFragment1.class`是要添加的Fragment的类。`null`是用于标识Fragment的标签，这里传入null表示不使用标签。
5. `.setReorderingAllowed(true)`：
   1. 允许Fragment事务中的视图重新排序。这通常用于动画效果，使Fragment的添加或替换看起来更平滑。
6. `.addToBackStack(null)`：
   1. 将这个Fragment事务添加到后退栈中。用户可以通过按后退键来返回到这个Fragment事务之前的状态。传入null表示不使用标签。
7. `.commit();`：
   1. 提交Fragment事务。这会触发Fragment的生命周期方法，如`onCreate`、`onStart`等，并最终将Fragment显示在界面上。

## 两种方式创建Fragment的生命周期的区别

<img src="https://gitee.com/ForeverHamburger/picgo_imgs1/raw/master/202410252120448.png" alt="img" style="zoom:50%;" />

### Fragment生命周期

Fragment的四种状态和回调状态：

运行状态：当一个碎片是可见的，并且它所关联的活动正处于运行状态时，该碎片也处于运行状态。

暂停状态：当一个活动进人暂停状态时（由于另一个未占满屏幕的活动被添加到了栈顶），与它相关联的可见碎片就会进人到暂停状态。

停止状态：当一个活动进入停止状态时，与它相关联的碎片就会进入到停止状态，或者通过调FragmentTransaction的remove()、replace()方法将碎片从活动中移除，但如果在事务提交之前调用addToBackStack()方法，这时的碎片也会进入到停止状态。

销毁状态：碎片总是依附于活动而存在的，因此当活动被销毁时，与它相关联的碎片就会进人到销毁状态。或者通过调用FragmentTransaction的remove()、replace()方法将碎片从活动中移除，但在事务提交之前并没有调用addToBackStack()方法，这时的碎片也会进人到销毁状态。

回调方法：

> onAttach()：当活动和碎片建立关联时使用
>
> OnCreateView：当为碎片创建视图时使用
>
> onActivityCreated()：确保与碎片相关联的活动一定已经创建完毕的时候调用。
>
> onDestroyView()。当与碎片关联的视图被移除的时候调用。
>
> onDetach()。当碎片和活动解除关联的时候调用。
