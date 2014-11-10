---
comments: true
date: 2014-11-06 16:42:25
layout: post
title: In App Purchases
categories:
- IOS
---

 >前些日子，项目用到了程序内置购买，现在正好有点空闲时间，决定把它记录下来，废话少说，直接开干：

配置篇
-------------

 1. 登录到开发者中心，创建APP ID。
 2. 登录iTunes Connect，创建App。
 3. 选择刚刚创建的App，然后选择In-App Purchase，选择create new按钮，它会提示你选择商品销售类型。 
 4. 填写product内容，Product ID确保唯一性，客户端程序通过Product ID来购买商品。
 5. 创建沙盒测试用户。回到iTunes connect主页--->Users and Roles --->Sandbox Testers。 
 > **Note:**
 - Consumable  购买一次，使用一次
 - Non-Consumable  购买一次，永久使用
 - Auto-Renewable Subscriptions 自动更新
 - Free Subscription  免费
 - Non-Renewing Subscription 非自动更新

 
 开发篇
-------------
### 提取产品列表 ###
 ----------
在你让你的用户购买你的商品之前，你必须向iTunes Connect发送一条请求，获取产品列表。

```
NSSet *productIdentifiers = [NSSet setWithObjects: @"xxxxxxxx", nil];
SKProductRequest *request = [[SKProductsRequest alloc] initWithProductIdentifiers:productIdentifiers];
request.delegate = self;
[request start];

```
实现SKProductRequestDelegate中的协议productsRequest: didReceiveResponse，当通过[request start]访问iTunes完成后，会调用这个回调。
```
#pragma mark - SKProductsRequestDelegate
- (void)productsRequest:(SKProductsRequest *)request didReceiveResponse:(SKProductsResponse *)response
{
    NSArray *products = response.products;
    
    if ([products count] == 1) {
        [self buyProduct:[products objectAtIndex:0]];
    }
    else {
        if (_delegate && [_delegate respondsToSelector:@selector(failedPurchases:)]) {
            
            NSDictionary *dict = [NSDictionary dictionaryWithObjectsAndKeys:@"沒有相關商品", NSLocalizedDescriptionKey, nil];
            NSError *error = [NSError errorWithDomain:@"" code:0 userInfo:dict];  
        }
    }
}
```
 > **Note:**因request传入的参数只有一个，所以必然返回的product的信息也只能是一个，如果多个或者别去的情况，那是我们不想要的，假设当前这个类是InAppPurchaseHelper，我们可以新建一个协议"InAppPurchaseHelperDelegate", 然后上面代码片中，通过代理通知viewController或者其他什么：[_delegate failedPurchases:error];

如果出现访问失败的情况，这时你会去SKProductRequestDelegate里找相应的回调，发现里面只有一个方法。如果你心细，你会发现SkProductRequestDelegate协议是从SKRequestDelegate派生而来：
```
@protocol SKRequestDelegate <NSObject>

@optional
- (void)requestDidFinish:(SKRequest *)request NS_AVAILABLE_IOS(3_0);
- (void)request:(SKRequest *)request didFailWithError:(NSError *)error NS_AVAILABLE_IOS(3_0);

@end
```
实现访问失败协议：
```
- (void)request:(SKRequest *)request didFailWithError:(NSError *)error
{
    _Log(@"error = %@", [error localizedDescription]);
    if (_delegate && [_delegate respondsToSelector:@selector(failedPurchases:)]) {
        [_delegate failedPurchases:error];
    }
}
```
### 购买商品 ###
 ----------
开始购买
```
- (void)buyProduct:(SKProduct *)product
{
    if ([SKPaymentQueue canMakePayments]) {
        
        SKPayment *payment  = [SKPayment paymentWithProduct:product];
        [[SKPaymentQueue defaultQueue] addPayment:payment];
        
    }
    else {
        if (_delegate && [_delegate respondsToSelector:@selector(failedPurchases:)]) {
            
            NSDictionary *dict = [NSDictionary dictionaryWithObjectsAndKeys:@"您無法從AppStore購買產品，請修改相關設置", NSLocalizedDescriptionKey, nil];
            NSError *error = [NSError errorWithDomain:@"" code:0 userInfo:dict];
            [_delegate failedPurchases:error];
        }
    }
    
}
```
创建一个SKPayment对象，然后把对象加到队列中去。当支付成功或者失败时，paymentQueue:updatedTransactions 这个函数将会被调用。
```
#pragma mark - SKPaymentTransactionObserver
- (void)paymentQueue:(SKPaymentQueue *)queue updatedTransactions:(NSArray *)transactions
{
    for (SKPaymentTransaction *transaction in transactions) {
        switch (transaction.transactionState) {
            case SKPaymentTransactionStatePurchased:
            {
                [self completeTransaction:transaction];
                break;
            }
            case SKPaymentTransactionStateFailed:
            {
                [self failedTransaction:transaction];
                break;
            }
            case SKPaymentTransactionStateRestored:
            {
                [self restoredTransaction:transaction];
                break;
            }
            default:
                break;
        }
    }
}
```
当然购买成功时，程序就会跳到completeTransaction:方法，苹果会给你一个回执，你可以用来做校验在iTunes内是否购买成功。restoredTransaction:方法的内容和成功的一样，失败后，failedTransaction：所做的事情只是告诉代理购买失败，然后返回error信息。
```
- (void)completeTransaction:(SKPaymentTransaction *)transaction
{
    
    NSString *receipt = [[NSString alloc] initWithData:[transaction transactionReceipt] encoding:NSUTF8StringEncoding];
    
    NSString *result = [self receiptString:receipt];
    
    NSDictionary *resultData = @{@"bookid" : _bookid, @"receipt" : result};
    
    if (_delegate && [_delegate respondsToSelector:@selector(completePurchases:)]) {
        [_delegate completePurchases:resultData];
    }
    [[SKPaymentQueue defaultQueue] finishTransaction:transaction];
}
```
一般情况下，我们可以讲这个类InAppPurchase的代理设给AppDelegate，这样无论你在app的任何地方购买商品，app都能捕获到信息，简化了代码。
```
#pragma mark - InAppPurchasesHelperDelegate

- (void)startPurchases
{
    [MBProgressHUD showHUDAddedTo:self.window animated:YES];
}

- (void)completePurchases:(id)obj
{
    [MBProgressHUD hideHUDForView:self.window animated:YES];
    _Log(@"obj = %@", obj);
    //anything you want do
}

- (void)failedPurchases:(NSError *)error
{
    [MBProgressHUD hideHUDForView:self.window animated:YES];
    [UIAlertView alertWithTitle:[error localizedDescription]];
}

```
 OK，第二篇博文，就酱紫啦，各位看官姥爷，欢迎指正哦！！！
