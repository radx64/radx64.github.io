---
layout: post
title: "Using std::chrono as a nice way to delay things"
categories: [cpp, general]
tags: [chrono, gameengine, literals]
---

### It always starts easy

In my free time, I tinker with my own game engine-because, honestly, who doesn't tried to create some kind of a game in the past? At some point, while messing around with the usual GUI stuff, I needed a good old-fashioned blinking text cursor. Simple enough, right? So, I rolled up my sleeves and built a `TimerService` to handle it. This bad boy keeps track of multiple `Timer` instances, making sure they fire at the right moments. Since it's hooked into the game engine's main loop, it always knows exactly how much time has passed and when to trigger callbacks. Nothing fancy-just a clean, efficient way to keep things ticking!

```cpp
class Timer
{
public:
    using Notification = std::function<void(void)>;
    Timer(const float delay, Notification notification);

protected:
    Notification notification_;
    float delay_;
};

// this is an excerpt of a whole timer class 

enum class TimerType
{
    OneShot,
    Repeating
};

class TimerService final
{
public:
    TimerService();
    // some code was purposely taken out
    void update(const float delta);

    void start(Timer* timer, const TimerType type);    
    void restart(Timer* timer, const TimerType type);
    void cancel(Timer* timer);

private:
    float currentTime_;
};
```

A big part of the TimerService is keeping track of time. If a timer isn't ready to fire yet, it simply gets ignored-no need to waste cycles on it. But when a repeating timer does go off, we just slap on a new delay and let it keep ticking. Simple, efficient, and keeps everything running like clockwork! 

```cpp
...
         // float       // float
    if ((currentTime_ < timerInstance.nextTick) || not timerInstance.active)
    {
        continue;
    }
    // float                  // float
    timerInstance.nextTick += timerInstance.timer->getDelay();
...
```

### Is it so?

You create a timer, give it a callback, set a delay (or a repeat interval), and hand it over to the `TimerService`. From there, it takes care of the rest-calling your function exactly when it should.  

Sounds simple, right? And hey, it works! But let's not just take my word for it-here's how it looks in some pseudo-code:

```cpp
auto myAwesomeTimer = Timer(32.f, [](){std::cout << "It works!" << std::endl;});
timerService.start(myAwesomeTimer, TimerType::Repeating);
```

Pretty cool, right? Right? We just created a repeating timer that can fire every 32 seconds… wait, I mean 32 milliseconds… or hey, why not 32 hours? You get the idea.

The problem? Using a raw `float` for time intervals is just asking for trouble. Maybe it's time (pun intended) to introduce some proper types for conversion. I remember watching [a CppCon talk](https://www.youtube.com/watch?v=pPSdmrmMdjY) by Mateusz Pusz on the `mp-units` library-absolute beast when it comes to unit conversions.

