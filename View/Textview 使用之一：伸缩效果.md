###android开发过程中，经常遇到Textview展示不完全的情况。遇到此情况，通常的处理是：


- 方案一、Textview添加android:ellipsize属性，让展示不完的部分使用省略号代替。


- 方案二、Textview采用走马灯效果，使其滚动展示全部文本内容。


对于方案一，如果想查看被省略后的内容，如何实现？微信的评论列表，豌豆荚视频详情介绍都有类似使用场景。
下面来看下Demo例子的收起效果，文本内容没有展示完全，使用省略号代替，提示“更多”和向下箭头标识，截图如下：

当点击“更多”和向下箭头时，被省略的内容全部展示出来，提示“更多”和向上收起标识箭头：


对于以上效果，实现思路如下：


- 1、设置Textview默认展示固定行，比如3行，内容展示不完全，在Textview尾部使用省略号代替。

xml文件内容为：

    <RelativeLayout xmlns:android="http://schemas.android.com/apk/res/android"  
    xmlns:tools="http://schemas.android.com/tools"  
    android:layout_width="fill_parent"  
    android:layout_height="fill_parent"  
    tools:context=".MainActivity" >  
      
    
    <!-- 显示文本 -->  
    <TextView  
    android:id="@+id/text_content"  
    android:layout_width="fill_parent"  
    android:layout_height="wrap_content"  
    android:ellipsize="end"  
    android:maxLines="3"  
    android:singleLine="false" />  
      
    <!-- 更多和箭头 -->  
    <RelativeLayout  
    android:id="@+id/show_more"  
    android:layout_below="@id/text_content"  
    android:layout_alignParentRight="true"  
    android:layout_width="wrap_content"  
    android:layout_height="wrap_content"  
    android:layout_marginTop="3dip"  
    >  
      
    <TextView  
    android:layout_width="wrap_content"  
    android:layout_height="wrap_content"  
    android:layout_alignParentRight="true"  
    android:textSize="13sp"  
    android:textColor="#999"  
    android:layout_marginRight="34dip"  
    android:text="更多" />  
      
      
    <ImageView  
    android:id="@+id/spread"  
    android:layout_width="wrap_content"  
    android:layout_height="wrap_content"  
    android:layout_alignParentRight="true"  
    android:background="@drawable/ic_details_more" />  
      
    <ImageView  
    android:id="@+id/shrink_up"  
    android:layout_width="wrap_content"  
    android:layout_height="wrap_content"  
    android:layout_alignParentRight="true"  
    android:background="@drawable/ic_shrink_up"  
    android:visibility="gone" />  
    </RelativeLayout>  
    </RelativeLayout>  






- 2、点击“更多”和向下箭头时，通过Textview的setMaxLines()方法改变Textview的最大行数。即可实现上述效果。

Java代码为：

    package com.example.testdemo;  
    import android.app.Activity;  
    import android.os.Bundle;  
    import android.view.View;  
    import android.view.View.OnClickListener;  
    import android.widget.ImageView;  
    import android.widget.RelativeLayout;  
    import android.widget.TextView;  
      
    public class MainActivity extends Activity implements OnClickListener {  
      
    private static final int VIDEO_CONTENT_DESC_MAX_LINE = 3;// 默认展示最大行数3行  
    private static final int SHOW_CONTENT_NONE_STATE = 0;// 扩充  
    private static final int SHRINK_UP_STATE = 1;// 收起状态  
    private static final int SPREAD_STATE = 2;// 展开状态  
    private static int mState = SHRINK_UP_STATE;//默认收起状态  
      
    private TextView mContentText;// 展示文本内容  
    private RelativeLayout mShowMore;// 展示更多  
    private ImageView mImageSpread;// 展开  
    private ImageView mImageShrinkUp;// 收起  
      
    @Override  
    protected void onCreate(Bundle savedInstanceState) {  
    super.onCreate(savedInstanceState);  
    setContentView(R.layout.activity_main);  
      
    initView();  
    initData();  
    }  
      
    private void initView() {  
    mContentText = (TextView) findViewById(R.id.text_content);  
    mShowMore = (RelativeLayout) findViewById(R.id.show_more);  
    mImageSpread = (ImageView) findViewById(R.id.spread);  
    mImageShrinkUp = (ImageView) findViewById(R.id.shrink_up);  
    mShowMore.setOnClickListener(this);  
      
    }  
      
    private void initData() {  
    mContentText.setText(R.string.txt_info);  
    }  
      
    @Override  
    public void onClick(View v) {  
    // TODO Auto-generated method stub  
    switch (v.getId()) {  
    case R.id.show_more: {  
    if (mState == SPREAD_STATE) {  
    mContentText.setMaxLines(VIDEO_CONTENT_DESC_MAX_LINE);  
    mContentText.requestLayout();  
    mImageShrinkUp.setVisibility(View.GONE);  
    mImageSpread.setVisibility(View.VISIBLE);  
    mState = SHRINK_UP_STATE;  
    } else if (mState == SHRINK_UP_STATE) {  
    mContentText.setMaxLines(Integer.MAX_VALUE);  
    mContentText.requestLayout();  
    mImageShrinkUp.setVisibility(View.VISIBLE);  
    mImageSpread.setVisibility(View.GONE);  
    mState = SPREAD_STATE;  
    }  
    break;  
    }  
    default: {  
    break;  
    }  
    }  
    }  
      
    }  


下面为Demo示例下载链接

[http://download.csdn.net/detail/improveyourself/7455779](http://download.csdn.net/detail/improveyourself/7455779 "下载链接")
