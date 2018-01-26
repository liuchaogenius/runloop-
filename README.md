# runloop-

RunLoop 顺序
                 1、进入
                 
                 2、通知Timer
                 3、通知Source
                 4、处理Source
                 5、如果有 Source1 调转到 11
                 6、通知 BeforWaiting
                 7、wait
                 8、通知afterWaiting
                 9、处理timer
                 10、处理 dispatch 到 main_queue 的 block
                 11、处理 Source1、
                 12、进入 2
                 
                 13、退出

//
//  PerformanceMonitor.m
//  SuperApp
//
//  Created by tanhao on 15/11/12.
//  Copyright © 2015年 Tencent. All rights reserved.
//

#import "PerformanceMonitor.h"
#import <CrashReporter/CrashReporter.h>

@interface PerformanceMonitor ()
{
    int timeoutCount;
    CFRunLoopObserverRef observer;
    
    @public
    dispatch_semaphore_t semaphore;
    CFRunLoopActivity activity;
}
@end

@implementation PerformanceMonitor

+ (instancetype)sharedInstance
{
    static id instance = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        instance = [[self alloc] init];
    });
    return instance;
}

static void runLoopObserverCallBack(CFRunLoopObserverRef observer, CFRunLoopActivity activity, void *info)
{
    PerformanceMonitor *moniotr = (__bridge PerformanceMonitor*)info;
    
    moniotr->activity = activity;
    
    dispatch_semaphore_t semaphore = moniotr->semaphore;
    dispatch_semaphore_signal(semaphore);
}

- (void)stop
{
    if (!observer)
        return;
    
    CFRunLoopRemoveObserver(CFRunLoopGetMain(), observer, kCFRunLoopCommonModes);
    CFRelease(observer);
    observer = NULL;
}

//NSDictionary *keyDict = @{
//                          @1 : @"kCFRunLoopEntry",
//                          @2 : @"kCFRunLoopBeforeTimers",
//                          @4 : @"kCFRunLoopBeforeSources",
//                          @32 : @"kCFRunLoopBeforeWaiting",
//                          @64 : @"kCFRunLoopAfterWaiting",
//                          @128 : @"kCFRunLoopExit",
//                          };
//
//CFRunLoopActivity flags =
//kCFRunLoopEntry |
//kCFRunLoopBeforeTimers |
//kCFRunLoopBeforeSources |
//kCFRunLoopBeforeWaiting |
//kCFRunLoopAfterWaiting |
//kCFRunLoopExit |
//kCFRunLoopAllActivities;
//
//CFRunLoopObserverRef runloopObserver = CFRunLoopObserverCreateWithHandler(
//                                                                          kCFAllocatorDefault, flags, YES, 0,
//                                                                          ^(CFRunLoopObserverRef observer, CFRunLoopActivity activity) {
//                                                                              NSLog(@"%@", keyDict[@(activity)]);
//                                                                          });

//CFRunLoopAddObserver(CFRunLoopGetCurrent(), runloopObserver, kCFRunLoopDefaultMode);
- (void)start
{
    if (observer)
        return;
    
    // 信号
    semaphore = dispatch_semaphore_create(0);
    
    // 注册RunLoop状态观察
    CFRunLoopObserverContext context = {0,(__bridge void*)self,NULL,NULL};
    observer = CFRunLoopObserverCreate(kCFAllocatorDefault,
                                       kCFRunLoopAllActivities,
                                       YES,
                                       0,
                                       &runLoopObserverCallBack,
                                       &context);
    CFRunLoopAddObserver(CFRunLoopGetMain(), observer, kCFRunLoopCommonModes);
    
    // 在子线程监控时长
    dispatch_async(dispatch_get_global_queue(0, 0), ^{
        while (YES)
        {
            long st = dispatch_semaphore_wait(semaphore, dispatch_time(DISPATCH_TIME_NOW, 50*NSEC_PER_MSEC));
            if (st != 0)
            {
                if (!observer)
                {
                    timeoutCount = 0;
                    semaphore = 0;
                    activity = 0;
                    return;
                }
                
                if (activity==kCFRunLoopBeforeSources || activity==kCFRunLoopAfterWaiting)
                {
                    if (++timeoutCount < 5)
                        continue;
                    
                    PLCrashReporterConfig *config = [[PLCrashReporterConfig alloc] initWithSignalHandlerType:PLCrashReporterSignalHandlerTypeBSD
                                                                                       symbolicationStrategy:PLCrashReporterSymbolicationStrategyAll];
                    PLCrashReporter *crashReporter = [[PLCrashReporter alloc] initWithConfiguration:config];
                    
                    NSData *data = [crashReporter generateLiveReport];
                    PLCrashReport *reporter = [[PLCrashReport alloc] initWithData:data error:NULL];
                    NSString *report = [PLCrashReportTextFormatter stringValueForCrashReport:reporter
                                                                              withTextFormat:PLCrashReportTextFormatiOS];
                    
                    NSLog(@"------------\n%@\n------------", report);
                }
            }
            timeoutCount = 0;
        }
    });
}

@end
