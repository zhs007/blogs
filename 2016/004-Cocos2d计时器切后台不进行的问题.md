# Cocos2d计时器切后台不进行的问题

Why
---
Cocos2d的计时器用起来很方便，但当程序切后台时是不走计时器的，一般情况下也没特别大的问题，毕竟大部分时候并不需要切后台还走计时器。

不过现在的项目，有些UI倒计时，还是需要切后台同步时间的，这里的方案可能很多，但我们项目代码量其实已经有些大了，希望在尽可能少改动的情况下实现这个功能。


解决方案
---
我们都知道，cocos2d的计时器其实是一个传入时间差的一个函数，最简单的做法就是在切后台的时候，记录下当前的时间，在切回前台时，算一下时间差，把这个时间差传入计时器函数即可。

当然，这要求计时器函数本来就是正确的写法才行。

这里，我们实现了一个``ScheduleEx``类，在这个类里，我们将所有需要后台计时同步的接口全部加进来，然后在``onPause``和``onResume``时自动处理计时器即可。

C++的实现
---
C++的实现其实最简单，处理``SEL_SCHEDULE``和``Ref``即可。

```
#ifndef __PWORLD_SCHEDULEEX_HPP__
#define __PWORLD_SCHEDULEEX_HPP__

#include "cocos2d.h"
#include "basedef.h"

struct ScheduleExNode{
    cocos2d::SEL_SCHEDULE callback;
    cocos2d::Ref* target;
};

class ScheduleEx{
    typedef std::list<ScheduleExNode> __List;
public:
    static ScheduleEx* getSingleton();
protected:
    ScheduleEx();
    virtual ~ScheduleEx();
public:
    void add(cocos2d::SEL_SCHEDULE callback, cocos2d::Ref* target);
    
    void remove(cocos2d::SEL_SCHEDULE callback, cocos2d::Ref* target);
    
    bool has(cocos2d::SEL_SCHEDULE callback, cocos2d::Ref* target);
    
    void onResume();
    
    void onPause();
    
    int64_t getCurrentTime();
protected:
    __List m_lst;
    int64_t m_timePause;
};

#endif //__PWORLD_SCHEDULEEX_HPP__
```

CPP文件实现如下：

```
#include "scheduleex.hpp"

ScheduleEx* ScheduleEx::getSingleton()
{
    static ScheduleEx s_singleton;
    
    return &s_singleton;
}

ScheduleEx::ScheduleEx()
{
    m_timePause = 0;
}

ScheduleEx::~ScheduleEx()
{
    
}

void ScheduleEx::remove(cocos2d::SEL_SCHEDULE callback, cocos2d::Ref* target)
{
    for (__List::iterator it = m_lst.begin(); it != m_lst.end(); ++it) {
        if (it->callback == callback && it->target == target) {
            it->target->release();
            
            m_lst.erase(it);
            
            return ;
        }
    }
}

bool ScheduleEx::has(cocos2d::SEL_SCHEDULE callback, cocos2d::Ref* target)
{
    for (__List::iterator it = m_lst.begin(); it != m_lst.end(); ++it) {
        if (it->callback == callback && it->target == target) {
            return true;
        }
    }
    
    return false;
}

void ScheduleEx::add(cocos2d::SEL_SCHEDULE callback, cocos2d::Ref* target)
{
    ScheduleExNode n;
    
    n.callback = callback;
    n.target = target;
    
    m_lst.push_back(n);
    
    target->retain();
}

void ScheduleEx::onResume()
{
    m_timePause = getCurrentTime();
}

void ScheduleEx::onPause()
{
    int64_t ct = getCurrentTime();
    if (m_timePause > 0 && ct > m_timePause) {
        float ot = (ct - m_timePause) / 1000.0f;
        
        for (__List::iterator it = m_lst.begin(); it != m_lst.end(); ++it) {
            (it->target->*(it->callback))(ot);
        }
        
        m_timePause = 0;
    }
}

int64_t ScheduleEx::getCurrentTime()
{
    timeval tv;
    gettimeofday(&tv, NULL);
    int64_t time = ((int64_t)tv.tv_sec) * 1000;
    time += tv.tv_usec / 1000;
    return time;
}
```

lua的实现
---
在cocos的lua实现里，``schedule``的实现其实不是直接走的C++的那一套，实现如下：

```
function schedule(node, callback, delay)
    local delay = cc.DelayTime:create(delay)
    local sequence = cc.Sequence:create(delay, cc.CallFunc:create(callback))
    local action = cc.RepeatForever:create(sequence)
    node:runAction(action)
    return action
end
```

同样的，我们也实现了一个``ScheduleEx``。

这里需要注意一下，下面的实现方式和我们现在项目代码风格一致，并没有很好的用面对对象的方式来实现，所以可能不同的项目，写法会有些差别的。

原理其实都很简单，就是维护一个回调函数队列，在``onPause``的时候计时，记住这里最好用毫秒级别的精度，秒级别会有些不够，所以需要socket才行。

在``onResume``里面把时间差算出来，传给回调函数即可。

```
require("socket")  

ScheduleEx = {
	lst = {},
	lasttime = 0
};

function ScheduleEx.has(node, callback)
	for i, v in ipairs(ScheduleEx.lst) do
		if (v.callback == callback and v.node == node) then
			return true
		end
	end

	return false
end

function ScheduleEx.add(node, callback)
	if (not ScheduleEx.has(node, callback)) then
		table.insert(ScheduleEx.lst, {node = node, callback = callback})
	end
end

function ScheduleEx.remove(node, callback)
	for i, v in ipairs(ScheduleEx.lst) do
		if (v.callback == callback and v.node == node) then
			table.remove(ScheduleEx.lst, i)

			return 
		end
	end
end

function ScheduleEx.onResume()
	local ct = socket.gettime()
	if (ScheduleEx.lasttime > 0 and ct > ScheduleEx.lasttime) then
		local ot = (ct - ScheduleEx.lasttime)
		for i, v in ipairs(ScheduleEx.lst) do
			v.callback(ot)
		end
	end
end

function ScheduleEx.onPause()
	ScheduleEx.lasttime = socket.gettime()
end 
```