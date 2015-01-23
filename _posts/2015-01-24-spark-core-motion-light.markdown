---
layout: post
title:  "Creating a Motion Sensing Light with Spark Core"
date:   2015-01-24 13:00:00
comments: true
categories: spark
---

> A nice beginner guide to a simple [IoT](http://en.wikipedia.org/wiki/Internet_of_Things) device for controlling lights.

![light on]({{ site.url }}/assets/light_on.gif)

#### Guide
* [Supplies](#supplies)
* [Hardware Set Up](#hardware)
* [Code](#code)
* [Conclusion](#conclusion)

<br>

### <a name="supplies">Supplies</a>
--------------------------

<br>
Here is a list of all the things I used, hopefully you already have some of this stuff, or possbily you already have a relay you can use. Otherwise, I *strongly* recommend the power tail switch to avoid having to mess with life threatening power.

#### 1. [Spark Core](http://spark.io)

<img width='200' src='http://d3uifzcxlzuvqz.cloudfront.net/images/stories/jreviews/tn/tn_1501_spark-core4-1367609352.jpg' />

#### 2. [Power Tail switch](http://www.powerswitchtail.com/Pages/default.aspx)

<img width="200" src='http://www.powerswitchtail.com/siteimages/powerswitch%20tail%20ii.jpg' />

#### 3. <a href="http://www.amazon.com/gp/product/B00FDPO9B8/ref=as_li_tl?ie=UTF8&camp=1789&creative=9325&creativeASIN=B00FDPO9B8&linkCode=as2&tag=dannpete-20&linkId=ZVTCBFMJ2GBPVJQP">Human Sensor (HC-SR501)</a>


<a href="http://www.amazon.com/gp/product/B00FDPO9B8/ref=as_li_tl?ie=UTF8&camp=1789&creative=9325&creativeASIN=B00FDPO9B8&linkCode=as2&tag=dannpete-20&linkId=ZVTCBFMJ2GBPVJQP"><img border="0" src="http://ws-na.amazon-adsystem.com/widgets/q?_encoding=UTF8&ASIN=B00FDPO9B8&Format=_SL160_&ID=AsinImage&MarketPlace=US&ServiceVersion=20070822&WS=1&tag=dannpete-20" ></a><img src="http://ir-na.amazon-adsystem.com/e/ir?t=dannpete-20&l=as2&o=1&a=B00FDPO9B8" width="1" height="1" border="0" alt="" style="border:none !important; margin:0px !important;" />

#### 4. <a href="http://www.amazon.com/gp/product/B00A6SOGC4/ref=as_li_tl?ie=UTF8&camp=1789&creative=9325&creativeASIN=B00A6SOGC4&linkCode=as2&tag=dannpete-20&linkId=SVG45KOJBI32J64I">Female to Male Jumper Cables</a>

<a href="http://www.amazon.com/gp/product/B00A6SOGC4/ref=as_li_tl?ie=UTF8&camp=1789&creative=9325&creativeASIN=B00A6SOGC4&linkCode=as2&tag=dannpete-20&linkId=SVG45KOJBI32J64I"><img border="0" src="http://ws-na.amazon-adsystem.com/widgets/q?_encoding=UTF8&ASIN=B00A6SOGC4&Format=_SL160_&ID=AsinImage&MarketPlace=US&ServiceVersion=20070822&WS=1&tag=dannpete-20" ></a><img src="http://ir-na.amazon-adsystem.com/e/ir?t=dannpete-20&l=as2&o=1&a=B00A6SOGC4" width="1" height="1" border="0" alt="" style="border:none !important; margin:0px !important;" />


#### 5. <a href="http://www.amazon.com/gp/product/B005GYB93M/ref=as_li_tl?ie=UTF8&camp=1789&creative=9325&creativeASIN=B005GYB93M&linkCode=as2&tag=dannpete-20&linkId=Z2MVFAHQWBMKROCR">Preformed cables</a>

<a href="http://www.amazon.com/gp/product/B005GYB93M/ref=as_li_tl?ie=UTF8&camp=1789&creative=9325&creativeASIN=B005GYB93M&linkCode=as2&tag=dannpete-20&linkId=Z2MVFAHQWBMKROCR"><img border="0" src="http://ws-na.amazon-adsystem.com/widgets/q?_encoding=UTF8&ASIN=B005GYB93M&Format=_SL160_&ID=AsinImage&MarketPlace=US&ServiceVersion=20070822&WS=1&tag=dannpete-20" ></a><img src="http://ir-na.amazon-adsystem.com/e/ir?t=dannpete-20&l=as2&o=1&a=B005GYB93M" width="1" height="1" border="0" alt="" style="border:none !important; margin:0px !important;" />

### <a name="hardware">Hardware Set Up</a>
--------------------------

<br>

Here is a quick sketched layout showing how I hooked up the hardward. In this picture, the 3 pin LED is suppose to represent the HC-SR501. 

>Note that I had to connect it to the Vin of the Spark Core because the HC-SR501 actually wants to run at 5v. 

![hardward Diagram]({{ site.url }}/assets/sketch.png)

Here is the actual layout of the electronics. 

![Actual Layout]({{ site.url }}/assets/actual.jpg)


### <a name="code">Code</a>
--------------------------

<br>

There's actually some pretty cool stuff going on here. First off, realize that the `out` pin for the HC-SR501 [(data sheet)](http://elecfreaks.com/store/download/datasheet/sensor/DYP-ME003/Specification.pdf) has to be set to a `INPUT_PULLDOWN` because it does not pull down otherwise and would never properly trigger the interrupt.

Also we use the idea of a [interrupt](https://en.wikipedia.org/wiki/Interrupt), which is something close to my Computer Engineering heart, so instead of constantly polling the pin to see if the state has changed. The pin will let us know when the state changes and fire off our function `light_on` when the state is `RISING` [(spark doc)](http://docs.spark.io/firmware/#interrupts-attachinterrupt).

Next, to handle turning the light off after 15 mins we use the [`millis()`](http://docs.spark.io/firmware/#time-millis) function. This function returns the number of milliseconds since the spark core was powered on. So, using a two second delay in our `loop()` function, we are checking that our `set_time` + 15 mins is less then the current time every two seconds. 

Finally, in our `light_on()` function we make sure to reset the `set_time` incase the interrupt is triggered again while it is areadly on.

{% highlight C %}
// functions
void light_on(void);
void light_off(void);
// variables
int ledPin = D6;
volatile int state = LOW;
unsigned long current_time;
volatile unsigned long set_time;

// 15 mins
#define FIFTEEN_MIN_MILLIS (15 * 60 * 1000)

void setup()
{
  pinMode(D0, INPUT_PULLDOWN );
  pinMode(ledPin, OUTPUT);
  attachInterrupt(D0, light_on, RISING);
  set_time = millis(); 
}

void loop()
{
  digitalWrite(ledPin, state);
  current_time = millis();
  if( set_time+FIFTEEN_MIN_MILLIS < current_time ){
    light_off();   
  }
  delay(2000);
}

void light_on(){
 set_time = millis();
 state = HIGH;
}

void light_off(){
 set_time = set_time-FIFTEEN_MIN_MILLIS;
 state = LOW;
}
{% endhighlight %}



### <a name="conclusion">Conclusion</a>
--------------------------

<br>

Quite simple and really easy to get set up once you have all the parts. Also its nice to have something home automated that actually is easier to use. Before I had the lights controlled by a button on a iOS app. While cool, this became impractical quick as it was a bigger inconvenience then this simple implementation. 

I do plan to add back in the functionality to turn the light off with either a siri command as sometimes you want the lights out when you watch a movie, but I'll save that for either another blog post or an update to this one. 

