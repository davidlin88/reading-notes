# HTTP/2 推送比我想象得更加艰难
摸鱼时偶然看到这篇文章，兴起翻译了一下。也是第一次尝试翻译技术文档，写到一半发现已经是17年的文章了，结论的时效性已经不强，但私心又不舍半途而废，因兴致而起的行为总是落地了、有了产出才会更有成就感。

原文链接：[HTTP/2 push is tougher than I thought](https://jakearchibald.com/2017/h2-push-tougher-than-i-thought/)

分割线以下为译文。
<hr>

说到页面加载性能问题的时候，我经常听到说：“`HTTP/2推送`会解决那个的。”但我不太了解它，所以我决定深究一下。
HTTP/2 推送比我刚开始想象得更复杂和底层，但真正让我措手不及的是它在浏览器之间表现得不一致性，我一度以为这都是板上钉钉、准备上生产的东西了。

这不是一篇所谓“HTTP/2推送是个辣鸡”的吐槽文 ——我觉得 HTTP/2 推送是真的很强大，而且未来会发挥作用，但我不会再觉得它是拿来就能用的东西了。

## Map of fetching

在你的页面和模板服务器之间，有一系列缓存之类的东西，它们能够阻塞 `intercept`请求：

![缓存](https://img-blog.csdnimg.cn/20200108110602201.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzMxMzkzNDAx,size_16,color_FFFFFF,t_70)

上图看起来很像那些”流程图怪“们用来解释 `Git`或者 `观察者模式`的东西，这种图对那些已经了解这东西的人很友好 ，但对不了解它们的人就很糟糕。如果你也这么觉得，那很抱歉orz！希望下面的部分能帮到你。

## HTTP/2推送 是如何工作的？

![页面与服务器对话](https://img-blog.csdnimg.cn/20200108110806591.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzMxMzkzNDAx,size_16,color_FFFFFF,t_70)

* 页面：嗨，example.com，我可以请求你的主页吗？
* 服务器：当然！奥，但当我在发送你主页的同时，还会发你一份表格、一些图片、一些JavaScript、一些JSON。
* 页面：额，当然。
* 页面：我刚才读取到HTML了，看上去我需要一个表格…奥！你已经发给我了，棒！

当服务器对请求作出响应的时候，可能包含了额外的资源，包括请求头的设置。所以浏览器知道之后该怎么去匹配它。它们被放在缓存里，直到浏览器去请求这些匹配的资源。

你可以获得性能提升，因为你没有等浏览器来请求资源就发送了他们。理论上说，这意味着页面加载得更快。

这跟我之前了解的HTTP/2推送差不多，听起来好像还蛮简单，但坑爹就坑在它的细节里。。

## 什么都能用推送缓存

HTTP/2推送是一个底层的网络特性——任何使用网络堆 `networking stack`的东西都能使用它。让它真正生效的关键是一致性和可预测性。

我试了试 `give this a spin` 推送资源，并用以下方式获取：

* fetch()
* XMLHttpRequest
* \<link rel="stylesheet" href="…">
* \<script src="…">
* \<iframe src="…">

我也减慢了所推送资源的传输速度，来观察资源还在推送过程中时，浏览器能不能匹配到资源。一些零碎的测试我放到了[github](https://github.com/jakearchibald/http2-push-test)上。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200108111318507.png)

* **Edge** 当使用`fetch()`、`XMLHttpRequest`和`<iframe>`时，没有从推送缓存中获取资源。[（issue，包含视频）](https://developer.microsoft.com/en-us/microsoft-edge/platform/issues/12142852/)
* **Safari** 是个怪胎，用不用推送缓存就像投硬币。Safari源自OSX的闭源的网络堆 `network stack`，但我觉得有些bug是Safari内部的。看起来它开启了太多连接，被推送的资源最终就分散在这些连接之间。这表示只有那个请求幸运地刚好使用了相同的连接时，你才能命中缓存——但这实在超出我的认知了。[（issue，包含视频）](https://bugs.webkit.org/show_bug.cgi?id=172639)

所有的浏览器（除了抽风时的Safari）都会匹配被推送的资源，即使他们的推送仍在进行中。这就很nice。

不幸的是，Chrome是唯一一个有开发工具支持的浏览器。它的network标签页会告诉你哪些是从推送缓存里被拉取的资源。

### 建议
如果浏览器没有从推送缓存中拉取资源，你最终会比没有推送得更慢。

Edge的支持很糟，但至少它糟得很稳定。你可以检测`user-agent`来确保只推送你知道能被使用的资源。如果出于某些原因做不到这一点的话，避免推送任何资源给Edge用户会更安全。

Safari的行为有不确定性，所以你不能通过hack来修正它。检测`user-agent`来避免向Safari用户推送资源吧。

## 您可以推送 no-cache 和 no-store 的资源
使用HTTP缓存的资源时，必有一些像`max-age`的东西来允许浏览器避免服务器的再校验。[（这是一个post请求的缓存头）](https://jakearchibald.com/2016/caching-best-practices/)HTTP/2 则不一样，推送资源时并不检查资源的新鲜度 `freshness`。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200108111531121.png)

所有浏览器都是这样。

### 建议
一些单页面应用受困于性能表现：因为他们不仅会被JS阻塞渲染，也被一些JS执行后才加载的数据数据所阻塞（JSON或blabla）。服务端渲染是最好的场景，如果没条件做的话你可以在发送页面的时候同时推送JS和JSON。

但是，像之前给Edge/Safari提的issue里提到的，内联的JSON更可靠。

## HTTP/2推送是浏览器最后检查的缓存
用HTTP/2推送资源意味着：在它提供响应之前，如果什么缓存都没有的话，浏览器才会使用被推送的资源。包括图片缓存、预加载的缓存、`service workder` 和HTTP缓存。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200108111823290.png)

所有浏览器在这点上表现一致。
### 建议
注意。我举个例子，如果你在HTTP缓存中有一个匹配项，根据`max-age`它是有效的，然后你推送了一个更新的资源，因为HTPP缓存中旧资源的存在，新的资源将被忽视（除非由于某些原因这个API绕过了HTTP缓存）。

成为链条上最后的一节真的不是一个问题，但知道缓存项和连接能帮助我理解许多我所看到的别的行为。

## 如果连接关闭了，拜拜👋推送缓存~
推送缓存依赖于HTTP/2 连接，所有如果连接关闭它就gg了。就算被推送的资源具有高度可缓存性这也会发生。

推送缓存超于HTTP缓存之外，所以直到浏览器请求他们，缓存项才会进入HTTP缓存。那时，他们会通过HTTP缓存、service workder  等等来把推送的缓存拉取到页面。

如果用户的连接不稳定，或许你成功地推送了一些东西，但在页面获取到他们之前，连接丢失了。这样的话用户不得不设置一个新的连接然后重新下载资源。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200108113657641.png)

所有浏览器在这一点上表现一致。

### 建议
不要依赖在推送缓存中等待了很久的资源。推送是紧急资源的最佳实践，所以在资源推送和页面接受到之间不应该由很长时间。

## 多个页面可以使用相同的HTTP/2推送
每个连接都有它自己的推送缓存，但是多个页面能使用单次连接，意味着多页面可能分享推送缓存。

在实践中，这意味着如果你单独推送了一个跳转响应（比如一个HTML页面），它不只是单独对那个页面有效（在本文的其他部分中，我将使用“页面”来描述，但实际生产中这包括了其他的上下文，比如`workers`）。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200108142445789.png)

**Edge** 看起来在每个标签页使用了一个新的连接（[issue，包含视频](https://developer.microsoft.com/en-us/microsoft-edge/platform/issues/12144186/)）

**Safari**不必要地对同源创建了多个连接。我很肯定这是它表现怪异的根源。（[issue，包含视频](https://bugs.webkit.org/show_bug.cgi?id=172639)）
### 建议
小心：当你将JSON数据之类的东西和页面一起推送时，你不能指望相同的页面也使用它。

这个行为可以变成一个优点，比如你可以使用通过由一个安装中的`service worker` 创建的请求来和一个页面一起推送的资源。

Edge的行为不是最佳的，但现在不是该担心这个的时候。一旦 Edge 有了`service worker` 的支持，它就*可能*大有作为。

再说一遍，我倾向于不对Safari用户使用推送缓存。

## 没有凭据的请求使用一个单独的连接
“凭据”`Credentials`一词将会在这篇文章多次出现。凭据是浏览器发送的用来鉴别单独用户的东西。它通常是cookies，但也可以是HTTP基础授权和连接级的认证，比如客户端证书。

如果你把1个HTTP/2连接看成是一次通话，一旦你介绍了你自己，这个通话将不再匿名：包括你之前说的所有内容。出于隐私的原因，浏览器对匿名请求设置了一个单独的“通话”。

但是，因为推送缓存基于连接，你可能会因为创建了无凭据的请求，最终丢失缓存项。譬如，如果你随页面推送了一个资源（一次有凭据的请求），然后`fetch()`了它（无凭据），将会设置一个新的连接，然后你就会丢失推送项。

如果一个跨域的样式表（有凭据的）推送了一个字体，那浏览器的字体请求（无凭据）将会在推送缓存中丢失。

### 建议
确保你的请求使用了相同的凭据模式。大部分情况下这意味着确保你的请求包含凭据，因为你的页面请求始终都会携带凭据。

要携带凭据地拉取，用：
```js
fetch(url, {credentials: 'incluse'});
```
你不能对一个跨域的字体请求添加凭据，但你*可以*把他们从样式表中移除：
```html
<link rel="stylesheet" href="…" crossorigin>
```
…这意味着样式表和字体都会走相同的连接。但是，如果此样式表也应用了背景图的话，那些请求始终是认证的，所以最终你将再次获得一个连接。这里唯一的问题是`service worker`，它能改变fetch在每个请求中是如何表现的。

我曾听有的开发者说：无凭据的请求表现更好，因为他们不需要发送cookie。但你要想想：创建一个新的连接还需要更多的消耗呢。而且，HTTP/2可以压缩掉请求之间重复的请求头，所以cookies真是不是什么大问题。

### 或许我们需要改变规则
Edge是此处唯一没有遵守规则的浏览器。它允许有凭据和无凭据的请求共享同一个连接。不过，我不打算在这里放浏览器支持度的对比，因为我希望看到规则的改变。

如果一个页面对它的源创建了一个无凭据的请求，那开启一个单独的请求便毫无意义。一个有凭据的资源发起了该请求，所以它可以把它的凭据通过URL添加到那个“匿名”请求上。

对别的场景我不太确定，但由于浏览器指纹的存在，如果你对同一个服务器创建了有凭据和无凭据的请求，没有太多匿名的方式。如果你想深究这个问题，这是[github上的讨论](https://github.com/whatwg/fetch/issues/341)、[一个Mozilla的邮件列表](https://groups.google.com/forum/#!topic/mozilla.dev.tech.network/glqron0mRko/discussion)、和[Firefox的bug跟踪](https://bugzilla.mozilla.org/show_bug.cgi?id=1363284)。

噫，专业名词用得稍微有点多，不好意思。

## 推送缓存中的项只能被用一次
一旦浏览器使用了推送缓存中的东西，它就会被移除。它可能会最终在HTTP缓存里（取决于[缓存头](https://jakearchibald.com/2016/caching-best-practices/)），但不会再出现在推送缓存里了。

![在这里插入图片描述](https://img-blog.csdnimg.cn/2020010814563226.png)

**Safari**在这里存在一个竞争关系。如果一个资源在它推送的时候被拉取多次，它就会被推送多次（[issue，包含视频](https://bugs.webkit.org/show_bug.cgi?id=172639)）。如果在资源已经结束推送后，它的表现才会正确，被拉取两次——第一次会返回推送缓存，而第二次不会。

### 建议
如果你决定推送东西给Safari用户，当你推送非缓存资源时，留意这个bug。可能会随响应返回一个随机的ID，如果你获取了相同的ID两次，那你就命中了这个bug。这时，请等一秒钟重新尝试。

总体来说，**使用缓存头或者一个`service worker`来缓存已经被拉取到的推送资源，除非它不需要缓存**（比如一次性的JSON）。

## 浏览器可以中止已拥有的推送项
当你推送内容时，与客户端是没有太多交互的。这意味着你可能推送了浏览器已经有了的缓存。在这种场景下，HTTP/2规范允许浏览器通过`CANCEL`或`REFUSED_STREAM`的CODE来中止传输中的流，以此来避免浪费带宽。

此处的规范不是严格的，所以这里我的意见是建议开发者视情况而定。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200108145701536.png)

**Chrome**：如果推送项已经在推送缓存中了，它会拒绝推送。它更倾向于使用`PROTOCOL_ERROR`而不是`CANCEL`或`REFUSED_STREAM`，但这是次要的（[issue](https://bugs.chromium.org/p/chromium/issues/detail?id=726725)）。很可惜，Chrome不会拒绝已经存在于HTTP缓存中的推送项。听起来这几乎是一个已确定的问题，但我没能测试它（[issue](https://bugs.chromium.org/p/chromium/issues/detail?id=232040)）。

**Safari**：如果推送项已经在推送缓存中了，它会拒绝推送，但只有当推送缓存的项根据缓存头（比如max-age）是新鲜的，除非用户点击刷新。这跟Chrome不一样，但我不觉得这是“错”的。可惜的是，跟Chrome一样，它不会拒绝已经存在于HTTP缓存中的项。（[issue](https://bugs.webkit.org/show_bug.cgi?id=172646)）

**Firefox**：如果推送项已经在推送缓存中了，它会拒绝推送，但它随后它会删除推送缓存中已经存在的项目，导致什么都不剩！这使得它非常不可靠，而且很难防备（[issue，包含视频](https://bugzilla.mozilla.org/show_bug.cgi?id=1368080)）。Firefox也不会拒绝在HTTP缓存中已经存在的项目（[issue](https://bugzilla.mozilla.org/show_bug.cgi?id=1367551)）。

**Edge**不会拒绝已经存在于推送缓存中的项目，但如果已经存在于HTTP缓存中，它会拒绝。

### 建议
可惜的是，即使有完美的浏览器支持，在获得取消的消息之前，你也会浪费带宽和服务器I/O。[缓存摘要](http://httpwg.org/http-extensions/cache-digest.html)旨在解决这个问题，通过预先告知服务器它缓存了哪些内容。

同时，如果你已经推送了静态资源给用户，你可能想通过cookie来跟踪。但是由于浏览器的怪异表现，项目可能从HTTP缓存中消失，而cookie却仍然存在，因此，cookie的存在并不意味着用户仍然用户缓存资源。

## 除了更新之外，还应使用HTTP语义匹配推送缓存中的项目
我们已经知道，被匹配到推送缓存中的更新会被忽视（这就是`no-store`和`no-cache`项被匹配的方式），但其他匹配机制应该被使用。我测试了`POST`请求，和`Vary: Cookie`。

**更新**：规范提到了推送请求“必须是可缓存的，必须是安全的，必不能包含一个请求体”——刚开始我遗漏了这些定义。`POST`请求不能算是“安全”的范畴，所以浏览器应该拒绝`POST`请求。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200108145819488.png)

**Chrome**接受`POST`推送流，但似乎没有使用他们（[issue](https://bugs.chromium.org/p/chromium/issues/detail?id=727653)）。匹配推送项时，Chrome也忽视了Vary头，即使这个issue表明使用QUIC时它会生效。

`Firefox`拒绝了 POST 的推送流。但匹配推送项时，Firefox也忽视了 Vary 头。（[issue](https://bugzilla.mozilla.org/show_bug.cgi?id=1368664)）

`Edge`同样会拒绝 POST 的推送流，也忽视了 Vary 头。（[issue](https://developer.microsoft.com/en-us/microsoft-edge/platform/issues/12173342/)）

`Safari`，类似Chrome，接受了 POST 的推送流，但看起来没有使用他们（[issue](https://bugs.webkit.org/show_bug.cgi?id=172706)）。它确实遵守了Vary头，也是浏览器中唯一这么做的。

### 建议
我有点难过，只有Safari观察了推送项的Vary头。这意味着可能会发生这种情况：你推送一些JSON给一个用户时，这个用户注销了而另一个用户登录，但你仍然会将上一个用户的JSON推送给新用户（如果上个用户还未收到）。

如果你要推送数据给某个用户，请同时携带预期的用户ID。如果跟你预期不符，请重新请求（之前的推送项将会丢失）。

在Chrome里，当用户注销时，你可以使用[Clear Site Data头部](https://www.w3.org/TR/clear-site-data/)（译者注：[mdn文档](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Headers/Clear-Site-Data)）的功能。这个也能通过中止HTTP/2连接来清除缓存的项。

## 你可以推送其他源的项目
作为 developers.google.com/web 的所有者，我们可以让我们的服务器推送包含我们想要的 android.com 内容的响应，并将其设置为缓存一年。一个简单的拉取就能把它拖到HTTP缓存里。之后，如果我们的访客去到 android.com ，他们就可以看到用大号粉丝风格的“NATIVE SUX – PWA RULEZ”，或者别的我们的要的任何东西。

当然，我们不会这么做，我们爱安卓。我就是说说而已…安卓：如果你弄乱了网页，我们就爱你（fuck you up）。

好吧~我开玩笑的，但以上确实可以发生。你无法推送静态资源给*任意源*，但你可以推送资源给那些你的连接具有权限性的源。

如果你看一下 developer.google.com 的证书，你可以看到：它对任何Google的源都有权威性，包括 android.com。

[油管视频地址](https://www.youtube.com/watch?time_continue=26&v=i9uXp64KUcw&feature=emb_title)

其实刚刚，我撒了个小谎，因为当我们拉取 android.com 时，它会执行DNS 查找，然后发现它的IP跟developers.google.com不同而终止，所以它会设置一个新的连接，从而丢失我们在推送缓存中的项。

我们可以用一个 ORIGIN 框架来达到这个效果。这允许连接说：“嗨，如果你需要 android.com 的任何东西，问我就对了。”只要它是有权限的，就不用去管DNS什么的。这对一般的连接合并很有效，但它太新了，仅在Firefox Nightly版中得到了支持。

如果你正在使用一个CDN或者是某些共享的域名，检查一下证书，可以看到哪些源能够向你的网站推送内容。有点骇人听闻。还好，据我所知，没有主机能够完全控制HTTP/2推送，或许要感谢规范中的这些内容：
> 多个承租方共享同个服务器的空间时，服务器**必须**确保承租方不能推送他们所没有权限的资源表现 representations of resources。
> ——HTTP/2 规范

理应如此。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200108145853970.png)

**Chrome**允许网站推送所拥有授权的源。如果其他源来自相同的IP，它会复用连接（那些被使用的推送资源也是一样）。Chrome还不支持 ORIGIN frame。

**Safari**允许网站推送已授权的资源，但它为其他源设置了一个新的连接，那些未使用的truism项也是。Safari不支持 ORIGIN frame。

**Firefox** 拒绝了其他源的推送。跟Safari类似，它为其他源设置了一个新连接。可是，我在Firefox里收到了证书警告，所以我不是很自信我的结论。Firefox Nightly版支持ORIGION frame。

**Edge**也拒绝了其他源的推送。而我又一次收到了证书警告，所以在用一个合适的证书后，这些结论可能会有些不同。Edge不支持ORIGIN frame。

### 建议
如果你在同一个页面使用了来自相同服务器的多个源，可以开始看看 ORIGIN frame了。一旦它被支持，就可以不需要DNS查找了，以此来提示性能。

如果你认为跨域推送给你带来更好的收益，写一些比我写得更好的测试，并确保浏览器会真的使用你所推送的资源。否则，使用user-agent检测来推送给不同的浏览器吧。

## 推送 vs 预加载
在使用HTML时，你也可以让浏览器预加载资源，而不是推送资源：
```html
<link
  rel="preload"
  href="https://fonts.example.com/font.woff2"
  as="font"
  crossorigin
  type="font/woff2"
>
```
或是一个页面头：
```js
Link: <https://fonts.example.com/font.woff2>; rel=preload; as=font; crossorigin; type='font/woff2'
```
- `hresf` - 需要预加载的URL
- `as` - 资源的目的地。这表示浏览器可以设置正确的头并应用正确的内容安全策略。
- `crossorigin` - 可选。表明请求应该是一个CORS请求。CORS请求将不带凭据地被发送，除非是`crossorigin="user-credentials"`。
- `type` - 可选。如果被提供的MIME类型不支持的话，允许浏览器忽视这个预加载。

一旦浏览器发现了一个预加载的链接，就会拉取它。这个功能与HTTP/2推送相似：

- 什么东西都能预加载。
- `no-cache`和`no-store`的项目能被预加载
- 你的请求将只会匹配凭据模式相同的预加载项目。
- 缓存项只能被使用一次，即使他们可能出现在将来拉取的HTTP缓存中。
- 项目应该使用HTTP语义匹配，除了更新的资源 freshness
- 你可以从其他源中预加载项目。

但也有所不同：
**浏览器拉取资源**，会从以下来源中按顺序拉起：service worker、HTTP缓存、HTTP/2 缓存、目标服务器。
**预加载资源和页面一起存储**(或worker)。这使得它是浏览器检查的第一缓存（在service worker和HTTP缓存之前），而且丢失一个连接也不会丢失你的预加载项。直接链接到页面也表示，如果预加载项没有被使用，devtool可以显示一个有用的警告。

每个页面都有它自己的预加载资源，所以预加载资源给其他页面是没有意义的。类似地，你不能在页面加载之后预加载资源来使用。从页面中预加载资源用于service worker的安装也是没有意义的，service worker不会检查页面的预加载缓存。

![在这里插入图片描述](https://img-blog.csdnimg.cn/2020010815001725.png)

**Chrome**不是所有API都支持预加载。例如，`fetch()`不使用预加载缓存。`XHR`仅在携带凭据发送的情况下，会使用预加载缓存。[(issue)](https://bugs.chromium.org/p/chromium/issues/detail?id=652228)

**Safari**在它最新的技术预览版本支持预加载。`fetch()`不使用预加载缓存，`XHR`也不使用。[(issue)](https://bugs.webkit.org/show_bug.cgi?id=158720)

**Firefox**不支持预加载，但已经提上日程了。[(issue)](https://bugzilla.mozilla.org/show_bug.cgi?id=1222633)

**Edge**不支持预加载。[如果你想要的话就投票吧。](https://wpdev.uservoice.com/forums/257854-microsoft-edge-developer/suggestions/7666725-preload)

### 建议
完美的预加载总是比完美的HTTP/2 推送慢那么一丢丢的，因为后者不需要等浏览器发送请求。然而，预加载异常地简单而且更容易排错。我建议今天使用它，因为浏览器的支持会越来越好——但确保关注devtools来确保你推送的项被使用了。

某些服务器把预加载的头换成了HTTP/2 推送。基于他们有这么多细微的差别，我认为这是一个错误，但我们可能还要忍受一些时间。但是，你应该确保这些服务在最终响应中删除了这些头，否则的话可能会发生竞速的情况，即在推送之前预加载了，导致了双倍的带宽损耗。

## 未来的推送
现在，HTTP/2 推送有很多粗糙的bug，但一旦他们被解决，我认为它对我们当前内联各种资源是非常理想的，尤其是关键CSS。一旦缓存摘要落地，就有望从内联中获益，也从缓存中获益。

推送能否正常运行，取决于服务器是否允许我们正确地确定内容流的优先级。例如，我想随页面头部一起加载我的关键CSS，于是就给这个CSS以完全的优先级，因为此时用户还不能渲染，花费带宽在主体上是一种浪费。

如果你的服务器响应有点慢（因为开销大的数据库查找或是别的什么），你可以用这段时间来推送可能需要的页面资源，然后在页面可用时改变优先级。

如我所述，写这篇文章不是信手拈来，希望它不会是过眼云烟。HTTP/2 推送确实可以提升性能，只是不要没经过谨慎的测试就使用它，否则可能会更慢。
