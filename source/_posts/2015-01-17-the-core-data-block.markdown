---
layout: post
title:  "Core Data的多线程"
date:   2015-01-17 23:58:24
categories: iOS CoreData
---
自己写的程序一直都没什么技术含量，很长一段时间都一直在用`NSUserDefaults`来做数据持久化。一些小型商业项目也几乎不存什么数据，更多的是展示逻辑和一些简单的数据存储。

最近有机会去处理大量数据，就开开心心的带着bug上路了。

之前的经验一直是在主线程处理所有的逻辑，所以也一直没有遇到什么太复杂的问题，而且UI上都是直接使用`NSManagedObject`的子类，现在回头看看也是勇敢。

当时还遇到一个问题，我还不大明白，CoreData取出来的数据有时候所有的字段都是nil，对象都是fault的，不知为何，这个会在后面的篇中来说说CoreData的Faulting机制，

苹果并没有 "不推荐" 我们在主线程中使用CoreData，它的默认实现（在建立工程的时候选择使用CoreData就可以看到，苹果在AppDelegate中为我们实现了一套必备代码）也是在主线程中工作的，理所当然的我也一直这么用，直到有一天发现数据多了会阻塞主线程导致卡顿，这时我才想到要分线程来做。

分线程很简单，都是用NSOperationQueue或者GCD，分分钟搞定的问题，虽然看起来也不错，不过偶尔会出现一些无法解释的小问题。

但事实并非如此。后来读了苹果的文档之后才知道CoreData的对象原来完全不能在线程间传递。而应该为每个单独线程指定一个`NSManagedObjectContext`，但是并不是说自己new一个NSOperationQueue当context在里面操作就好了。

在看了一些文章之后，暂时我理解到的正确的做法应该是这样的：

* 建立不同的context
* 使用context的`performBlock:`等方法来执行任务，而不是自己维护的queue
* 在不同的context间传递对象应该使用`objectID`然后使用`objectWithID:`或`existingObjectWithID:error:`来在接收context中获取对象
* 不要或者避免给UI直接使用`NSManagedObject`而是使用自己的`NSObject`来传递这样你就可以避免一些空壳数据的情况（往往是UI不知道你删除了CoreData中的数据导致的）
* 还要注意处理好不同context数据merge的问题

这里引用[objc.io的一篇文章](http://objccn.io/issue-10-5/)的一部分来说明人家的`最佳实现`

使用下面的代码设置一个 managed object context：

```objectivec
- (NSManagedObjectContext *)setupManagedObjectContextWithConcurrencyType:(NSManagedObjectContextConcurrencyType)concurrencyType
  {
    NSManagedObjectContext *managedObjectContext = [[NSManagedObjectContext alloc] initWithConcurrencyType:concurrencyType];
    managedObjectContext.persistentStoreCoordinator =
            [[NSPersistentStoreCoordinator alloc] initWithManagedObjectModel:self.managedObjectModel];
    NSError* error;
    [managedObjectContext.persistentStoreCoordinator addPersistentStoreWithType:NSSQLiteStoreType 
                                                                  configuration:nil 
                                                                            URL:self.storeURL 
                                                                        options:nil 
                                                                          error:&error];
    if (error) {
        NSLog(@"error: %@", error.localizedDescription);
    }
    return managedObjectContext;
  }
  
```

然后我们调用这个方法两次，一次是为主 managed object context，一次是为后台 managed object context：

```objectivec
self.managedObjectContext = [self setupManagedObjectContextWithConcurrencyType:NSMainQueueConcurrencyType];
self.backgroundManagedObjectContext = [self setupManagedObjectContextWithConcurrencyType:NSPrivateQueueConcurrencyType];
```

注意传递的参数 NSPrivateQueueConcurrencyType 告诉 Core Data 创建一个独立队列，这将确保后台 managed object context 的运行发生在一个独立的线程中。

现在就剩一步了：每当后台 context 保存后，我们需要更新主线程。我们在[之前第 2 期的这篇文章](http://objccn.io/issue-2-2/)中描述了如何操作。我们注册一下，当 context 保存时得到一个通知，如果是后台 context，调用 mergeChangesFromContextDidSaveNotification: 方法。这就是我们要做的所有事情：
```objectivec
[[NSNotificationCenter defaultCenter] addObserverForName:NSManagedObjectContextDidSaveNotification
                    object:nil
                     queue:nil
                usingBlock:^(NSNotification* note) {
    NSManagedObjectContext *moc = self.managedObjectContext;
    if (note.object != moc) {
        [moc performBlock:^(){
            [moc mergeChangesFromContextDidSaveNotification:note];
        }];
    }
 }];
```
这儿还有一个小忠告：mergeChangesFromContextDidSaveNotification: 是在 performBlock:中发生的。在我们这个情况下，moc 是主 managed object context，因此，这将会阻塞主线程。

注意你的 UI（即使是只读的）必须有能力处理对象的改变，或者事件的删除。Brent Simmons 最近写了两篇文章，分别是 [《Why Use a Custom Notification for Note Deletion》](http://inessential.com/2014/02/25/why_use_a_custom_notification_for_note_d) 和 [《Deleting Objects in Core Data》](http://inessential.com/2014/02/25/more_about_deleting_objects_in_core_data)。这些文章解释说明了如何面对这些情况，如果你在你的 UI 中显示一个对象，这个对象有可能会发生改变或者被删除。