---
title: NSDataDetector
categories: iOS
tags: 基础知识
date: 2016-11-29 11:55:47

---


# 初识
NSDataDetector 是 NSRegularExpression 的子类，而不只是一个 ICU 的模式匹配，它可以检测半结构化的信息：日期，地址，链接，电话号码和交通信息。

NSDataDetector 对象用一个需要检查的信息的位掩码类型来初始化，然后传入一个需要匹配的字符串。像 NSRegularExpression 一样，在一个字符串中找到的每个匹配是用 NSTextCheckingResult 来表示的，它有诸如字符范围和匹配类型的详细信息。然而，NSDataDetector 的特定类型也可以包含元数据，如地址或日期组件。

```objectivec
NSError *error = nil;
NSDataDetector *detector = [NSDataDetector dataDetectorWithTypes:NSTextCheckingTypeAddress
                                                        | NSTextCheckingTypePhoneNumber
                                                           error:&error];

NSString *string = @"123 Main St. / (555) 555-5555";
[detector enumerateMatchesInString:string
                           options:kNilOptions
                             range:NSMakeRange(0, [string length])
                        usingBlock:
^(NSTextCheckingResult *result, NSMatchingFlags flags, BOOL *stop) {
  NSLog(@"Match: %@", result);
}];
```

>当初始化 NSDataDetector 的时候，确保只指定你感兴趣的类型。每当增加一个需要检查的类型，随着而来的是不小的性能损失为代价。

<!-- more -->
# 数据检测器匹配类型
下面是 NSDataDetector 的各种 NSTextCheckingTypes 匹配，及其相关属性表：



类型 | 属性
---- | ----
NSTextCheckingTypeDate |<ul><li> dated </li> <li> uration </li> <li> timeZone </li> </ul>
NSTextCheckingTypeAddress | <ul><li> addressComponents* </li> <li> NSTextCheckingNameKey </li> <li> NSTextCheckingJobTitleKey </li> <li> NSTextCheckingOrganizationKey </li> <li> NSTextCheckingStreetKey </li> <li> NSTextCheckingCityKey </li> <li> NSTextCheckingStateKey </li> <li> NSTextCheckingZIPKey </li> <li> NSTextCheckingCountryKey </li> <li> NSTextCheckingPhoneKey </li></ul> 
NSTextCheckingTypeLink | <ul><li> url </li></ul>
NSTextCheckingTypePhoneNumber | <ul><li> phoneNumber </li></ul>
NSTextCheckingTypeTransitInformation | <ul><li> components* </li> <li> NSTextCheckingAirlineKey </li> <li> NSTextCheckingFlightKey </li></ul>

*号  NSDictionary properties have values at defined keys.

# 在 iOS 上做数据检测

有点混乱的是，iOS 也定义了 UIDataDetectorTypes。这些位掩码的值可以设置成一个 UITextView 的 dataDetectorTypes，来自动检测显示的文本。

UIDataDetectorTypes 和 NSTextCheckingTypes 相同的那些枚举常量其实是不同的（如 UIDataDetectorTypePhoneNumber 和 NSTextCheckingTypePhoneNumber），他们的整数值并不一样，而且一个中的所有值也并不能在另外一个里面都能找到。可以用以下方法把 UIDataDetectorTypes 转换为 NSTextCheckingTypes：

```objectivec
static inline NSTextCheckingType NSTextCheckingTypesFromUIDataDetectorTypes(UIDataDetectorTypes dataDetectorType) {
    NSTextCheckingType textCheckingType = 0;
    if (dataDetectorType & UIDataDetectorTypeAddress) {
        textCheckingType |= NSTextCheckingTypeAddress;
    }

    if (dataDetectorType & UIDataDetectorTypeCalendarEvent) {
        textCheckingType |= NSTextCheckingTypeDate;
    }

    if (dataDetectorType & UIDataDetectorTypeLink) {
        textCheckingType |= NSTextCheckingTypeLink;
    }

    if (dataDetectorType & UIDataDetectorTypePhoneNumber) {
        textCheckingType |= NSTextCheckingTypePhoneNumber;
    }

    return textCheckingType;
}
```

参考：[http://nshipster.com/nsdatadetector/](http://nshipster.com/nsdatadetector/)

UILabel的使用  [TTTAttributedLabel](https://github.com/TTTAttributedLabel/TTTAttributedLabel)