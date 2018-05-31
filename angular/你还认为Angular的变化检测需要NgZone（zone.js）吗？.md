深入了解NgZone的实现及其用法
![](https://cdn-images-1.medium.com/max/800/1*ATHYiYJ5Ae8wGrJaqKbuAQ.jpeg)
> 请注意，这篇文章不是关于Zones（zone.js），而是关于Angular如何使用Zones来实现NgZone及其与变化检测机制的关系。要了解有关Zones的更多信息，请阅读[I reverse-engineered Zones (zone.js) and here is what I’ve found](https://blog.angularindepth.com/i-reverse-engineered-zones-zone-js-and-here-is-what-ive-found-1f48dc87659b)

我看到的大多数文章都强烈地将`Zone`(`zone.js`)和`NgZone`与Angular的变更检测联系在一起。虽然它们确实有关，但从技术上讲它们不是整体的一部分。是的，`Zone`和`NgZone`用于自动地触发由于异步操作而引起的变化检测。但由于变化检测是一个独立的机制，它可以在没有`Zone`和`NgZone`下运行。在第一章中，我将介绍如何在没有`zone.js`的情况下使用Angular 。文章的第二部分解释了Angular和zone.js如何通过`NgZone`相互作用。最后，我还将介绍为什么自动变更检测有时不适用于[Google API Client Library](https://developers.google.com/api-client-library/)（gapi）等第三方库。

我已经写了很多关于Angular变化检测的深度文章，并且这篇文章完成了这个图片。如果你正在寻找一个关于如何改变检测工作的全面的概述，我建议从这个[These 5 articles will make you an Angular Change Detection expert.](https://blog.angularindepth.com/these-5-articles-will-make-you-an-angular-change-detection-expert-ed530d28930)开始阅读。

使用没有Zone (zone.js)的Angular
==============
为了证明Angular可以在没有Zone的情况下成功工作，我最初计划提供一个模拟zone对象，它根本不会做任何事情。但即将推出的Angular版本5为我简化了一些事情。现在它提供了一种使用[Noop Zone](https://github.com/angular/angular/blob/30d5a2ca83c9cf44f602462597a58547b05b75dd/packages/core/src/zone/ng_zone.ts#L318)的[方法](https://github.com/angular/angular/commit/344a5ca)，该方法不会通过configuration进行任何操作。

要做到这一点，我们首先删除依赖`zone.js`。我将使用stackblitz来演示应用程序，因为它使用Angular-CLI，我将从`polyfils.ts`文件中删除以下导入：
```
* Zone JS is required by Angular itself. */
import 'zone.js/dist/zone';  // Included with Angular CLI.
```
v6版本在后面添加
```
...
(window as any)['Zone'] = {
    current: {
      get() {}
    }
  };
```

之后，我将配置Angular以使用noop Zone实现，如下所示：
```
platformBrowserDynamic()
    .bootstrapModule(AppModule, {
        ngZone: 'noop'
    });
```
如果您现在[运行应用程序](https://stackblitz.com/edit/angular-jmlwb7)，您将看到变更检测完全可操作并在DOM中呈现`name`组件属性。

现在，如果我们使用`setTimeout`方式更新属性:
```
export class AppComponent  {
    name = 'Angular 4';

    constructor() {
        setTimeout(() => {
            this.name = 'updated';
        }, 1000);
    }
```
您可以看到变化没有更新。这是预料到的，因为`NgZone`没有使用，因此变化检测不会自动触发。但如果我们手动触发，它仍然可以正常工作。这可以通过注入[ApplicationRef](https://angular.io/api/core/ApplicationRef)和触发`tick`方法来启动变更检测：
```
export class AppComponent  {
    name = 'Angular 4';

    constructor(app: ApplicationRef) {
        setTimeout(()=>{
            this.name = 'updated';
            app.tick();
        }, 1000);
    }
```
现在您可以看到[update is successfully rendered](https://stackblitz.com/edit/angular-lr1rss)。

总而言之，上述演示的目的是向您说明`zone.js`和`NgZone`并不是变更检测实现的一部分。这是一个**非常方便**的机制，通过调用`app.tick()`自动触发变更检测，而不是在某些点手动执行。我们将马上看到那些点。

NgZone如何使用Zones
==========
在我[之前关于Zone（zone.js）](https://blog.angularindepth.com/i-reverse-engineered-zones-zone-js-and-here-is-what-ive-found-1f48dc87659b)的文章中，我深入介绍了Zone provides的内部工作和API。在那里我解释了[forking a zone](https://blog.angularindepth.com/i-reverse-engineered-zones-zone-js-and-here-is-what-ive-found-1f48dc87659b#5306)和[running a task in a particular zone](https://blog.angularindepth.com/i-reverse-engineered-zones-zone-js-and-here-is-what-ive-found-1f48dc87659b#13d4)的核心概念。我将在这里提到这些概念。

我还演示了Zone provides的两种功能 - [context propagation](https://blog.angularindepth.com/i-reverse-engineered-zones-zone-js-and-here-is-what-ive-found-1f48dc87659b#bfdd)和出色的[异步任务跟踪](https://blog.angularindepth.com/i-reverse-engineered-zones-zone-js-and-here-is-what-ive-found-1f48dc87659b#4320)。Angular实现了很大程度上依赖于任务跟踪机制的[NgZone](https://github.com/angular/angular/blob/30d5a2ca83c9cf44f602462597a58547b05b75dd/packages/core/src/zone/ng_zone.ts#L86)类。

NgZone只是一个[forked child zone](https://github.com/angular/angular/blob/30d5a2ca83c9cf44f602462597a58547b05b75dd/packages/core/src/zone/ng_zone.ts#L252)的包装：
```
function forkInnerZoneWithAngularBehavior(zone: NgZonePrivate) {
    zone._inner = zone._inner.fork({
        name: 'angular',
        ...
```
forked zone保留在`_inner`属性中，通常称为Angular zone。这是在执行`NgZone.run()`时用于执行callback 的zone：
```
run(fn, applyThis, applyArgs) {
    return this._inner.run(fn, applyThis, applyArgs);
}
```

forking Angular zone时的当前zone保留在`_outer`属性中，并在执行`NgZone.runOutsideAngular()`时用于执行callback：
```
runOutsideAngular(fn) {
    return this._outer.run(fn);
}
```
此方法通常用于在Angular zone之外运行性能繁重的操作，以避免不断触发变更检测。

NgZone拥有的`isStable`属性表明是否没有未完成的micro或macro任务。它还定义了四个事件：
```
+------------------+-----------------------------------------------+
|      Event       |                     Description               |
+------------------+-----------------------------------------------+
| onUnstable       | Notifies when code enters Angular Zone.       |
|                  | This gets fired first on VM Turn.             |
|                  |                                               |
| onMicrotaskEmpty | Notifies when there is no more microtasks     |
|                  | enqueued in the current VM Turn.              |
|                  | This is a hint for Angular to do change       |
|                  | detection which may enqueue more microtasks.  |
|                  | For this reason this event can fire multiple  |
|                  | times per VM Turn.                            |
|                  |                                               |
| onStable         | Notifies when the last `onMicrotaskEmpty` has |
|                  | run and there are no more microtasks, which   |
|                  | implies we are about to relinquish VM turn.   |
|                  | This event gets called just once.             |
|                  |                                               |
| onError          | Notifies that an error has been delivered.    |
+------------------+-----------------------------------------------+
```
Angular使用[ApplicationRef](https://github.com/angular/angular/blob/30d5a2ca83c9cf44f602462597a58547b05b75dd/packages/core/src/application_ref.ts#L364)中的`onMicrotaskEmpty`事件来自动触发更改检测：
```
this._zone.onMicrotaskEmpty.subscribe(
    {next: () => { this._zone.run(() => { this.tick(); }); }});

```
从上一节你该记得`tick()`方法是用于运行应用程序的变更检测。

NgZone如何实现onMicrotaskEmpty事件
=========
现在让我们看看如何`NgZone`实现`onMicrotaskEmpty`事件。该事件从[checkStable](https://github.com/angular/angular/blob/30d5a2ca83c9cf44f602462597a58547b05b75dd/packages/core/src/zone/ng_zone.ts#L233)函数发出：
```
function checkStable(zone: NgZonePrivate) {
  if (zone._nesting == 0 && !zone.hasPendingMicrotasks && !zone.isStable) {
    try {
      zone._nesting++;
      zone.onMicrotaskEmpty.emit(null); <-------------------
```
这个功能通常由三个Zone钩子触发：
+ [onHasTask](https://blog.angularindepth.com/i-reverse-engineered-zones-zone-js-and-here-is-what-ive-found-1f48dc87659b#4320)
+ [onInvokeTask](https://blog.angularindepth.com/i-reverse-engineered-zones-zone-js-and-here-is-what-ive-found-1f48dc87659b#4320#4bea)
+ [onInvoke](https://blog.angularindepth.com/i-reverse-engineered-zones-zone-js-and-here-is-what-ive-found-1f48dc87659b#4320#4bea#1834)

正如在有关Zones的文章中解释的那样，当最后两个钩子被触发时，microtask队列中可能会发生变化，所以Angular 每次触发钩子时都必须运行`stable`检查。该`onHasTask`钩子也用于执行检查，因为它跟踪整个队列更改。

常见的陷阱
==========
与变化检测相关的stackoveflow上最常见的问题之一是为什么有时使用第三方库时，组件中的更改不起作用。以下是涉及[Google API Client Library](https://developers.google.com/api-client-library/)（gapi）的[此类问题的示例](https://stackoverflow.com/a/46286400/2545680)。这种问题的常见解决方案是简单地在Angular区域内运行回调，如下所示：
```
gapi.load('auth2', () => {
    zone.run(() => {
        ...
```
但是，一个有趣的问题是Zone 为什么**没有**注册导致**没有**通知的请求？并且没通知,`NgZone`不会自动触发更改检测。

要了解我只要挖掘`gapi` **minified**的来源，并发现它使用[JSONP](http://schock.net/articles/2013/02/05/how-jsonp-really-works-examples/)来提出网络请求。这种方法不使用常见的AJAX API，如[XMLHttpRequest](https://developer.mozilla.org/en-US/docs/Web/API/XMLHttpRequest)或[Fetch API](https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API)，它们由Zones修补和跟踪。相反，它创建一个带有源URL 的`script`标签，并定义一个全局回调，当从服务器获取所请求的带有数据的script时，触发该回调。这不能被Zones修补或检测到，因此这框架仍然没有察觉到使用此技术执行的请求。

下面是来自 [gapi minimized version](https://apis.google.com/js/api.js)的相关信息：
```
Ja = function(a) {
    var b = L.createElement(Z);
    b.setAttribute(“src”, a);
    a = Ia();
    null !== a && b.setAttribute(“nonce”, a);
    b.async = “true”;
    (a = L.getElementsByTagName(Z)[0]) ? 
        a.parentNode.insertBefore(b, a) : 
        (L.head || L.body || L.documentElement).appendChild(b)
}
```
该`Z`变量等于`"script"`并且该参数`a`保存了请求URL：
```
https://apis.google.com/_.../cb=gapi.loaded_0
```
URL的最后一部分是`gapi.loaded_0`全局回调：
```
typeof gapi.loaded_0 
“function”
```