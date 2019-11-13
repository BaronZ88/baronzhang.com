---
title: 观察者模式（ObserverPattern）
date: 2017-02-06 00:16:14
categories: DesignPatterns
tags:
- Design Pattern
- Java
---

> 首发于公众号：**BaronTalk**，文章中的例子和思路均来自于《Head First》

## 场景

我们接到一个来自气象局的需求：气象局需要我们构建一套系统，这系统有两个公告牌，分别用于显示当前的实时天气和未来几天的天气预报。当气象局发布新的天气数据（WeatherData）后，两个公告牌上显示的天气数据必须实时更新。气象局同时要求我们保证程序拥有足够的可扩展性，因为后期随时可能要新增新的公告牌。

## 概况

这套系统中主要包括三个部分：气象站（获取天气数据的物理设备）、WeatherData（追踪来自气象站的数据，并更新公告牌）、公告牌（用于展示天气数据）

![WeatherStation](http://resources.baronzhang.com/DesignPatterns/ObserverPattern/WeatherStation.png)

WeatherData知道如何跟气象站联系，以获得天气数据。当天气数据有更新时，WeatherData会更新两个公告牌用于展示新的天气数据。

<!--more-->

## 错误示范
我们现来看看隔壁老王的实现思路：

```java
public class WeatherData {

    //实例变量声明
    ...
    
    public void measurementsChanged() {
    
        float temperature = getTemperature();
        float humidity = getHumidity();
        float pressure = getPressure();
        List<Float> forecastTemperatures = getForecastTemperatures();
        
        //更新公告牌
        currentConditionsDisplay.update(temperature, humidity, pressure);
        forecastDisplay.update(forecastTemperatures);
    }
    ...
}
```

上面这段代码是典型的针对实现编程，这会导致我们以后增加或删除公告牌时必须修改程序。我们现在来看看观察者模式，然后再回来看看如何将观察者模式应用到这个程序。

## 观察者模式介绍
观察者模式面向的需求是：A对象（观察者）对B对象（被观察者）的某种变化高度敏感，需要在B变化的一瞬间做出反应。举个例子，新闻里喜闻乐见的警察抓小偷，警察需要在小偷伸手作案的时候实施抓捕。在这个例子里，警察是观察者、小偷是被观察者，警察需要时刻盯着小偷的一举一动，才能保证不会错过任何瞬间。程序里的观察者和这种真正的【观察】略有不同，观察者不需要时刻盯着被观察者（例如A不需要每隔1ms就检查一次B的状态），二是采用__注册__(_Register_)或者成为__订阅__(_Subscribe_)的方式告诉被观察者：我需要你的某某状态，你要在它变化时通知我。采取这样被动的观察方式，既省去了反复检索状态的资源消耗，也能够得到最高的反馈速度。

观察者模式通常基于__Subject__和__Observer__接口类来设计，下面是是类图：
![Observer](http://resources.baronzhang.com/DesignPatterns/ObserverPattern/Observer.png)

## 观察者模式的应用
结合上面的类图，我们现在将观察者模式应用到WeatherData项目中来。于是有了下面这张类图：
![ObserverForWeatherStation](http://resources.baronzhang.com/DesignPatterns/ObserverPattern/ObserverForWeatherStation.png)

__主题接口__

```java
/**
 * 主题（发布者、被观察者）
 */
public interface Subject {

    /**
     * 注册观察者
     */
    void registerObserver(Observer observer);

    /**
     * 移除观察者
     */
    void removeObserver(Observer observer);

    /**
     * 通知观察者
     */
    void notifyObservers(); 
}
```

__观察者接口__

```java

/**
 * 观察者
 */
public interface Observer {
    void update();
}
```


__公告牌用于显示的公共接口__

```java
public interface DisplayElement {
    void display();
}
```

__下面我们再来看看WeatherData是如何实现的__

```java
public class WeatherData implements Subject {

    private List<Observer> observers;

    private float temperature;//温度
    private float humidity;//湿度
    private float pressure;//气压
    private List<Float> forecastTemperatures;//未来几天的温度

    public WeatherData() {
        this.observers = new ArrayList<Observer>();
    }

    @Override
    public void registerObserver(Observer observer) {
        this.observers.add(observer);
    }

    @Override
    public void removeObserver(Observer observer) {
        this.observers.remove(observer);
    }

    @Override
    public void notifyObservers() {
        for (Observer observer : observers) {
            observer.update();
        }
    }

    public void measurementsChanged() {
        notifyObservers();
    }

    public void setMeasurements(float temperature, float humidity, 
    float pressure, List<Float> forecastTemperatures) {
        this.temperature = temperature;
        this.humidity = humidity;
        this.pressure = pressure;
        this.forecastTemperatures = forecastTemperatures;
        measurementsChanged();
    }

    public float getTemperature() {
        return temperature;
    }

    public float getHumidity() {
        return humidity;
    }

    public float getPressure() {
        return pressure;
    }

    public List<Float> getForecastTemperatures() {
        return forecastTemperatures;
    }
}
```

__显示当前天气的公告牌CurrentConditionsDisplay__

```java
public class CurrentConditionsDisplay implements Observer, DisplayElement {

    private WeatherData weatherData;

    private float temperature;//温度
    private float humidity;//湿度
    private float pressure;//气压

    public CurrentConditionsDisplay(WeatherData weatherData) {
        this.weatherData = weatherData;
        this.weatherData.registerObserver(this);
    }

    @Override
    public void display() {
        System.out.println("当前温度为：" + this.temperature + "℃");
        System.out.println("当前湿度为：" + this.humidity);
        System.out.println("当前气压为：" + this.pressure);
    }

    @Override
    public void update() {
        this.temperature = this.weatherData.getTemperature();
        this.humidity = this.weatherData.getHumidity();
        this.pressure = this.weatherData.getPressure();
        display();
    }
}
```

__显示未来几天天气的公告牌ForecastDisplay__

```java
public class ForecastDisplay implements Observer, DisplayElement {

    private WeatherData weatherData;

    private List<Float> forecastTemperatures;//未来几天的温度

    public ForecastDisplay(WeatherData weatherData) {
        this.weatherData = weatherData;
        this.weatherData.registerObserver(this);
    }

    @Override
    public void display() {
        System.out.println("未来几天的气温");
        int count = forecastTemperatures.size();
        for (int i = 0; i < count; i++) {
            System.out.println("第" + i + "天:" + forecastTemperatures.get(i) + "℃");
        }
    }

    @Override
    public void update() {
        this.forecastTemperatures = this.weatherData.getForecastTemperatures();
        display();
    }
}
```

到这里，我们整个气象局的WeatherData应用就改造完成了。两个公告牌`CurrentConditionsDisplay`和`ForecastDisplay`实现了`Observer`和`DisplayElement`接口，在他们的构造方法中会调用`WeatherData`的`registerObserver`方法将自己注册成观察者，这样被观察者`WeatherData`就会持有观察者的应用，并将它们保存到一个集合中。当被观察者``WeatherData`状态发送变化时就会遍历这个集合，循环调用观察者`公告牌`更新数据的方法。后面如果我们需要增加或者删除公告牌就只需要新增或者删除实现了`Observer`和`DisplayElement`接口的公告牌就好了。

观察者模式将观察者和主题（被观察者）彻底解耦，主题只知道观察者实现了某一接口（也就是Observer接口）。并不需要观察者的具体类是谁、做了些什么或者其他任何细节。任何时候我们都可以增加新的观察者。因为主题唯一依赖的东西是一个实现了`Observer`接口的对象列表。

好了，我们测试下利用观察者模式重构后的程序：

```java
public class ObserverPatternTest {

    public static void main(String[] args) {

        WeatherData weatherData = new WeatherData();
        CurrentConditionsDisplay currentConditionsDisplay = new CurrentConditionsDisplay(weatherData);
        ForecastDisplay forecastDisplay = new ForecastDisplay(weatherData);

        List<Float> forecastTemperatures = new ArrayList<Float>();
        forecastTemperatures.add(22f);
        forecastTemperatures.add(-1f);
        forecastTemperatures.add(9f);
        forecastTemperatures.add(23f);
        forecastTemperatures.add(27f);
        forecastTemperatures.add(30f);
        forecastTemperatures.add(10f);
        weatherData.setMeasurements(22f, 0.8f, 1.2f, forecastTemperatures);
    }
}
```

输出结果：

	当前温度为：22.0℃
	当前湿度为：0.8
	当前气压为：1.2
	未来几天的气温
	第0天:22.0℃
	第1天:-1.0℃
	第2天:9.0℃
	第3天:23.0℃
	第4天:27.0℃
	第5天:30.0℃
	第6天:10.0℃

> 源码地址：[https://github.com/BaronZ88/DesignPatterns/tree/master/src/com/baron/patterns/observer](https://github.com/BaronZ88/DesignPatterns/tree/master/src/com/baron/patterns/observer)

------

> 如果你喜欢我的文章，就关注下我的公众号 **BaronTalk** 、 [**知乎专栏**](https://zhuanlan.zhihu.com/baron) 或者在 [**GitHub**](https://github.com/BaronZ88) 上添个 Star 吧！
>
> - 微信公众号：**BaronTalk**
> - 知乎专栏：[https://zhuanlan.zhihu.com/baron](https://zhuanlan.zhihu.com/baron)  
> - GitHub：[https://github.com/BaronZ88](https://github.com/BaronZ88)

<div align="center"><img src="https://resources.baronzhang.com/blog/common/gzh3.png" width="85%"/></div>