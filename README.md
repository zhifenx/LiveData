# LiveData

这是一个使用泛型类型的LiveData类,实现了观察者模式,可以用于发布数据更新的通知。

主要功能有:

1. 定义了Observer结构体,封装了事件回调。

2. LiveData类使用泛型定义,可以持有任意类型的数据。

3. value属性通过didSet发布更新通知,调用notityObservers()方法。

4. 提供subscribe方法进行订阅,返回ObserverKey。

5. subscribeWithKey方法可以直接获取ObserverKey。

6. 使用字典存储观察者,Key是ObserverKey。

7. notityObservers遍历观察者字典,调用每个观察者的回调方法。

8. 提供removeKey方法移除指定观察者。

9. 提供removeAll方法移除所有观察者。


这个LiveData实现了基于观察者模式的数据绑定功能,通过订阅LiveData的value变化来获取数据更新。可以用于MVVM架构中ViewModel和View之间的数据绑定。

```
//  LiveData.swift
//
//  Created by zhifenx on 2021/5/20.

import Foundation

public struct Observer<T> {
    
    private let event: (T?) -> Void
    
    init(_ eventHandler: @escaping (T?) -> Void) {
        self.event = eventHandler
    }
    
    func publish(_ value: T?) {
        self.event(value)
    }
}

public struct ObserverKey: Hashable {
    let rawValue: UInt64
}

public class LiveData<T> {
    private var _nextKey = ObserverKey(rawValue: 0)
    private var observers: [ObserverKey: Observer<T>]?
    private var queue = DispatchQueue(label: "com.livedata.serqueue")
    
    public init() {}
    
    /// 通过属性观察器 发布数据更新的消息
    public var value: T? {
        didSet {
            notityObservers()
        }
    }
    
    /// 订阅消息
    /// - Parameters:
    ///   - fire: 订阅时是否立即触发一次通知
    ///   - event: 事件回调
    public func subscribe(fire: Bool, event: @escaping (T?) -> Void) {
        _ = self.subscribeWithKey(fire: fire, event)
    }
    
    /// 订阅消息（返回ObserverKey 用来移除监听者）
    /// - Parameters:
    ///   - fire: 订阅时是否立即触发一次通知
    ///   - event: 事件回调
    /// - Returns: 观察者 key 可以用removeKey(_ key:)来移除特定的观察者
    public func subscribeWithKey(fire: Bool, _ event: @escaping (T?) -> Void) -> ObserverKey {
                queue.sync {
            let observer = Observer(event)
            
            let key = _nextKey
            _nextKey = ObserverKey(rawValue: _nextKey.rawValue + 1)
            
            if self.observers == nil {
                self.observers = [key: observer]
            }else {
                self.observers![key] = observer
            }
            
            if fire {
                self.notityObservers()
            }
            return key
        }
    }

    /// 移除制定key的观察者
    public func removeKey(_ key: ObserverKey) {
        queue.sync {
            self.observers?.removeValue(forKey: key)
        }
    }
    
    /// 移除所有的观察者
    public func removeAll() {
        queue.sync {
            self.observers?.removeAll()
        }
    }
    
    private func notityObservers() {
        if let observers = self.observers {
            for obj in observers {
                obj.value.publish(value)
            }
        }
    }
}

```
具体使用：

```
class DemoViewModel {
    
    let data: LiveData<Int> = LiveData()
    
    /// 网络请求等...
    func netRequest() {
        
        //...
        //拿到data或者data数据改变时通知观察者
        self.data.value = 2
        
    }
    
}

class DemoViewController: UIViewController {
    
    let vm = DemoViewModel()
    
    override func viewDidLoad() {
        super.viewDidLoad()
        
        /// 监听vm对象中的某个数据的改变
        vm.data.subscribe(fire: false) { dataValue in
            print(dataValue)
        }
        
    }
    
}

```