But let's be real-I'm way too lazy to build something like that from scratch just for time handling. Lucky for us, some smart folks already did the heavy lifting in the STL with [`std::chrono`](https://en.cppreference.com/w/cpp/chrono).

### std::chrono for the rescue

`std::chrono` implements few important parts:
- clocks
- timepooints
- durations
- calendar dates and time zone information (C++20)

Let's take chrono durations and crank them up a notch to fit our needs.

`std::chrono::duration` is defined as follows

```cpp
template<
    class Rep,
    class Period = std::ratio<1>
> class duration;
```
It is a template class with two template arguments `Rep` representing underlying type of clock tick, and `Period` which is a number of "second's fractions per tick", i like rather call it "number of ticks in one second"

So we can define our `Duration` as:

```cpp
    using rep = uint32_t;                   // timer ticks are counted as uint32_t
    using period = std::ratio<1, 1000>;     // 1 tick takes 1 milisecond (1000 ticks in one second)
    using duration = std::chrono::duration<rep, period>; // our chrono duration
```

Since we've got a duration, why not take it a step further and create a `std::chrono::time_point`? Perfect for time travel-just kidding… more like time computations.
Our `TimerService` needs to keep track of time, and `std::chrono::time_point` is just the tool for the job. But there's a catch-it needs a "clock" class as a template argument. 

```cpp
template<
    class Clock,
    class Duration = typename Clock::duration
> class time_point;
```
so we need to bundle everything together in nice `Clock` class:
```cpp

#include <chrono>
#include <cstdint>

struct Clock
{
    using rep = uint32_t;                   // timer ticks are counter as uint32_t
    using period = std::ratio<1, 1000>;     // 1000 ticks in 1 second
    using duration = std::chrono::duration<rep, period>; // our chrono duration
    using time_point = std::chrono::time_point<Clock>;
};

```
Now we've got ourselves a pretty solid clock-perfect for all our time-related shenanigans.
Time to ditch that raw `float` and upgrade our `Timer` to use `Clock::duration` instead. And while we're at it, let's do the same for `TimerService`. Cleaner, safer, and just all-around better!
```cpp

class Timer
{
public:
    using Notification = std::function<void(void)>;
    Timer(const Clock::duration& delay, Notification notification);   // float was here

protected:
    Notification notification_;
    Clock::duration delay_;     // float was here
};

// this is an excerpt of a whole timer class 

class TimerService final
{
public:
    TimerService();
    // some code was purposely taken out
    void update(const Clock::duration& delta);  // float was here

    void start(Timer* timer, const TimerType type);    
    void restart(Timer* timer, const TimerType type);
    void cancel(Timer* timer);

private:
    Clock::time_point currentTime_;   // float was here, replaced by time_point (does what name suggest)
};
```

The difference is mostly cosmetic. Even integrating the new types into `TimerService` is smooth as butter.
```cpp
...
         // time_point  // time_point
    if ((currentTime_ < timerInstance.nextTick) || not timerInstance.active)
    {
        continue;
    }
    // time_point             // duration
    timerInstance.nextTick += timerInstance.timer->getDelay();
...
```

### Grand finale

Now, let's shift our focus to the “client” side of these changes. We can define our duration as
```cpp
auto duration = std::chrono::seconds(30);
```
But hey, why stop there? Since C++11, we've had user-defined literals-lucky for us, `std::chrono` makes great use of them. So, what does this mean for our timer? Let's find out!
We can level up code by using `std::literals` with those sweet time suffixes.

```cpp

using namespace std::literals; // literal suffixes are defined in std::literals namespace

auto myAwesomeMsTimer = Timer(32ms, [](){std::cout << "It works!" << std::endl;});
timerService.start(myAwesomeMsTimer, TimerType::Repeating);

auto myAwesomeSTimer = Timer(32s, [](){std::cout << "It works!" << std::endl;});
timerService.start(myAwesomeSTimer, TimerType::Repeating);

auto myAwesomeMinTimer = Timer(32min, [](){std::cout << "It works!" << std::endl;});
timerService.start(myAwesomeMinTimer, TimerType::Repeating);

auto myAwesomeHTimer = Timer(32h, [](){std::cout << "It works!" << std::endl;});
timerService.start(myAwesomeHTimer, TimerType::Repeating);
```

Now that we're using literals, it's crystal clear what type of duration we're dealing with. It's clean, it's descriptive, and the conversions between different time ranges? They just work, right out of the box. And the best part? It takes barely any changes. I'm all for it-because hey, I'm lazy. And let's be honest, everyone deserves a little lazy time. 


### It wasn't that bad, was it?

In this blog post we enhanced the basic timer functionality with `std::chrono` to handle durations and time points more effectively. We swapped out raw `float` values for Clock::duration, making the code cleaner and safer. By leveraging C++11 user-defined literals, we further improved the clarity and expressiveness of the code, ensuring that duration types are explicit and conversions happen effortlessly. With just a few simple changes, we made the timer system more powerful and readable-because, hey, sometimes being a little lazy leads to cleaner, more efficient solutions!
