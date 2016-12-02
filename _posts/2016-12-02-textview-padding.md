---
layout: post
title:  "TextView 去掉自适应默认的fontpadding"
desc: " TextView 去掉自适应默认的fontpadding"
keywords: "textview,padding"
date: 2016-12-02
categories: [Android]
tags: [TextView]
icon: fa-bookmark-o
---
&emsp;&emsp;最近在项目中使用textview时发现在使用android:layout_height="wrap_content"这个属性设置后，textview会有默认的padding，也就是fontpadding。这样就会造成textview和其他view中间的间距会比自己的设置的大。那么我们怎么来remove掉这个间距呢？  
　　第一、先试试设置includefontpadding=false ，如果不能达到目的的话，可以按照第二种方法。  
　　第二、实现自定义TextView，只需继承自TextView然后重写onDraw方法就可以了。
　　
```
FontMetricsInt fontMetricsInt;
@Override
protected void onDraw(Canvas canvas) {
    if (adjustTopForAscent){//设置是否remove间距，true为remove
        if (fontMetricsInt == null){
            fontMetricsInt = new FontMetricsInt();
            getPaint().getFontMetricsInt(fontMetricsInt);
        }
        canvas.translate(0, fontMetricsInt.top - fontMetricsInt.ascent);
    }
    super.onDraw(canvas);
}
```

&emsp;&emsp;第二种方法一般能达到目的，如果还是不能的话，那只能使用marginTop等于负值来实现了，不过不推荐这种方法。