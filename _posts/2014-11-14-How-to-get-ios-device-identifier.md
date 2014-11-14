---
comments: true
date: 2014-11-14 15:36:25
layout: post
title: How to get ios device identifier
categories:
- IOS
---

###背景###

----------

在ios5以前，UIDevice 提供一个获取设备唯一标识的方法 uniqueIdentifier，因为这个唯一标识与设备一一对应，苹果觉得可能会造成用户隐私泄露，所以在ios6以后，用了两个方法替代了uniqueIdentifier方法：identifierForVendor和advertisingIdentifier。但是，当删除app后，再安装app会生成不同的identifier，这显然不是我们想要的结果。于是乎，大家想到了用wifi的mac地址来充当唯一标识。又但是，在ios7中，苹果无情的封杀了mac地址，使用之前的方法获取到的mac地址02:00:00:00:00:00。

###Keychain###

----------

**Introduction：**

keychain是苹果提供给用户保存私密数据的地方，比如密码，证书，keys，等等。相比于NSUserDefault存储数据，keychain更安全，它不会因为app的删除而丢，在用户重新安装app后，数据依然有效。在ios里面，每个app只能访问自己在keychain里的数据。苹果还提供了访问同一开发商的多个app的keychain的途径，下文将做介绍。

一条item（keychain）主要包括三个部分：

 - class：keychain的类型，一般情况都会使用kSecClassGenericPassword 作为value值。
 - attributes：不同的item（keychain）会包含不同的attribute，这是查询keychain时，区分item的主要条件。
 - secValue：每条item会包含一条密码项来存储密码。

###同一开发商的多个app的keychain共享###

----------

在实际开发情况中，我们经常会有这样的情况，同一开发商的多个app之间会有很多内容共享，最直白的例子就是密码。想要做到这一步，需要3个步骤：

 1. 在app target的build setting里面设置Code Signing Entitlements![enter image description here](http://wutao.me/upload/2014/11/14/1.png)
 2. 在工程目录下新建一个KeychainAccessGroups.plist文件，改文件必须和工程文件在同一目录。该文件的结构中最顶层的节点必须是一个名为“keychain-access-groups”的Array，并且该Array中每一项都是一个描述分组的NSString。对于String的格式也有相应要求，格式为:"AppIdentifier.com.xxxx"，其中AppIdentifier就是你的签名中最前面的那部分![enter image description here](http://wutao.me/upload/2014/11/14/2.png)
 3. 在调用SecItemAdd添加数据的时候指定kSecAttrAccessGroup。
 

**代码：**

首先，导入Security.framework，添加两个常量。

```
static const char kKeychainUDIDItemIdentifier[]  = "UUID";
static const char kKeyChainUDIDAccessGroup[] = "AppIdentifier.com.xtownmobile.ccleReading";
```

无论是增加、删除、更新、查找都需要一个dictionary来作为查询条件，所以我们可以写一个创建dictionary的方法，里面包含一些必要元素。

```
- (NSMutableDictionary *)newSearchDictionary:NSString *)identifier {
	NSMutableDictionary *searchDictionary = [[NSMutableDictionary alloc] init];  

	[searchDictionary setObject:(id)kSecClassGenericPassword forKey:(id)kSecClass];

	NSData *encodedIdentifier = [NSData dataWithBytes:identifier length:strlen(identifier)];
	[searchDictionary setObject:encodedIdentifier forKey:(id)kSecAttrGeneric];
	NSString *accessGroup = [NSString stringWithUTF8String:kKeyChainUDIDAccessGroup];
	
if (accessGroup != nil)
    {
#if TARGET_IPHONE_SIMULATOR
//nothing to do
#else
        [searchDictionary setObject:accessGroup forKey:(id)kSecAttrAccessGroup];
#endif
    }
  return searchDictionary;
}
```

查询：

```
- (NSString *)searchKeychainCopyMatching:(NSString *)identifier {
	NSMutableDictionary *searchDictionary = [self newSearchDictionary:identifier];

  // Add search attributes
	[searchDictionary setObject:(id)kSecMatchLimitOne forKey:(id)kSecMatchLimit];

  // Add search return types
	[searchDictionary setObject:(id)kCFBooleanTrue forKey:(id)kSecReturnData];

	NSData *result = nil;
	NSString *udid = @"";
	OSStatus status = SecItemCopyMatching((CFDictionaryRef)searchDictionary,
                                        (CFTypeRef *)&result);

	if (result) {
        udid = [NSString stringWithUTF8String:result.bytes];
    }
  return udid;
}
```

增加

```
- (BOOL)createKeychainValue:(NSString *)udid forIdentifier:(NSString *)identifier {
	NSMutableDictionary *dictionary = [self newSearchDictionary:identifier];

	NSString *accessGroup = [NSString stringWithUTF8String:kKeyChainUDIDAccessGroup];
    if (accessGroup != nil)
    {
#if TARGET_IPHONE_SIMULATOR
        [dictionary setObject:accessGroup forKey:(id)kSecAttrAccessGroup];
#endif
    }

	const char *udidStr = [udid UTF8String];
    NSData *keyChainItemValue = [NSData dataWithBytes:udidStr length:strlen(udidStr)];
	[dictionary setObject:keyChainItemValue forKey:(id)kSecValueData];

	OSStatus status = SecItemAdd((CFDictionaryRef)dictionary, NULL);
	if (status == errSecSuccess) {
	    return YES;
	}
	return NO;
}
```

删除、更新都大同小异啦，不再做过多描述。