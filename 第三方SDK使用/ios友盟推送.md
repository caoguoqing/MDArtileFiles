#序言

这里是很久以前封装的一个小工具类，现在搬到本博客里。这里没有详细教你如何插入友盟推送的教程，只是直接将原来在某小公司里所做的项目需要到集成时，写了一个demo，这里公开给大家而已。

#注册

友盟推送官网：[http://www.umeng.com/push](http://www.umeng.com/push)

#注意事项

友盟推送中，有一个问题，那就是应用处于前台时接收到的推送消息如何显示的问题。
友盟提供了默认的显示框，但是样式不是我们想要的，因此友盟也提供了用户自定义显示框的功能，
但是在用户点击后，友盟要求调用指定的API向友盟反馈。

```
// 如果不调用此方法，统计数据会拿不到，但是如果调用此方法，会再弹一次友盟定制的alertview显示推送消息
// 所以这里根据需要来处理是否屏掉此功能
[UMessage sendClickReportForRemoteNotification:[HYBUMessageHelper shared].userInfo];
```

#如何封装

下面是我所封装的友盟推送工具类：

```
//
//  HYBUMessageHelper.h
//  UMessageDemo
//
//  Created by 黄仪标 on 14/11/20.
//  Copyright (c) 2014年 黄仪标. All rights reserved.
//

#import <Foundation/Foundation.h>
#import <UIKit/UIKit.h>

/*!
 * @brief 友盟消息推送API相关封装类
 * @author huangyibiao
 */
@interface HYBUMessageHelper : NSObject <UIAlertViewDelegate>

/// 在应用启动时调用此方法注册
+ (void)startWithLaunchOptions:(NSDictionary *)launchOptions;

+ (void)registerDeviceToken:(NSData *)deviceToken;
+ (void)didReceiveRemoteNotification:(NSDictionary *)userInfo;
// 关闭接收消息通知
+ (void)unregisterRemoteNotifications;

// default is YES
// 使用友盟提供的默认提示框显示推送信息
+ (void)setAutoAlertView:(BOOL)shouldShow;

// 应用在前台时，使用自定义的alertview弹出框显示信息
+ (void)showCustomAlertViewWithUserInfo:(NSDictionary *)userInfo;

@end

//
//  HYBUMessageHelper.m
//  UMessageDemo
//
//  Created by 黄仪标 on 14/11/20.
//  Copyright (c) 2014年 黄仪标. All rights reserved.
//

#import "HYBUMessageHelper.h"
#import "UMessage.h"
#include <objc/runtime.h>

#define kUMessageAppKey @"546d9a53fd98c533600016bb"

// ios 8.0 以后可用，这个参数要求指定为固定值
#define kCategoryIdentifier @"xiaoyaor"

@interface HYBUMessageHelper ()

@property (nonatomic, strong) NSDictionary *userInfo;

@end

@implementation HYBUMessageHelper

+ (HYBUMessageHelper *)shared {
  static HYBUMessageHelper *sharedObject = nil;
  
  static dispatch_once_t onceToken;
  dispatch_once(&onceToken, ^{
    if (!sharedObject) {
      sharedObject = [[[self class] alloc] init];
    }
  });
  
  return sharedObject;
}

+ (void)startWithLaunchOptions:(NSDictionary *)launchOptions {
  // set AppKey and LaunchOptions
  [UMessage startWithAppkey:kUMessageAppKey launchOptions:launchOptions];
  
#if __IPHONE_OS_VERSION_MAX_ALLOWED >= __IPHONE_8_0
  if([[[UIDevice currentDevice] systemVersion] floatValue] >= 8.0) {
    // register remoteNotification types
    UIMutableUserNotificationAction *action1 = [[UIMutableUserNotificationAction alloc] init];
    action1.identifier = @"action1_identifier";
    action1.title=@"Accept";
    action1.activationMode = UIUserNotificationActivationModeForeground;// 当点击的时候启动程序
    
    UIMutableUserNotificationAction *action2 = [[UIMutableUserNotificationAction alloc] init];  // 第二按钮
    action2.identifier = @"action2_identifier";
    action2.title = @"Reject";
    action2.activationMode = UIUserNotificationActivationModeBackground;// 当点击的时候不启动程序，在后台处理
    // 需要解锁才能处理，如果action.activationMode = UIUserNotificationActivationModeForeground;则这个属性被忽略；
    action2.authenticationRequired = YES;
    action2.destructive = YES;
    
    UIMutableUserNotificationCategory *categorys = [[UIMutableUserNotificationCategory alloc] init];
    categorys.identifier = kCategoryIdentifier;// 这组动作的唯一标示
    [categorys setActions:@[action1,action2] forContext:(UIUserNotificationActionContextDefault)];
    
    UIUserNotificationType types = UIUserNotificationTypeBadge
    | UIUserNotificationTypeSound
    | UIUserNotificationTypeAlert;
    UIUserNotificationSettings *userSettings = [UIUserNotificationSettings settingsForTypes:types
                                                                                 categories:[NSSet setWithObject:categorys]];
    
    [UMessage registerRemoteNotificationAndUserNotificationSettings:userSettings];
  } else {
    // register remoteNotification types
    UIRemoteNotificationType types = UIRemoteNotificationTypeBadge
    | UIRemoteNotificationTypeSound
    | UIRemoteNotificationTypeAlert;
    
    [UMessage registerForRemoteNotificationTypes:types];
  }
#else
  // iOS8.0之前使用此注册
  // register remoteNotification types
  UIRemoteNotificationType types = UIRemoteNotificationTypeBadge
  | UIRemoteNotificationTypeSound
  | UIRemoteNotificationTypeAlert;
  
  [UMessage registerForRemoteNotificationTypes:types];
#endif
  
#if DEBUG
  [UMessage setLogEnabled:YES];
#else
  [UMessage setLogEnabled:NO];
#endif
}

+ (void)registerDeviceToken:(NSData *)deviceToken {
  [UMessage registerDeviceToken:deviceToken];
  return;
}

+ (void)unregisterRemoteNotifications {
  [UMessage unregisterForRemoteNotifications];
  return;
}

+ (void)didReceiveRemoteNotification:(NSDictionary *)userInfo {
  [UMessage didReceiveRemoteNotification:userInfo];
  return;
}

+ (void)setAutoAlertView:(BOOL)shouldShow {
  [UMessage setAutoAlert:shouldShow];
  return;
}

+ (void)showCustomAlertViewWithUserInfo:(NSDictionary *)userInfo {
  [HYBUMessageHelper shared].userInfo = userInfo;
  
  // 应用当前处于前台时，需要手动处理
  if ([UIApplication sharedApplication].applicationState == UIApplicationStateActive) {
    dispatch_async(dispatch_get_main_queue(), ^{
      [UMessage setAutoAlert:NO];
      UIAlertView *alertView = [[UIAlertView alloc] initWithTitle:@"推送消息"
                                                          message:userInfo[@"aps"][@"alert"]
                                                         delegate:[HYBUMessageHelper shared]
                                                cancelButtonTitle:@"取消"
                                                otherButtonTitles:@"确定", nil];
      [alertView show];
    });
  }
  return;
}

#pragma mark - UIAlertViewDelegate
- (void)alertView:(UIAlertView *)alertView clickedButtonAtIndex:(NSInteger)buttonIndex {
  if (buttonIndex == 1) {
    // 如果不调用此方法，统计数据会拿不到，但是如果调用此方法，会再弹一次友盟定制的alertview显示推送消息
    // 所以这里根据需要来处理是否屏掉此功能
    [UMessage sendClickReportForRemoteNotification:[HYBUMessageHelper shared].userInfo];
  }
  return;
}

@end
```

#测试使用

下面是测试代码：

```
//
//  AppDelegate.m
//  UMessageDemo
//
//  Created by 黄仪标 on 14/11/20.
//  Copyright (c) 2014年 黄仪标. All rights reserved.
//

#import "AppDelegate.h"
#import "HYBUMessageHelper.h"
#import "UMessage_Sdk_1.1.0/UMessage.h"

@interface AppDelegate ()

@end

@implementation AppDelegate


- (BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions {
  self.window = [[UIWindow alloc] initWithFrame:[[UIScreen mainScreen] bounds]];
  // Override point for customization after application launch.
  
  [HYBUMessageHelper startWithLaunchOptions:launchOptions];
  
  self.window.backgroundColor = [UIColor whiteColor];
  [self.window makeKeyAndVisible];
  return YES;
}

- (void)application:(UIApplication *)application didRegisterForRemoteNotificationsWithDeviceToken:(NSData *)deviceToken {
  [HYBUMessageHelper registerDeviceToken:deviceToken];
  
  NSLog(@"%@",[[[[deviceToken description] stringByReplacingOccurrencesOfString: @"<" withString: @""]
                stringByReplacingOccurrencesOfString: @">" withString: @""]
               stringByReplacingOccurrencesOfString: @" " withString: @""]);
  return;
}

- (void)application:(UIApplication *)application didReceiveRemoteNotification:(NSDictionary *)userInfo {
  [HYBUMessageHelper didReceiveRemoteNotification:userInfo];
  
  [HYBUMessageHelper setAutoAlertView:NO];
  return;
}

- (void)application:(UIApplication *)application didReceiveRemoteNotification
                   :(NSDictionary *)userInfo fetchCompletionHandler
                   :(void (^)(UIBackgroundFetchResult))completionHandler {
  [HYBUMessageHelper didReceiveRemoteNotification:userInfo];
  
  [HYBUMessageHelper setAutoAlertView:NO];
  completionHandler(UIBackgroundFetchResultNewData);
  return;
}

@end
```

#最后

接下来就需要去官网发通知测试一下，在发通知之前，需要先注册设备，否则在开发环境下，是不会有设备收到信息的。

#源代码

之前一直给了错误的链接，原来给的是友盟统一的demo的链接，这里已经更正过来了！

请到`github`下载：[CoderJackyHuang：UMessageDemo_Push](https://github.com/CoderJackyHuang/UMessageDemo_Push)

