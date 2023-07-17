# LiveData

LiveData是使用Swift属性观察器+泛型+闭包实现的观察者模式通信工具

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
