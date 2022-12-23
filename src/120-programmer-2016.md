## ENJOY 的 Apple Pay 应用内支付接入实践

文/陈乘方

>Apple Pay 的应用内支付提供了一种全新的在线支付形式，如果将 Apple Pay 应用内支付自身的特点与 App 本身的产品形态相结合，用户的在线支付体验将得到大幅提升。ENJOY 作为 Apple Pay 中国区首发的支持 Apple Pay 应用内支付的 App 之一，在跟 Apple Pay 的接入时与产品功能做了深度集成，本文基于此对包括可用性、payment sheet、服务器解密、交易处理等在内的 Apple Pay 技术接入要点进行了经验上的深度分享。

Apple Pay 已经在大陆地区正式上线，但大家的关注点大多集中在其线下支付的体验上。对于我们应用开发者而言，Apple Pay 的应用内支付给我们提供了一种全新的在线支付形式。如果将 Apple Pay 应用内支付自身的特点跟 App 本身的产品形态相结合，用户的在线支付体验可以得到大幅提升。

### Apple Pay 与现有支付方式对比

在国内开发包含在线支付功能的应用，目前可用的选择就是接入第三方支付平台，比如支付宝或者微信支付。这些支付方式在接入方式上大同小异，就是在 App 中引入对应平台的 SDK，在将支付信息组织好之后，调用对应第三方平台的 SDK 来完成支付。不同平台的 SDK 对支付的请求处理各不相同，总的来说完成支付有两种方式：调起对应的 App 或者打开一个网页。比如微信，就只支持打开微信 App 来进行支付这一种形式。

Apple Pay 与现有第三方支付平台相比的优点有：

- 系统级支持，支付过程不需要跳转到第三方 App；

- 支付过程可以获取用户信息，比如手机号、送货地址等。

Apple Pay 应用内支付的接入方式跟微信等第三方平台不一样。作为 iOS 系统原生支持的特性，Apple Pay 的相关功能包含在系统的 PassKit 这个 Framework 里，不需要引入第三方 SDK 便可集成。

### Apple Pay 深度集成

拿我们的产品 ENJOY 来说，作为 Apple Pay 中国区首发的支持 Apple Pay 应用内支付的 App 之一，在跟 Apple Pay 的接入时与产品功能做了深度集成。除了 Apple Pay 有着目前最短的支付路径这一特点，还有一个我们认为的最大优点，就是 Apple Pay 提供了系统级的由用户自行维护的个人信息。基于这些特点，与我们现有的用户系统和支付系统相结合，应用内支付体验有了很大提升。

ENJOY 与 Apple Pay 集成后特点：

- 未登录用户通过 Apple Pay 直接购买商品；

- 首页商品一键购买；

- 闪购商品一键购买；

- 比第三方支付提前一步完成购买。

其中最有亮点的地方就是第一点，未登录用户可以直接购买商品。就目前的电商应用来说，用户只有在登录应用之后，才能购买商品。而 ENJOY 之所以能做到这一点，是因为对 ENJOY 来说，只要能够拿到用户的手机号，便可以与我们的用户体系相关联，并完成购买流程。正是利用了手机号可以由 Apple Pay 提供给应用的这一特性，ENJOY 实现了未登录用户通过 Apple Pay 可以直接购买商品这一功能（如图1所示）。

<img src="http://ipad-cms.csdn.net/cms/attachment/201603/56d654dcb6130.png" alt="图1 未登录状态下购买商品时Payment sheet截图，可以看到其中的联系方式字段" title="图1 未登录状态下购买商品时Payment sheet截图，可以看到其中的联系方式字段" />

图1 未登录状态下购买商品时 Payment sheet 截图，可以看到其中的联系方式字段

### Apple Pay 技术接入要点

首先来看一下 Apple Pay 应用内支付的时序图（如图2所示）。

<img src="http://ipad-cms.csdn.net/cms/attachment/201603/56d654eb5d3ac.png" alt="图2 Apple Pay时序图" title="图2 Apple Pay时序图" />

图2 Apple Pay 时序图

从时序图中可以清楚地看到，应用内支付流程分为以下几个步骤：

- App 显示 payment sheet，提示用户进行支付验证操作（指纹和 PIN 码）；

- 用户支付验证完成，iOS 系统与 Apple 的服务器进行交互，处理支付信息（主要是加密）；

- App 收到系统回调，然后将回调里包含的支付信息传给自己的服务器；

- 自己服务器收到 App 发来的信息，进行解密处理后，将需要的信息组织成银联报文，然后调用银联扣款接口，完成扣款；

- 服务器通知 App 支付结果，App 更新相应的 UI。

