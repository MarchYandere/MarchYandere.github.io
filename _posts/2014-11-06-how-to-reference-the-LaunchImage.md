---
comments: true
date: 2014-11-06 16:42:25
layout: post
title: how to reference the LaunchImage
categories:
- IOS
---

    LaunchImage-568h@2x.png  
    LaunchImage-700-568h@2x.png  
    LaunchImage-700-Landscape@2x~ipad.png  
    LaunchImage-700-Landscape~ipad.png  
    LaunchImage-700-Portrait@2x~ipad.png  
    LaunchImage-700-Portrait~ipad.png  
    LaunchImage-700@2x.png  
    LaunchImage-Landscape@2x~ipad.png  
    LaunchImage-Landscape~ipad.png  
    LaunchImage-Portrait@2x~ipad.png  
    LaunchImage-Portrait~ipad.png  
    LaunchImage.png  
    LaunchImage@2x.png  
    LaunchImage-800-667h@2x.png (iPhone 6)  
    LaunchImage-800-Portrait-736h@3x.png (iPhone 6 Plus Portrait)  
在app内，系统会生成如上所示图片，根据需求获取图片：
```object-c
    _imageview = [[UIImageView alloc] initWithFrame:self.view.bounds];
    
    GBDeviceDetails *detail = [GBDeviceInfoIOS deviceDetails];
    
    switch (detail.display) {
        case GBDeviceDisplayiPhone35Inch:
            _imageview.image = [UIImage imageNamed:@"LaunchImage"];
            break;
        case GBDeviceDisplayiPhone4Inch:
            _imageview.image = [UIImage imageNamed:@"LaunchImage-568h"];
            break;
        case GBDeviceDisplayiPhone47Inch:
            _imageview.image = [UIImage imageNamed:@"LaunchImage-800-667h"];
            break;
        case GBDeviceDisplayiPhone55Inch:
            _imageview.image = [UIImage imageNamed:@"LaunchImage-800-Portrait-736h"];
            break;
        default:
            _imageview.image = [UIImage imageNamed:@"LaunchImage"];
            break;
    }
```
ok，第一篇博文就酱紫啦！！！