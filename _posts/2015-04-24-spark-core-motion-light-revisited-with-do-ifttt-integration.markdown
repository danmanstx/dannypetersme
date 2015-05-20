---
layout: post
title:  "Creating a Motion Sensing Light with a Spark Core - Revisited with DO IFTTT integration"
date:   2015-04-24 13:00:00
comments: true
categories: spark
---

> [original article]({% post_url 2015-01-24-spark-core-motion-light %})
>
> New in this Artice: Code Review and Adding in IFTTT Do functions. 

![light on]({{ site.url }}/assets/light_on.gif)



#### Guide
* [New Code](#new_code)
* [IFTTT Do Functions](#do_func)
* [Conclusion](#conclusion)

<br>

### <a name="new_code">New Code</a>
--------------------------

<br>
After posting about my experience on the [spark.io forums, now particle.io](https://community.particle.io/t/hc-sr501-motion-detecting-light-sensor/9561/6). I received some great feedback and made some changes to my code to make it run smoother and decided to add the ability to toggle the lights on and off for a set amount of time. One of the big changes here are moving some functionality to seperate functions. Including functions to set the lights off (`set_light_off`) for a set amount of time and then a function to overwrite that setting and turn the lights on (`set_light_on`). Another change was to remove the delay from the loop, this removes the noticable lag between triggering the motion sensor and the light turning on. Also as recommend in the thread, there is now a flag `can_turn_light_on` to avoid banging away on the digital pin to write it high every time it loops. The final code that I am running at home is shown below. You can view the [original article]({% post_url 2015-01-24-spark-core-motion-light %}) to see how the circuit is laid out as well as the supplies used.

{% highlight C %}
// functions declarations
void trigger_light_on(void);
void light_off(void);
int set_light_on(String args);
int set_light_off(String args);

// variable declarations
int ledPin = D6;
volatile int state = LOW;
unsigned long current_time;
volatile unsigned long set_time;
volatile bool hold_light_off = false;
volatile bool can_turn_light_on = false;

// CONSTANTS
#define FIFTEEN_MIN_MILLIS (15 * 60 * 1000)
#define THREE_HOURS_MILLIS (3 * 60 * 60 * 1000)

void setup(){
  pinMode(D0, INPUT_PULLDOWN );
  pinMode(ledPin, OUTPUT);
  attachInterrupt(D0, trigger_light_on, RISING);
  set_time = millis(); 
  Spark.function("LightsOn", set_light_on);
  Spark.function("LightsOff", set_light_off);
}

void loop(){
  if(can_turn_light_on){
    set_time = millis();
    state = HIGH;
    digitalWrite(ledPin, state);
    can_turn_light_on = false;
  }
  current_time = millis();
  if( set_time+FIFTEEN_MIN_MILLIS < current_time ){
    light_off();
    hold_light_off = false;
  }
}

void trigger_light_on(){
  if (!hold_light_off){
    set_light_on("triggered");
  }
}

void light_off(){
  state = LOW;
  digitalWrite(ledPin, state);
}



int set_light_on(String args){
    if(args != "triggered"){
        hold_light_off = false;
    }
  can_turn_light_on = true;
  return 1;
}

int set_light_off(String args){
  hold_light_off = true;
  set_time = millis() + THREE_HOURS_MILLIS;
  light_off();
  return 1;
}
{% endhighlight %} 



### <a name="do_func">IFTTT Do Functions</a>
--------------------------

<br>

I am using the [Do app for iOS](https://ifttt.com/products) which provides access to spark functions. One of the keys to having access to your functions is getting them to compile correctly. To do this I had to make sure that my functions returned a value and that they took some string arguments. I am not really sure why this is required and would love some feedback on that, but I digress. below is my code striped to a single function and only the important parts. First, we have are declarations which must have a return type and take arguments. Next, in setup we must declare our function for use by the Spark API using `Spark.function(call_name, function);`. In this case function is whatever the name of the function you want is call is, and call_name is what you will show external systems. Sample code is shown below

{% highlight C %}
// functions declarations
void light_off(void);
int set_light_off(String args);

// variable declarations
int ledPin = D6;
volatile int state = HIGH;

void setup(){
  pinMode(D0, INPUT_PULLDOWN );
  pinMode(ledPin, OUTPUT);
  Spark.function("LightsOff", set_light_off);
}

void loop(){
  light_off();
}

void light_off(){
  state = LOW;
  digitalWrite(ledPin, state);
}

int set_light_off(String args){
  light_off();
  return 1;
}
{% endhighlight %} 

To test your function from the command line you can install the [spark-cli](https://github.com/spark/spark-cli). And use the following call assuming your spark core is called `sample_core`

`spark call sample_core LightsOff`

the key here is to use the function call name you defined as the first argument for `Spark.function` in `setup`.

Now to make things even more interesting you can use the [Do app from IFTTT](https://ifttt.com/products/do/button). The setup here is insanely easy and very useful.

1. Inside the App, Configure IFTTT to use the spark channel
2. Select Spark under Channels
3. Select Create a New Recipe
4. Select Call a Function
5. Select your core and the proper function in this case `LightsOff` on "sample_core"
6. provide an unused input.

now you should be able to turn the light off from your phone or anywhere you can trigger the ifttt event!


### <a name="conclusion">Conclusion</a>
--------------------------

<br>

Nice little update with some cleaner, more readable code, and added functionaility. Really nice to have the ability to turn my lights out from my phone before starting a movie and have them stay off.  

###Congratulations you can now set your light on or off from your phone. 🍻 