由上面的步骤可以看出，虽然实际使用时 Apple Pay 支付的时间很短，但 App 从显示 payment sheet 一直到支付成功，其中的过程还是很复杂的。在实际开发中，有很多需要注意的地方，下面具体说明一下。

#### Apple Pay 可用性判断

应用如果显示 Apple Pay 按钮，需要先判断 Apple Pay 功能是否可用。这项实现需要结合 PassKit Framework 中 PKPaymentAuthorizationViewController 类的两个类函数才能完成判断，它们分别是：

```
class func canMakePayments() -> Bool
```

和

```
class func canMakePaymentsUsingNetworks(supportedNetworks: [String]) -> Bool
```

第一个函数返回值代表当前设备是否支持 Apple Pay，是用来判断设备硬件的。就目前来说，只有 iPhone 6、iPhone 6 Plus、iPhone 6s 和 iPhone 6s Plus 这四款设备支持 Apple Pay。

第二个函数则是判断指定的支付网络是否支持。Apple Pay 在国内的支付网络就是银联，所以需要传入
PKPaymentNetworkChinaUnionPay 这个值。

同时，需注意 API 的版本可用性，以上两个函数都是 iOS 8 才开始支持的 API，其中 PKPaymentNetworkChinaUnionPay 这个支付网络更是从 iOS 9.2 才开始支持，所以判断是否支持 Apple Pay 的最终代码是这样的：

```
if #available(iOS 9.2, *) {
         if PKPaymentAuthorizationViewController.canMakePayments() && PKPaymentAuthorizationViewController.canMakePaymentsUsingNetworks([PKPaymentNetworkChinaUnionPay]) {
                // TODO: 支持 Apple Pay
            }
}
```

注意：一定要把工程 target 设置的 Capabilities 里的 Apple Pay 选项打开，不然在真机调试的时候，以上代码判断结果肯定是不支持的。

#### Payment Sheet 显示

payment sheet 也就是 Apple Pay 支付的 UI，有不少地方是可以自定义的。想要自定义 payment sheet，需要对类 PKPaymentAuthorizationViewController 的初始化参数 paymentRequest 做相应的设置。paymentRequest 参数是一个 PKPaymentRequest 类型的对象（如图3所示）。

<img src="http://ipad-cms.csdn.net/cms/attachment/201603/56d654f8aa49a.png" alt="图3 iOS端Apple Pay结构" title="图3 iOS端Apple Pay结构" />

图3 iOS端 Apple Pay 结构

首先，初始化一个 PKPaymentRequest 对象，并设置必需内容：

```
let request = PKPaymentRequest()
request.merchantIdentifier = "merchant.xxxxx”
request.merchantCapabilities = [.Capability3DS, .CapabilityEMV, .CapabilityCredit, .CapabilityDebit]
request.countryCode = "CN"
request.currencyCode = "CNY"
request.supportedNetworks = [PKPaymentNetworkChinaUnionPay]
```

merchantIdentifier 需要与苹果开发者后台的配置相对应。以上代码中还有一个必需属性 paymentSummaryItems 没有设置，该属性代表在 payment sheet 中显示的价格列表，是一个 PKPaymentSummaryItem 类型的数组。该数组至少要有一个元素，并且最后一个元素所包含的价格，就是实际要支付的价格。如果需要显示手机号或者收货地址，则可以对 requiredShippingAddressFields 这个属性做相应的设置。

设置完毕后，通过以下代码可以调起 payment sheet：

```
let vc = PKPaymentAuthorizationViewController(paymentRequest: request)
vc.delegate = self
presentViewController(vc, animated: true, completion: nil)
```

关于 payment sheet 这个 UI 需要特别说明：这个 UI 的视图优先级相当高，连 alert 都会被盖住，App 在处理这个 UI 的时候需要注意。当 payment sheet 显示和消失时，对应的 App 也相当于进入后台和切回前台，所以，系统通知和 AppDelegate 里的回调函数也都会有所响应。

#### 服务器解密流程

在用户输入指纹和PIN码验证之后，回调函数 func paymentAuthorizationViewController(_, didAuthorizePayment: completion: )将会被调用，其中参数 didAuthorizePayment 类型是 PKPayment，里面包含两类信息。第一种是明文的用户信息，对应的是初始化 payment sheet 时的设置。另外一种就是加密过的支付信息，PKPayment 有一个 token 属性，该属性的类型是 PKPaymentToken，其中的 paymentData 属性就是加密过的支付信息。paymentData 的内容是一个 JSON 的二进制流，将其解析之后，结构如下：

