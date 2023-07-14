---
title: "Playwright 笔记"
date: 2023-07-15T01:35:10+08:00
draft: false
categories:
    - selenium
    - testing
    - crawler
tags:
    - playwright
---

## 简介
最近需要实现网页自动化操作，直接 hack 网站接口不太好搞，于是到处找 UI 操作自动化的解决方案。

一开始我在桌面系统上来解决这个事，最好支持操作的录制和重放。一通搜索发现确实有戏，Mac 系统自带了 `Automator` 可以完成 脚本录制 和 操作重放；Win 系统也有不少很好的桌面软件支持。看来各生态都有这类自动化的方案。

我的笔记本是 Mac ，我简单测试了 `Automator` 算不上完美，不过一般常见也够用。
除了支持基本 UI 交互录制，还开放了部分系统和应用的功能支持，也可以通过脚本拓展。对于个人办公场景，这套工具可以满足大部分场景，且尽可能不用编写代码。不过因为是系统绑定，以后不方便丢到 VPS 执行，也不方便服务化和规模化。

我确认过全任务流程可以调整为全部在浏览器里完成，这不就是 Web 自动化测试嘛。
提到 Web UI 自动化, 那就不得不提一下大名鼎鼎的 `Selenium` 了 ———— 提供了一系列与浏览器交互的接口并能模拟各种用户行为，有着庞大的开源社区生态。但近年来，微软开源的 `Playwright`[^1] 同样支持与浏览器交互和用户行为模拟，且功能更为强大和便于使用。使用示例如下：

1. 启动浏览器录制交互，实时生成交互代码

```shell
# 打开youtube页面，并开始录制操作脚本
playwright codegen https://www.youtube.com
# 支持很多浏览器参数，加载原有Session, 并开始录制
playwright codegen --load-session=cookies.json https://www.youtube.com
```

2. 甚至支持导出测试视频 😂

```python
context = browser.new_context(
    record_video_dir="videos/",
    record_video_size={"width": 640, "height": 480}
)
```

3. 支持加载浏览器插件

```python
    context = await playwright.chromium.launch_persistent_context(
        user_data_dir,
        headless=False,
        args=[
            f"--disable-extensions-except={path_to_extension}",
            f"--load-extension={path_to_extension}",
        ],
    )
```

4. 支持 `pytest` 插件，可以方便的整合测试用例

`pytest --browser webkit --headed`
 
还有更多其他丰富特性，可以说强大到叹为观止。绝对是当前 Web 自动化测试不二之选。

## 使用 Playwright

上手使用很简单，来通过一段示例代码感受下

```python
# modify dom element with playwright (by js)
from playwright.sync_api import sync_playwright

def run(playwright):
    browser = playwright.chromium.launch(headless=False)
    context = browser.new_context()
    page = context.new_page()

    # Navigate to a website
    page.goto("https://example.com")

    # Select an element by its ID and modify its inner text
    page.evaluate("""() => {
        const element = document.getElementById('element-id');
        element.innerText = 'Modified text';
    }""")

    # Take a screenshot of the modified page
    page.screenshot(path="modified_page.png")

    # Close the browser
    browser.close()

with sync_playwright() as playwright:
    run(playwright)
```

playwright 提供了多种抽象实体来操作浏览器，也支持 sync 和 async 两种操作（提供了并发的可能）。

通过例样可以看出 browser, context, page 实体分别封装了不同层级能力的抽象。
其中网站页面的操作对应到 page 上，若需要新页面可通过 `page = context.new_page()` 直接创建；
而 context 均包含独立的 session，每次新建 `browser.new_context()` 都是独立的匿名会话；
而 browser 则是浏览器程序实例，可以自定义多种浏览器属性。

通过 page 对象可以完成页面上的各种元素查找和交互：

```python
    # 定位搜索框并录入 query
    page.locator('#search').fill('query')
    # 处理对话弹框
    page.on("dialog", lambda dialog: dialog.accept())
    # 点击按钮
    page.get_by_role("button").click()
```

若需要通过 js 直接对 dom 读写也可通过 `page.evaluate` 来完成。

值得注意的是，playwright 会在所有交互前执行检查，以此来保障动作可以按照期望的方式进行。
这该如何理解呢？举例来说，`page.click()` 会执行一个点击操作在dom元素上，但在此之前 playwright 会自行确保目标元素 `attached`, `visible`, `stable`, `enabled` 等此类前置属性。只有所有关联的默认检查都通过（超时则抛出异常）才会开始执行交互动作，playwright 称之为 `auto-waiting`, 此举可大幅提高动作执行的成功率。

记得很久以前用 `selenium` 时，渲染或者动作响应 突然无法严格按照预期时序执行，就会导致交互动作失败。
这种问题的调试和解决是很无奈的，大多数时候这种随机是没有最优解的，只能加 sleep 等待。
虽然很久没进行相关的调试了，但我感觉这个特性是很好的解决方案, 免除了开发者不必要的烦恼。

最后，在脚本执行的最后需要（手动或者自动）关闭实例以释放资源，尤其是对于需长期运行的服务。


## 总结

本文从我的一个小需求讲到自动化方案选型，桌面端自动化方案和平台绑定，具有特殊的使用场景(Desktop)。
而 Web 自动化更灵活和轻量，通过封装浏览器运行环境可以实现服务化和规模化。
相比于 selenium，微软的 playwright 更高效和强大，上手简单，且特性丰富，可以很好地满足自动化测试需求。

[^1]: Playwright: https://playwright.dev/python/docs/intro


