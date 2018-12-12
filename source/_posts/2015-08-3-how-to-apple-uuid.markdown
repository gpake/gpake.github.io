---
layout: post
title:  "如何使用苹果自带的UUID"
date:   2015-08-3 23:58:24
categories: iOS
---

## TL;DR

其实苹果以 C 函数的方式提供了 UUID 的方法

```objective-c
- (void)testUUID
{
    uuid_t u;
    uuid_t ur;
    uuid_t ut;
    
    uuid_generate(u);
    uuid_generate_random(ur);
    uuid_generate_time(ut);
    
    char a[16];
    uuid_unparse(u, a);
    char b[16];
    uuid_unparse(ur, b);
    char c[16];
    uuid_unparse(ut, c);
    
//    printf("%s\n", a);
//    printf("%s\n", b);
    printf("%s\n", c);
}

- (NSString *)uuid{
    // Create universally unique identifier (object)
    CFUUIDRef uuidObject = CFUUIDCreate(kCFAllocatorDefault);
    
    // Get the string representation of CFUUID object.
    NSString *uuidStr = (NSString *)CFBridgingRelease(CFUUIDCreateString(kCFAllocatorDefault, uuidObject));
    
    // If needed, here is how to get a representation in bytes, returned as a structure
    // typedef struct {
    //   UInt8 byte0;
    //   UInt8 byte1;
    //   ...
    //   UInt8 byte15;
    // } CFUUIDBytes;
    //    CFUUIDBytes bytes = CFUUIDGetUUIDBytes(uuidObject);
    
    CFRelease(uuidObject);
    
    return uuidStr;
}
```

<!-- more -->

## 测试代码：

```objectivec
//
//  ViewController.m
//  test
//
//  Created by Ashbringer on 7/7/15.
//  Copyright (c) 2015 Dian.fm. All rights reserved.
//

#import "ViewController.h"
//#import "sole-master/sample.cc"

#include <uuid/uuid.h>

#include <iostream>
#include "sole.hpp"

int testMethod()
{
    sole::uuid u0 = sole::uuid0(), u1 = sole::uuid1(), u4 = sole::uuid4();
    
    std::cout << "uuid v0 string : " << u0 << std::endl;
    std::cout << "uuid v0 base62 : " << u0.base62() << std::endl;
    std::cout << "uuid v0 pretty : " << u0.pretty() << std::endl << std::endl;
    
    std::cout << "uuid v1 string : " << u1 << std::endl;
    std::cout << "uuid v1 base62 : " << u1.base62() << std::endl;
    std::cout << "uuid v1 pretty : " << u1.pretty() << std::endl << std::endl;
    
    std::cout << "uuid v4 string : " << u4 << std::endl;
    std::cout << "uuid v4 base62 : " << u4.base62() << std::endl;
    std::cout << "uuid v4 pretty : " << u4.pretty() << std::endl << std::endl;
    
    u1 = sole::rebuild("F81D4FAE-7DEC-11D0-A765-00A0C91E6BF6");
    u4 = sole::rebuild("GITheR4tLlg-BagIW20DGja");
    
    std::cout << "uuid v1 rebuilt : " << u1 << " -> " << u1.pretty() << std::endl;
    std::cout << "uuid v4 rebuilt : " << u4 << " -> " << u4.pretty() << std::endl;
    
    return 0;
}

@interface ViewController ()

@end

@implementation ViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    // Do any additional setup after loading the view, typically from a nib.
    
//    testMethod();
    [self testUUID];
    NSLog(@"%@", [self uuid]);
    for (int i = 0; i < 100; i++) {
    }
}

- (void)didReceiveMemoryWarning {
    [super didReceiveMemoryWarning];
    // Dispose of any resources that can be recreated.
}

- (void)testUUID
{
    uuid_t u;
    uuid_t ur;
    uuid_t ut;
    
    uuid_generate(u);
    uuid_generate_random(ur);
    uuid_generate_time(ut);
    
    char a[16];
    uuid_unparse(u, a);
    char b[16];
    uuid_unparse(ur, b);
    char c[16];
    uuid_unparse(ut, c);
    
//    printf("%s\n", a);
//    printf("%s\n", b);
    printf("%s\n", c);
}

- (NSString *)uuid{
    // Create universally unique identifier (object)
    CFUUIDRef uuidObject = CFUUIDCreate(kCFAllocatorDefault);
    
    // Get the string representation of CFUUID object.
    NSString *uuidStr = (NSString *)CFBridgingRelease(CFUUIDCreateString(kCFAllocatorDefault, uuidObject));
    
    // If needed, here is how to get a representation in bytes, returned as a structure
    // typedef struct {
    //   UInt8 byte0;
    //   UInt8 byte1;
    //   ...
    //   UInt8 byte15;
    // } CFUUIDBytes;
    //    CFUUIDBytes bytes = CFUUIDGetUUIDBytes(uuidObject);
    
    CFRelease(uuidObject);
    
    return uuidStr;
}


@end

```