```
{
    data = "xxxxx";
    header = {
        publicKeyHash = "xxxxx";
        transactionId = bce2f62f92cb1f5b16696f1ea3b1b375cbfe8fe45c9132ae381c77ed45202455;
        wrappedKey = "xxxx";
    };
    signature = "xxx";
    version = "RSA_v1";
}
```

使用 signature 字段对消息进行验签，防止信息被篡改。之后取出 header.wrappedKey 的内容，它是使用非对称加密算法加密过的对称秘钥。用苹果开发者后台配置 merchantID 时的私钥进行解密，就能得到这个对称秘钥。然后，使用这个对称秘钥对 data 所包含的加密数据进行解密，取得 Apple 返回的支付信息。此支付信息是加密过的，包含了用户支付的卡号和 PIN 码等信息，理论上只有银联才能解析出来真正的内容，对我们商户而言其内容是加密的。服务器端需要将这些解密过的信息组织成银联所需的报文内容，随后调用银联的扣款接口，完成扣款。

#### 银联交易状态确认

在调用银联的扣款接口后，有一点非常重要，就是要确保扣款结果是否成功。只有确切的知道了扣款结果，才能正确地设置自己平台的用户订单状态。银联的扣款接口设计是这样的，在调用接口之后，返回同时有同步和异步两种形式。如果同步结果返回成功，只是说明银联成功收到并开始处理扣款请求，并不代表扣款成功。扣款是否成功，是通过异步形式来通知的。而扣款不成功的原因有很多，比如卡被冻结、PIN 码错误、余额不足等等。

我们采用的做法是这样的，在调用扣款接口后，银联接口的同步返回结果里会包含对应的银联流水号。接下来设置一个等待时间，推荐的时间是5秒，如果5秒内收到了对应接口的异步回调，则直接处理自己的订单状态。如果超过5秒未收到异步回调，则由服务器使用银联流水号向银联发起交易状态接口的轮询请求，直到拿到这笔交易的确切结果为止。

#### 交易失败的处理

如果发生交易失败的情况，App 需要做相应的 UI 处理。按照 Apple 的交互要求，以下几种失败情况，payment sheet 不能退出，其余的情况则需要退出 payment sheet 后，由 App 负责通知给用户。

- 未输入 PIN 码；

- PIN 码错误；

- PIN 码失败次数超过限制。

payment sheet 的显示结果是由 func paymentAuthorizationViewController(_, didAuthorizePayment: completion: )这个回调函数中的 completion 这个参数来控制的。该参数是一个 closure，它的第一个参数类型是 PKPaymentAuthorizationStatus 的枚举，打开该枚举的定义，可以看到上面的情况已经由操作系统定义好了，对应起来分别是：

- PINRequired

- PINIncorrect

- PINLockout

UI 显示的细节：在调用 completion 后，如果传入的 PKPaymentAuthorizationStatus 值是 Success 或者 Failure，payment sheet 在展示出相应的结果之后都会自动 dismiss 掉。而如果传入的是上面三个有关 PIN 码的值，则 payment sheet 并不会消失，用户可以继续选择银行卡，然后进行指纹验证等支付操作。

#### 交易撤销

在 payment sheet 显示出来到消失的这段时间内的任意时刻，用户都可以通过点击 payment sheet 右上角的“取消”按钮来手动取消支付，那这就涉及到了交易撤销的问题。交易撤销是由服务器来完成的，那么 App 要做的首先是把用户取消的事件通知给服务器。由于整个支付过程步骤很多，总的来说，服务器撤销交易要面临以下三种情况：

- 还未向银联调用扣款接口；

- 已调用扣款接口，但扣款结果还未返回；

- 扣款成功。

第一种情况很容易处理，只要服务器之后不再调用银联扣款接口即可。第二种和第三种情况的处理方法一样，区别在于第二种情况的处理时机是在获取到扣款成功结果之后。如果是后两种情况，则调用银联的交易撤销接口，以撤销之前的交易，把已经扣除的用户款项返还给用户。

注意：由于有些银行不支持交易撤销，所以调用交易撤销接口后需要判断一下返回结果，如果不支持，则还需调用退款接口来返回款项。

### 总结

Apple Pay 应用内支付在技术上并不难，但所涉及的细节非常多，需要花费不少功夫处理好各种情况。如果对 Apple Pay 应用内支付定制化需求并不强，有不少第三方支付平台目前已经提供了自己封装的SDK来供开发者使用，比如银联的 CUP SDK 等。这些 SDK 屏蔽了支付过程中的很多细节，可以方便地接入到 App 中。但如果需要接入原生的 Apple Pay 来实现好的应用内支付体验，那么本文可以给你提供一些帮助。
