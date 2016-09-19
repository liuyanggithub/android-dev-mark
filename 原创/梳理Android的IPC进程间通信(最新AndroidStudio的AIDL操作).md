## 前言
前面梳理了Android的线程间的通信[《Thread、Handler和HandlerThread关系何在？》](http://blog.csdn.net/ly502541243/article/details/52414637)，这些都是在同一个进程中，那进程间的通信，或者说不同的应用间的通信该如何实现呢？这个时候就要用到AIDL(*Android Interface Definition Language*Android接口定义语言 )。
## 使用方法（AndroidStudio）
我发现现在AIDL的教程基本上还是eclipse的，但是在AndroidStudio里面使用AIDL还是有一些不同的，来看看怎么用，首先新建一个工程当做server服务端：

创建好后在任意文件夹右键**New-->AIDL-->AIDL File**，编辑文件名后会自动在**src/main**目录下面新建**aidl**文件夹，包的目录结构如下：
- main
  - aidl
    - com.example.tee.testapplication.aidl 
  - java
     - com.example.tee.testapplication
  - res
  - AndroidManifest.xml
 
自动生成的aidl文件如下：

```
// AidlInterface.aidl
package com.example.tee.testapplication.aidl;

// Declare any non-default types here with import statements

interface AidlInterface {
    /**
     * Demonstrates some basic types that you can use as parameters
     * and return values in AIDL.
     */
    void basicTypes(int anInt, long aLong, boolean aBoolean, float aFloat,
            double aDouble, String aString);
}

```
我们可以看到aidl文件的代码格式跟java很像，支持java的基础类型以及List、Map等，如果是自定义类的话需要手动导入，我们后面再说，先来最简单的，新建一个 **IMyAidlInterface.aidl**文件，修改如下：
```
package com.example.tee.testapplication.aidl;

interface IMyAidlInterface {
     String getValue();
}
```
在接口中定义一个**getValue**方法，返回一个字符串，现在可以编译一下工程，找到**app/build/generated/source/aidl/debug**目录，在我们应用包名下会发现生成了一个Interface类，名字跟我们定义的aidl的文件名字一样，这说明其实aidl文件在最后还是会转换成接口来实现，而且这个文件不需要我们维护，在编译后自动生成。


然后新建一个类继承Service：

```
public class MAIDLService extends Service{
    public class MAIDLServiceImpl extends IMyAidlInterface.Stub{
        @Override
        public String getValue() throws RemoteException {
            return "get value";
        }
    }
    @Nullable
    @Override
    public IBinder onBind(Intent intent) {
        return new MAIDLServiceImpl();
    }
}

```
在MAIDLService类中定义一个内部类继承**IMyAidlInterface.Stub**，并且重写我们在aidl也就是在接口中定义的**getValue**方法，返回字符串**get value**。

到了这里，我们就新建好了这个服务端，作用是在调用后返回一个字符串，最后在AndroidManifest文件中声明：
```
<service
     android:name=".MAIDLService"
     android:process=":remote"//加上这句的话客户端调用会创建一个新的进程
     android:exported="true"//默认就为true，可去掉，声明是否可以远程调用
    >
     <intent-filter>
        <category android:name="android.intent.category.DEFAULT" />
        <action android:name="com.example.tee.testapplication.aidl.IMyAidlInterface" />
     </intent-filter>
</service>
```
android:process=":remote"这一行的作用是声明是否调用时**新建进程**，接下来写客户端代码，新建一个工程，将刚才创建的aidl文件拷贝到这个工程中，注意同样也是要放在aidl文件夹下，然后在MainActivity中编写代码如下：

```
public class MainActivity extends AppCompatActivity {
    private TextView mValueTV;
    private IMyAidlInterface mAidlInterface = null;
    private ServiceConnection mServiceConnection = new ServiceConnection() {
        @Override
        public void onServiceConnected(ComponentName name, IBinder service) {
            mAidlInterface = IMyAidlInterface.Stub.asInterface(service);
        }

        @Override
        public void onServiceDisconnected(ComponentName name) {

        }
    };
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        Intent intent = new Intent("com.example.tee.testapplication.aidl.IMyAidlInterface");
        bindService(intent, mServiceConnection, BIND_AUTO_CREATE);
        mValueTV = (TextView) findViewById(R.id.tv_test_value);
        mValueTV.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                try {
                    mValueTV.setText(mAidlInterface.getValue());
                } catch (RemoteException e) {
                    e.printStackTrace();
                }
            }
        });
    }
    
    @Override
    protected void onDestroy() {
        if(mAidlInterface != null){
            unbindService(mServiceConnection);
        }
        super.onDestroy();
    }
}
```
注意这里新建Intent的传入的参数字符串是在**manifest**里面自定义的**action**标签，并且在**onDestroy**记得取消绑定服务。

执行结果就是我们在点击TextView时会显示服务端给我们返回的**get value**字符串

## 自定义的对象
刚才我们使用的是基础类型**String**，在使用我们自己定义的类的时候用上面的方法是不行的，用我们自定义的类需要手动导入，修改刚才我们创建的作为服务端的工程

首先在开始生成的aidl包下（**所有aidl相关的文件都要放在这个包下**）新建Student.java

```
public class Student implements Parcelable{
    public String name;
    public int age;
    protected Student(Parcel in) {
        readFromParcel(in);
    }
    public Student() {
    }

    public static final Creator<Student> CREATOR = new Creator<Student>() {
        @Override
        public Student createFromParcel(Parcel in) {
            return new Student(in);
        }

        @Override
        public Student[] newArray(int size) {
            return new Student[size];
        }
    };

    @Override
    public int describeContents() {
        return 0;
    }

    @Override
    public void writeToParcel(Parcel dest, int flags) {
        dest.writeInt(age);
        dest.writeString(name);
    }

    public void readFromParcel(Parcel in){
        age = in.readInt();
        name = in.readString();
    }

    @Override
    public String toString() {
        return String.format(Locale.ENGLISH, "STUDENT[%s:%d]", name, age);
    }
}
```
需要实现**Parcelable**序列化接口，AndroidStudio会自动生成静态内部类**CREATOR**和**describeContents**方法，这些部分我们都不需要修改，用自动生成的就好。然后重写**writeToParcel**方法，自定义**readFromParcel**方法，注意这两个方法里面的**属性顺序必须一致**，一个是写入，一个是读取。在构造方法Student(Parcel in)中调用readFromParcel(in)方法。

接下来新建Student.aidl文件（也是在aidl包中）：
```
// Student.aidl
package com.example.tee.testapplication.aidl;

// Declare any non-default types here with import statements
parcelable Student;

```
注意这里Student前面的关键字parcelable首字母是**小写**哦，再修改IMyAidlInterface.aidl文件如下：

```
// IMyAidlInterface.aidl
package com.example.tee.testapplication.aidl;

// Declare any non-default types here with import statements
import com.example.tee.testapplication.aidl.Student;

interface IMyAidlInterface {
     Student getStudent();
     void setStudent(in Student student);
     String getValue();
}

```
定义了两个方法，一个是设置Student，一个是获取Student，在**setStudent**这个方法注意参数在类型前面有个**in**关键字，在**aidl**里参数分为**in**输入，**out**输出

现在在MAIDLService.java中重写新加的两个方法：

```
private Student mStudent;
public class MAIDLServiceImpl extends IMyAidlInterface.Stub{


    @Override
    public Student getStudent() throws RemoteException {
        return mStudent;
    }

    @Override
    public void setStudent(Student student) throws RemoteException {
        mStudent = student;
    }

    @Override
    public String getValue() throws RemoteException {
            return "get value : " + Thread.currentThread().getName() + Thread.currentThread().getId();
    }
}
```
服务端代码修改完毕，来到客户端工程，同样要把刚才的aidl包下的文件拷贝覆盖过来，保持两边一致，然后在MainActivity.java中修改如下：
```
mValueTV = (TextView) findViewById(R.id.tv_test_value);
mStudentTV = (TextView) findViewById(R.id.tv_test_student);
mValueTV.setOnClickListener(new View.OnClickListener() {
    @Override
    public void onClick(View v) {
        try {
            mValueTV.setText(mAidlInterface.getValue());
        } catch (RemoteException e) {
            e.printStackTrace();
        }
    }
});
mStudentTV.setOnClickListener(new View.OnClickListener() {
    @Override
    public void onClick(View v) {
        try {
            Student student = new Student();
            student.age = 10;
            student.name = "Tom";
            mAidlInterface.setStudent(student);
            mStudentTV.setText(mAidlInterface.getStudent().toString());
        } catch (RemoteException e) {
            e.printStackTrace();
        }
    }
});
```
现在编译工程，会发现工程会报错，找不到类Student，我们需要在app目录下的build.gradle文件添加代码如下：

```
android {
    sourceSets {
        main {
            manifest.srcFile 'src/main/AndroidManifest.xml'
            java.srcDirs = ['src/main/java', 'src/main/aidl']
            resources.srcDirs = ['src/main/java', 'src/main/aidl']
            aidl.srcDirs = ['src/main/aidl']
            res.srcDirs = ['src/main/res']
            assets.srcDirs = ['src/main/assets']
        }
    }
}
```
也就是指定一下文件目录，现在再编译就没有问题了
## 总结
Android的IPC使用起来还是挺简单的，AIDL文件的语法也跟我们平时使用接口的时候很相似，但是它只支持基础类型，只能引用AIDL文件，需要使用自定义类的时候要稍微麻烦一点。