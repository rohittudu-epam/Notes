# Selenium with Java — Scenario-Based Interview Questions

---

## 1. Architecture & Internals

**Q1. You write a simple `driver.findElement(By.id("btn")).click()` — walk me through every step that happens under the hood from your Java code to the browser actually clicking.**

The Java client library serializes the `findElement` call into an HTTP POST request to the driver server (e.g., `POST /session/{id}/element` with payload `{"using":"css selector","value":"#btn"}`). The driver server (ChromeDriver/GeckoDriver) receives this, translates it into browser-native automation commands, locates the element in the DOM, and returns the element's internal reference ID. The client then sends a second HTTP POST (`/session/{id}/element/{elementId}/click`). The driver scrolls the element into view, calculates its center point, verifies interactability, performs the click, and returns an empty success response. The Java client unblocks and execution continues.

Key points: two separate HTTP roundtrips, W3C WebDriver Protocol, synchronous request-response, driver is a standalone HTTP server.

---

**Q2. Your test works perfectly with Selenium 3 but after upgrading to Selenium 4, the `moveToElement()` hover action is clicking the wrong area. What changed?**

Selenium 4 uses the W3C WebDriver Protocol exclusively. In W3C mode, `moveToElement()` moves the cursor to the **center** of the element. Selenium 3 (JSON Wire Protocol) moved to the **top-left corner**. If the test had offset calculations based on the old behavior, they're now off. Fix by adjusting or removing offsets, and testing with the new center-based origin.

Also check: `DesiredCapabilities` → must use browser `Options` classes; unprefixed custom capabilities are rejected; action chain behavior differences.

---

**Q3. You're setting a custom capability `testEnvironment: staging` on Selenium Grid 4 but sessions keep going to the wrong nodes. What's wrong?**

W3C WebDriver Protocol requires all custom capabilities to use a vendor prefix with `:` (e.g., `custom:testEnvironment`). Unprefixed capabilities are silently ignored or rejected by Grid 4. The session request falls back to matching only standard capabilities, which may match any node. Fix by renaming to `custom:testEnvironment` and ensuring nodes advertise the same prefixed capability.

---

## 2. Browser Setup & Configuration

**Q4. Your Selenium tests run fine locally but Chrome keeps crashing inside Docker containers. What do you investigate?**

Primary cause: Chrome uses `/dev/shm` (shared memory) for rendering, and Docker's default is 64MB — too small. Fix:
- Add `--shm-size=2g` to the Docker run command, OR
- Add `--disable-dev-shm-usage` to `ChromeOptions` (forces Chrome to use `/tmp` instead)

Also check: `--no-sandbox` is needed if running as root; `--headless=new` for headless execution; ensure memory/CPU limits on the container aren't too restrictive.

---

**Q5. You need to test file downloads in Chrome, but the browser keeps showing a "Save As" dialog. How do you handle it?**

Configure Chrome preferences via `ChromeOptions`:
```java
Map<String, Object> prefs = new HashMap<>();
prefs.put("download.default_directory", "/path/to/downloads");
prefs.put("download.prompt_for_download", false);
prefs.put("download.directory_upgrade", true);
options.setExperimentalOption("prefs", prefs);
```
To verify the download, poll the directory for the expected file, check it doesn't have a `.crdownload` extension (still downloading), and verify file size > 0.

Hidden nuance: on Selenium Grid, the file downloads to the **remote node**, not the client machine. Use `se:downloadsEnabled` capability and the Grid download API to retrieve the file.

---

**Q6. Your team runs tests on Chrome and Firefox in parallel. Headless mode works on Chrome with `--headless=new` but fails silently on Firefox. What's different?**

Firefox uses `-headless` (single dash, no `=new`), not `--headless=new`. The argument syntax is browser-specific. Additionally, Firefox preferences for downloads require both `browser.download.folderList` (set to 2 for custom directory) AND `browser.download.dir` plus MIME types in `browser.helperApps.neverAsk.saveToDisk`. These are `about:config` key-value pairs, not command-line arguments like Chrome.

Abstract browser-specific differences into a driver factory to avoid scattering browser-specific logic across tests.

---

**Q7. Tests using `PageLoadStrategy.NORMAL` take too long because pages have heavy analytics scripts and large images. What's the trade-off with other strategies?**

- `EAGER` — waits for `DOMContentLoaded` only (DOM is parsed and interactive, but images/scripts may still load). Best for SPAs where you interact with DOM elements, not images.
- `NONE` — returns immediately after navigation starts. Fastest, but you must manually wait for readiness (poll `document.readyState` or wait for specific elements).

The page load strategy affects ALL navigation: `driver.get()`, `navigate().to()`, link clicks, form submissions. Switch to `EAGER` and add explicit waits for the specific elements your tests need.

---

## 3. Locators

**Q8. You need to locate an element whose `id` is auto-generated (e.g., `input_x7k2p`) and changes on every deployment. How do you write a stable locator?**

Options in priority order:
1. Ask developers to add a `data-testid` or `data-qa` attribute — stable, test-specific, and a standard practice.
2. Use a CSS selector with a stable parent + tag + attribute: `form.login-form input[type='email']`
3. Use XPath with text or position relative to a stable element: `//label[text()='Email']/following-sibling::input`
4. Use Selenium 4 Relative Locators: `RelativeLocator.with(By.tagName("input")).below(labelElement)`

Avoid: the auto-generated ID, index-dependent XPath, or any locator that relies on build-specific class names (CSS modules, Tailwind JIT hashes like `btn-abc12`).

---

**Q9. Your CSS selector `.item:nth-child(2)` selects the wrong element. The HTML has `<div class="item">`, `<span>`, `<div class="item">`. Why?**

`:nth-child(2)` counts ALL sibling elements regardless of tag. The second child overall is the `<span>`, not the second `.item`. Since `<span>` doesn't have class `item`, the selector matches nothing or matches unexpectedly.

Fix: use `:nth-of-type(2)` if all target elements share the same tag, or use a more explicit selector like `div.item:nth-of-type(2)`. Alternatively, use `(//div[@class='item'])[2]` in XPath for positional selection.

Also know: CSS can't select by text content (no `:contains()`), can't traverse upward (no parent selector). Use XPath when you need these capabilities.

---

**Q10. You need to click a button inside a Shadow DOM. `findElement(By.cssSelector("#shadow-button"))` returns `NoSuchElementException` even though the element is visible on the page. Why?**

Shadow DOM creates an encapsulated DOM subtree. Standard `findElement` searches only the light DOM; it cannot pierce shadow boundaries.

Fix for open shadow roots (Selenium 4.1+):
```java
WebElement host = driver.findElement(By.cssSelector("my-component"));
SearchContext shadow = host.getShadowRoot();
WebElement button = shadow.findElement(By.cssSelector("#shadow-button"));
button.click();
```

Key constraints: only CSS selectors work inside shadow roots (not XPath); nested shadow DOMs require traversing each level; closed shadow roots are inaccessible via `getShadowRoot()`.

---

**Q11. Your XPath `//div[text()='John Doe']` isn't matching, but you can clearly see "John Doe" on the page. What's happening?**

Several possibilities:
1. **Whitespace**: The actual text has `\n`, `\t`, or extra spaces: `"  John Doe  "`. Fix: use `//div[normalize-space(text())='John Doe']` or `//div[normalize-space()='John Doe']`.
2. **Nested text**: The text is split across child elements: `<div><span>John</span> Doe</div>`. `text()` only returns direct text nodes. Fix: use `.` (dot) instead: `//div[normalize-space(.)='John Doe']`.
3. **Wrong frame**: The element is inside an iframe. Switch to the frame first.

Also: `text()='value'` is exact match — even an extra trailing space breaks it. `normalize-space()` is almost always safer.

---

**Q12. You use `By.className("btn primary")` and get an `InvalidSelectorException`. Why?**

`By.className()` accepts only a **single** class name. The string `"btn primary"` contains a space, which is interpreted as two classes — invalid for this method.

Fix: use a CSS selector `.btn.primary` (matches elements with BOTH classes) or XPath `//button[contains(@class,'btn') and contains(@class,'primary')]`. Note the XPath approach can over-match (e.g., `class="btn-primary"` also matches `contains(@class,'btn')`).

---

## 4. Element Interactions

**Q13. Your test calls `sendKeys("test@email.com")` on a React input field. The text appears in the browser, but the React component's state doesn't update and form submission fails. Why?**

React uses controlled components where state is managed by React, not the DOM. `sendKeys()` dispatches native keyboard events which React may process, but sometimes React's synthetic event system doesn't fully pick up Selenium's events — especially if the input has custom event handlers or debouncing.

If this happens:
1. First try `clear()` then `sendKeys()` — ensure the field starts empty.
2. If that fails, use JavaScript to set the value AND dispatch the proper events:
```java
js.executeScript(
    "var el = arguments[0]; el.value = arguments[1]; " +
    "el.dispatchEvent(new Event('input', {bubbles: true})); " +
    "el.dispatchEvent(new Event('change', {bubbles: true}));",
    inputElement, "test@email.com");
```

The key insight: setting `.value` via JS alone does NOT fire `input`/`change` events — you must dispatch them manually for React/Angular to detect the change.

---

**Q14. `getAttribute("value")` and `getDomAttribute("value")` return different results on an input field. Which one should you use?**

In Selenium 4:
- `getAttribute("value")` → returns the **IDL property** (current JavaScript property value — what the user sees now)
- `getDomAttribute("value")` → returns the **HTML attribute** (the initial value from the markup)
- `getDomProperty("value")` → also returns the current JS property (same as `getAttribute` in Selenium 4)

**Use `getAttribute("value")` or `getDomProperty("value")` to check what's currently in the field.** Use `getDomAttribute("value")` to check the initial/default value.

Example: `<input value="default">` → user types "hello" → `getAttribute("value")` returns `"hello"`, `getDomAttribute("value")` returns `"default"`.

---

**Q15. Your test clicks a button, but gets `ElementClickInterceptedException`. The button is visible and enabled. What's happening?**

Another element is covering the button's center point — common culprits:
1. Cookie consent banners (fixed at bottom)
2. Sticky navigation headers (fixed at top)
3. Loading overlays/spinners
4. Toast notifications
5. A transparent overlay `<div>` with higher z-index

Debugging steps:
- Take a screenshot to see what's visually on top
- Inspect the error message — it tells you which element intercepted the click
- Use `executeScript("arguments[0].getBoundingClientRect()", element)` to check position

Fixes:
- Wait for the overlay to disappear: `wait.until(invisibilityOfElementLocated(By.cssSelector(".overlay")))`
- Close/dismiss the cookie banner first
- Scroll extra pixels so the element isn't behind a sticky header
- As a last resort, use `JavascriptExecutor` click — but this masks real UI bugs

---

**Q16. You find an element, store it in a variable, navigate away, come back, and try to use it. It throws `StaleElementReferenceException`. Why?**

When you navigate away, the page's DOM is destroyed. Even though you navigated back to the same URL and the element exists again with the same attributes, the **WebElement reference points to the original DOM node** which no longer exists. The reference is now "stale."

Rule: after any page navigation, AJAX update, or SPA re-render, stored `WebElement` references are invalid. Store the `By` locator and re-find the element each time:

```java
By loginButton = By.id("login");
// ... navigate away and back ...
driver.findElement(loginButton).click(); // fresh reference
```

This is the #1 cause of flaky tests in SPAs (React/Angular/Vue), where components re-render frequently even when the UI looks unchanged.

---

**Q17. `getText()` returns an empty string for an `<input>` field that clearly has text in it. Why?**

`getText()` returns the **visible rendered text content** of an element — the text between the opening and closing tags. `<input>` elements don't have inner text; their displayed value is in the `value` property.

Use `element.getAttribute("value")` or `element.getDomProperty("value")` to read input field content.

Similarly, `getText()` returns `""` for hidden elements (`display:none`, `visibility:hidden`) even if they have text in the DOM.

---

## 5. Waits & Synchronization

**Q18. You set implicit wait to 10 seconds and explicit wait to 15 seconds. A test that should timeout in 15 seconds is taking over 2 minutes. What's wrong?**

You're mixing implicit and explicit waits — a well-known anti-pattern.

The explicit wait polls the condition every 500ms (default). Each poll internally calls `findElement`, which inherits the 10-second implicit wait. If the element doesn't exist, each polling attempt blocks for 10 seconds before returning failure. Over a 15-second explicit wait window, that's potentially 10s × 30 polls = 300 seconds worst case.

Fix: set implicit wait to 0 and use **only** explicit waits:
```java
driver.manage().timeouts().implicitlyWait(Duration.ofSeconds(0));
WebDriverWait wait = new WebDriverWait(driver, Duration.ofSeconds(15));
wait.until(ExpectedConditions.visibilityOfElementLocated(By.id("element")));
```

The official Selenium docs explicitly warn against mixing the two.

---

**Q19. You use `ExpectedConditions.presenceOfElementLocated()` and the wait succeeds, but the subsequent `click()` throws `ElementNotInteractableException`. Why?**

`presenceOfElementLocated` only checks that the element **exists in the DOM** — it can be hidden (`display:none`), zero-size, or off-screen. It does not verify visibility or interactability.

Use the right condition for your action:
| Action | Expected Condition |
|--------|-------------------|
| Read attribute / check existence | `presenceOfElementLocated` |
| Read text / verify visibility | `visibilityOfElementLocated` |
| Click or type | `elementToBeClickable` |
| Wait for disappearance | `invisibilityOfElementLocated` |

Note: even `elementToBeClickable` only checks `isDisplayed() && isEnabled()` — it does NOT check if the element is **obscured** by another element. You can still get `ElementClickInterceptedException`.

---

**Q20. Your `FluentWait` keeps throwing `StaleElementReferenceException` during polling instead of retrying. How do you fix it?**

By default, `FluentWait` only ignores `NoSuchElementException`. If the element goes stale during a poll cycle, the exception propagates immediately.

Fix: add `StaleElementReferenceException` to the ignored exceptions:
```java
new FluentWait<>(driver)
    .withTimeout(Duration.ofSeconds(15))
    .pollingEvery(Duration.ofMillis(500))
    .ignoring(NoSuchElementException.class)
    .ignoring(StaleElementReferenceException.class)
    .until(driver -> driver.findElement(By.id("dynamic")).getText().contains("Loaded"));
```

The `ignoring()` clause tells the wait to swallow that exception during polling and retry. Without it, any stale reference during a DOM re-render causes an immediate failure.

---

**Q21. You write a custom wait condition: `wait.until(driver -> driver.findElements(By.cssSelector(".item")).size() == 5)`. It returns `false` when there are 0 elements, but it should keep waiting. However, with implicit wait set to 10s, each poll takes 10 seconds. Why?**

`findElements()` with an implicit wait > 0 waits the full implicit duration before returning an empty list — on every single poll cycle. The explicit wait polls every 500ms, but each poll blocks for 10 seconds because of the implicit wait applied to `findElements`.

This is another manifestation of the implicit + explicit wait conflict. With implicit = 0, `findElements()` returns immediately with whatever it finds (including an empty list), allowing the explicit wait to poll efficiently.

---

## 6. Windows, Frames & Alerts

**Q22. Your test clicks a link that opens a new tab. You call `driver.switchTo().window(handles.toArray()[1])` and it works in Chrome but selects the wrong tab in Firefox. Why?**

`getWindowHandles()` returns a `Set<String>`. The iteration order of a `Set` is **not guaranteed** across browsers or runs. `toArray()[1]` assuming the second element is the "new" tab is unreliable.

Correct pattern:
```java
String originalHandle = driver.getWindowHandle();
// click link that opens new tab
Set<String> allHandles = driver.getWindowHandles();
for (String handle : allHandles) {
    if (!handle.equals(originalHandle)) {
        driver.switchTo().window(handle);
        break;
    }
}
```

For even more safety, store handles before the click, then diff: `newHandles.removeAll(oldHandles)`.

---

**Q23. After closing a popup window with `driver.close()`, the next command throws `NoSuchWindowException`. What did you miss?**

`driver.close()` closes the **current** window only. After closing, the driver's window context is undefined — it doesn't automatically switch back to any other window. You must explicitly switch:

```java
driver.close();                            // closes popup
driver.switchTo().window(originalHandle);   // switch back
```

Additionally: if you `close()` the last remaining window without calling `quit()`, the WebDriver process stays alive as an orphan. Always use `quit()` to end the session.

---

**Q24. Your test needs to interact with an element inside a nested iframe (iframe inside iframe). `findElement` can't locate it even though you switched to the outer iframe. What's the approach?**

You must switch into each iframe level sequentially:
```java
driver.switchTo().frame("outerFrame");      // enter outer iframe
driver.switchTo().frame("innerFrame");      // enter inner iframe
driver.findElement(By.id("target")).click(); // now accessible
```

To go back:
- `driver.switchTo().parentFrame()` — goes up one level
- `driver.switchTo().defaultContent()` — jumps all the way to the top-level document

Key insight: element locators are always scoped to the **current frame context**. An element inside an iframe is completely invisible from the parent frame's perspective — even if it appears visible on screen.

Use `ExpectedConditions.frameToBeAvailableAndSwitchToIt(By)` to combine waiting for the frame and switching in one step.

---

**Q25. Your test encounters a modal popup. You try `driver.switchTo().alert()` but get `NoAlertPresentException`. The popup is clearly visible. What's wrong?**

It's likely a **custom HTML modal** (Bootstrap, Material UI, sweet alert), not a native browser alert. Only JavaScript `alert()`, `confirm()`, and `prompt()` create true browser dialogs that Selenium's `switchTo().alert()` can handle.

Custom modals are regular HTML elements — interact with them using standard `findElement` + `click`:
```java
wait.until(visibilityOfElementLocated(By.cssSelector(".modal")));
driver.findElement(By.cssSelector(".modal .btn-confirm")).click();
```

How to tell the difference: native alerts block all browser interaction and have a plain OS-style appearance. Custom modals are styled HTML with inspect-able DOM elements.

---

## 7. Actions API

**Q26. Your drag-and-drop test using `actions.dragAndDrop(source, target)` doesn't work on a Trello-like board that uses HTML5 Drag and Drop. The element doesn't move. Why?**

HTML5 Drag and Drop API uses specific events (`dragstart`, `dragenter`, `dragover`, `drop`, `dragend`) that Selenium's `Actions.dragAndDrop()` doesn't fully trigger. The Actions API simulates mouse-level events which don't map 1:1 to the HTML5 DnD event sequence.

Workarounds:
1. Use an explicit Actions chain: `clickAndHold(source).pause(Duration.ofMillis(500)).moveToElement(target).pause(Duration.ofMillis(500)).release().perform()`
2. Use a JavaScript-based drag-and-drop helper that dispatches the correct HTML5 DnD events.
3. Ask developers for a non-DnD fallback or test API.

---

**Q27. Your hover test works locally but fails in headless Chrome — the tooltip never appears. What's different?**

In headless mode, there's no physical cursor. `moveToElement()` still dispatches mouse events, but some CSS rules (`:hover` pseudo-class) or JavaScript hover handlers may behave differently without a real display.

Fixes:
- Verify the window size in headless mode — the default may be too small: `addArguments("--window-size=1920,1080")`
- Add `Actions.pause()` after `moveToElement()` to give the tooltip time to render
- Check if the tooltip relies on CSS `:hover` or JavaScript `mouseenter` — Selenium triggers JS events but CSS hover pseudo-class response can be inconsistent in headless
- Use `--headless=new` (Chrome 109+) which uses the full rendering engine — the old `--headless` had a separate implementation with different behavior

---

**Q28. After performing a keyboard shortcut with `keyDown(Keys.CONTROL).sendKeys("a").perform()`, all subsequent text inputs are Ctrl+modified. Why?**

`keyDown()` presses and **holds** the key. Without a corresponding `keyUp()`, the Control key remains held down. Every subsequent action inherits the pressed modifier.

Fix: always release keys:
```java
actions.keyDown(Keys.CONTROL).sendKeys("a").keyUp(Keys.CONTROL).perform();
```

This also applies to Shift, Alt, and Command keys. A new `Actions(driver)` instance starts with a clean state, so creating a new `Actions` object is another escape hatch.

---

## 8. JavaScript Executor

**Q29. You use `executeScript("arguments[0].click()", element)` to bypass a click intercepted error. The test passes, but in production users report the button doesn't work. What went wrong?**

JavaScript `click()` bypasses Selenium's interactability checks — it doesn't verify visibility, enabled state, or obstruction. By using JS click, you masked a real UI bug: the button was actually covered by an overlay or disabled for users.

The JS click fired the click event handler, making the test pass, but real users with real mouse clicks can't reach the button.

Rule: use `executeScript("click()")` only as a diagnostic tool. If you need it to make a test pass, investigate **why** the regular click fails — that's often the actual bug. Fix the overlay, sticky header, or z-index issue.

---

**Q30. You use `executeScript` to set a value on an Angular input: `js.executeScript("arguments[0].value = 'test'", input)`. The field shows "test" but Angular's form validation still says "required". Why?**

Setting `.value` via JavaScript modifies the DOM property directly but does NOT fire `input`, `change`, or `keyup` events. Angular (and React/Vue) rely on these events to update their internal state and trigger validation.

Fix: dispatch the events after setting the value:
```java
js.executeScript(
    "var el = arguments[0]; " +
    "el.value = arguments[1]; " +
    "el.dispatchEvent(new Event('input', {bubbles: true})); " +
    "el.dispatchEvent(new Event('change', {bubbles: true}));",
    input, "test");
```

Better approach: use `sendKeys()` first (which dispatches all native events). Only fall back to JS when `sendKeys()` genuinely doesn't work.

---

**Q31. You call `executeAsyncScript` but it hangs for 30 seconds and throws `ScriptTimeoutException`. The script logic looks correct. What's likely wrong?**

You forgot to invoke the callback. In `executeAsyncScript`, the last argument is the callback function that signals completion. If you never call it, the script hangs until `scriptTimeout` (default 30s).

Wrong:
```javascript
"setTimeout(function() { return 'done'; }, 1000);"
```
Right:
```javascript
"var callback = arguments[arguments.length - 1]; setTimeout(function() { callback('done'); }, 1000);"
```

The callback pattern is `arguments[arguments.length - 1]` because Selenium appends the callback after any arguments you pass.

---

## 9. UI Components

**Q32. You wrap a custom React Select dropdown with `new Select(element)` and get `UnexpectedTagNameException`. How do you handle it?**

The `Select` class only works on native `<select>` HTML elements. React Select, Material UI Select, Ant Design dropdowns, and similar libraries render as `<div>` structures — not `<select>`.

Interact with them as regular HTML elements:
```java
driver.findElement(By.cssSelector(".react-select__control")).click(); // open dropdown
wait.until(visibilityOfElementLocated(By.cssSelector(".react-select__menu")));
driver.findElement(By.xpath("//div[contains(@class,'react-select__option') and text()='Option A']")).click();
```

Some custom dropdowns use virtual scrolling — not all options are in the DOM at once. You may need to scroll within the dropdown list or type to filter.

---

**Q33. You need to upload a file, but the `<input type="file">` is hidden with `display:none`. `sendKeys()` on it throws an exception. What do you do?**

Make the input visible via JavaScript, then send keys:
```java
WebElement fileInput = driver.findElement(By.cssSelector("input[type='file']"));
js.executeScript("arguments[0].style.display = 'block'; arguments[0].style.visibility = 'visible';", fileInput);
fileInput.sendKeys("C:\\path\\to\\file.pdf");
```

Alternative: remove the `hidden` attribute: `js.executeScript("arguments[0].removeAttribute('hidden')", fileInput)`.

On Selenium Grid: the file must exist on the client machine, not the Grid node. Set a `LocalFileDetector`:
```java
((RemoteWebDriver) driver).setFileDetector(new LocalFileDetector());
```
This streams the file from the client to the node before uploading.

---

**Q34. You need to extract data from a dynamic HTML table with 100 rows. Each `findElement` call takes time because of network roundtrips. How do you optimize?**

Each Selenium command is an HTTP request. Reading 100 rows × 5 columns = 500+ `findElement` calls = significant overhead, especially on Grid.

Optimize with a single `executeScript` call:
```java
List<Map<String, Object>> data = (List<Map<String, Object>>) js.executeScript(
    "var rows = document.querySelectorAll('table tbody tr');" +
    "var headers = [...document.querySelectorAll('table thead th')].map(h => h.textContent.trim());" +
    "return [...rows].map(row => {" +
    "  var cells = row.querySelectorAll('td');" +
    "  var obj = {};" +
    "  headers.forEach((h, i) => obj[h] = cells[i]?.textContent.trim() || '');" +
    "  return obj;" +
    "});"
);
```

One HTTP roundtrip instead of 500+. Also: if the table is paginated or lazy-loaded, only the visible rows are in the DOM — you must click "next page" or scroll to load more.

---

## 10. Screenshots & CDP

**Q35. Your screenshot comparison shows differences between local and CI runs even though the UI hasn't changed. What causes this?**

Common causes:
- **Window size**: headless mode may use a different default viewport size. Explicitly set `--window-size=1920,1080`.
- **Font rendering**: different OS / font libraries (Linux CI vs Windows local) render text differently.
- **Timing**: screenshot captured before animations completed — add waits for visual stability.
- **Transient UI**: toast messages, loading spinners, cursor blink captured in the screenshot.
- **DPI / device pixel ratio**: different between local display and CI. Use `Emulation.setDeviceMetricsOverride` via CDP for consistency.
- **Standard TakesScreenshot captures viewport only** — use Firefox's `getFullPageScreenshotAs()` or Chrome CDP for full-page screenshots.

---

**Q36. You want to test that your app handles slow network conditions gracefully. How do you simulate a 3G connection in Selenium?**

Use Chrome DevTools Protocol:
```java
DevTools devTools = ((HasDevTools) driver).getDevTools();
devTools.createSession();
devTools.send(Network.enable(Optional.empty(), Optional.empty(), Optional.empty()));
devTools.send(Network.emulateNetworkConditions(
    false,   // offline
    200,     // latency (ms)
    750000,  // download throughput (bytes/sec) ~750KB/s for 3G
    250000,  // upload throughput (bytes/sec)
    Optional.of(ConnectionType.CELLULAR3G)
));
```

Important: CDP commands only work on Chrome and Edge. For cross-browser network simulation, use a proxy like BrowserMob Proxy. Also: network interception adds overhead — disable it in tests that don't need it.

---

**Q37. You want to capture all JavaScript console errors during test execution. How?**

Multiple approaches:

**CDP approach (Chrome/Edge):**
```java
DevTools devTools = ((HasDevTools) driver).getDevTools();
devTools.createSession();
devTools.getDomains().events().addConsoleListener(event -> {
    if ("error".equals(event.getType())) {
        System.err.println("JS Error: " + event.getMessages());
    }
});
```

**BiDi approach (cross-browser, Selenium 4):**
```java
driver.script().addJavaScriptErrorHandler(error -> {
    System.err.println("JS Error: " + error.getMessage());
});
```

**Legacy approach:**
```java
LogEntries logs = driver.manage().logs().get(LogType.BROWSER);
logs.forEach(entry -> {
    if (entry.getLevel() == Level.SEVERE) {
        System.err.println(entry.getMessage());
    }
});
```

Capture these in your test framework listener and attach to reports — JS errors during tests often indicate bugs.

---

## 11. Selenium Grid

**Q38. A test works locally but hangs on Selenium Grid — it just waits without any error. After a few minutes, it fails with a timeout. What do you check?**

Common Grid-specific issues:
1. **No matching node**: capabilities don't match any available node. Check `/graphql` or `/status` endpoint. Typos in capability names cause silent mismatches.
2. **Session queue timeout**: all nodes are busy. Sessions queue up and timeout if `--session-request-timeout` is exceeded.
3. **Network latency**: each Selenium command is an HTTP roundtrip. On Grid, this adds latency. Implicit waits compound this.
4. **File upload without `LocalFileDetector`**: the file path doesn't exist on the remote node. Set `((RemoteWebDriver) driver).setFileDetector(new LocalFileDetector())`.
5. **Browser crash on node**: check node logs. Chrome often crashes in Docker without `--shm-size=2g` or `--disable-dev-shm-usage`.

Debug: use Grid's `/graphql` endpoint to query active sessions, node status, and queue depth.

---

**Q39. You need to gracefully remove a Grid node for maintenance without killing active test sessions. How?**

Use Grid 4's **drain mode**:
```
POST /se/grid/distributor/node/{nodeId}/drain
```

In drain mode, the node:
- Stops accepting new sessions
- Continues running existing sessions until they complete
- Can be shut down after all sessions finish

This prevents test failures from abrupt node removal. Monitor the node's active session count via `/graphql` before shutting it down.

---

## 12. Cookies, Storage & Sessions

**Q40. Your test adds a cookie to skip login, but `addCookie()` throws an error. The cookie domain is `.example.com` but the current page is `about:blank`. What's wrong?**

Cookies are domain-scoped. You must navigate to the target domain before adding cookies:
```java
driver.get("https://example.com");        // navigate to domain first
driver.manage().addCookie(authCookie);      // now you can set cookies for this domain
driver.get("https://example.com/dashboard"); // navigate to target page
```

Adding cookies on `about:blank` or a different domain fails because the browser has no matching domain context.

Also: `SameSite=None` cookies require the `Secure` flag. Tests running on `http://` (non-HTTPS) may silently fail to set these cookies in modern browsers.

---

**Q41. Your test sets an auth token in `localStorage` to skip login. It works locally but fails on the first run in a fresh browser. Why?**

`localStorage` is origin-scoped (protocol + domain + port). You can only write to it after navigating to the target origin:
```java
driver.get("https://example.com");   // establish the origin context
js.executeScript("window.localStorage.setItem('authToken', 'abc123');");
driver.get("https://example.com/dashboard");  // now the app reads the token
```

On a fresh browser with `about:blank`, there's no origin to attach storage to. Navigate to the domain first, set the token, then navigate to the actual page.

---

## 13. Page Object Model & Page Factory

**Q42. Your Page Factory-based test passes on the first run but throws `StaleElementReferenceException` on retry after a page AJAX update. You're using `@CacheLookup`. What's the issue?**

`@CacheLookup` caches the `WebElement` reference after the first lookup and never re-fetches it. When AJAX updates the DOM, the cached reference becomes stale. Removing `@CacheLookup` allows the lazy proxy to re-locate the element on each interaction.

Rule: only use `@CacheLookup` on **truly static** elements that never change during the page lifecycle (e.g., a page logo, static navigation links). Never use it on elements inside dynamic content areas.

Broader consideration: the Selenium project discourages Page Factory entirely — direct `By` locators + explicit waits give more control and easier debugging than proxy magic.

---

**Q43. A developer on your team puts assertions inside page objects: `loginPage.verifyErrorMessage("Invalid credentials")`. Why is this problematic?**

Page objects should model the page's **structure and actions**, not test logic. Assertions are test concerns — they define what's correct or incorrect, which varies per test scenario.

Problems:
- Different tests may expect different error messages — the page object can't know which is "correct"
- It violates single responsibility — the page object now has two reasons to change (UI changes AND assertion logic changes)
- It reduces reusability — other tests that just need to read the error message can't reuse the method without the assertion

Correct approach:
```java
// Page object — returns data
public String getErrorMessage() {
    return driver.findElement(errorMessageLocator).getText();
}

// Test class — asserts
assertEquals(loginPage.getErrorMessage(), "Invalid credentials");
```

---

## 14. Cross-Browser & Debugging

**Q44. Your test suite passes on Chrome but multiple tests fail on Safari. What Safari-specific limitations should you check?**

Safari (`safaridriver`) has significant limitations:
- **No headless mode** — tests always run headed
- **No CDP support** — no network interception, geolocation mocking, or console capture via DevTools
- **No `executeScript` across cross-origin iframes** — same-origin policy is stricter
- **Limited `Actions` API support** — some complex action chains may behave differently
- **No extensions** — can't install browser extensions
- **macOS only** — `safaridriver` only runs on macOS; can't test on Windows/Linux CI
- **Must enable manually**: Safari → Develop → Allow Remote Automation

For cross-browser test suites, identify Safari-unsupported features and skip or adapt those tests with conditional logic.

---

**Q45. A test passes locally but fails intermittently on CI. It's not a timing issue — the explicit waits are generous. What else could it be?**

Non-timing causes of CI flakiness:
1. **Screen size**: CI default viewport may be smaller. Elements may be off-screen or layout may differ. Set `--window-size=1920,1080` explicitly.
2. **Font/rendering differences**: Linux CI renders fonts differently than Windows/macOS. Visual assertions break.
3. **Resource contention**: CI agents run multiple jobs. Slow CPU/memory causes sluggish rendering that waits don't account for.
4. **DNS/network issues**: external services (CDN, APIs) may be slower or blocked in CI.
5. **Test order dependency**: tests pass individually but fail when run after other tests due to shared state (cookies, localStorage, global variables).
6. **Browser version mismatch**: CI has a different Chrome version than local. Use Selenium Manager or pin versions.
7. **Zombie processes**: previous test runs left orphaned `chromedriver` processes consuming resources.

Debug: add detailed logging, screenshots on failure, browser console log capture, and run the failing test in isolation on CI.

---

**Q46. You need to debug why a Selenium command is failing at the protocol level. How do you see the raw HTTP requests between your code and the driver?**

Enable verbose driver logging:
```java
ChromeDriverService service = new ChromeDriverService.Builder()
    .withVerbose(true)
    .withLogFile(new File("chromedriver.log"))
    .build();
WebDriver driver = new ChromeDriver(service, options);
```

The log shows every HTTP request/response: `POST /session/{id}/element` → `{"using":"css selector","value":"#btn"}` → response with element ID or error.

For GeckoDriver: `System.setProperty("webdriver.gecko.driver.log", "geckodriver.log")`.

On Grid: check the node logs and use the `/graphql` endpoint to query session state. Each component (Router, Distributor, Node) has its own logs.

---

## 15. Selenium 4 Migration

**Q47. After upgrading from Selenium 3 to 4, your event logging breaks. You were using `EventFiringWebDriver` and `WebDriverEventListener` but they don't exist anymore. What's the replacement?**

Selenium 4 replaced the event system:
- Old: `EventFiringWebDriver` wrapper + `WebDriverEventListener` interface
- New: `EventFiringDecorator` + `WebDriverListener` interface

```java
WebDriverListener listener = new WebDriverListener() {
    @Override
    public void beforeClick(WebElement element) {
        System.out.println("About to click: " + element);
    }
    @Override
    public void afterClick(WebElement element) {
        System.out.println("Clicked: " + element);
    }
};
WebDriver decoratedDriver = new EventFiringDecorator<>(listener).decorate(driver);
```

The new API uses the decorator pattern instead of wrapping. `WebDriverListener` has default methods for all events — override only what you need. Available hooks include `beforeFindElement`, `afterGetText`, `beforeNavigateTo`, `afterScreenshot`, etc.

---

**Q48. You're migrating tests to Selenium 4 and `getAttribute("class")` now returns different values than before. What changed?**

In Selenium 4, `getAttribute()` returns the **IDL property** (JavaScript property value), not the HTML attribute. For `class`, the IDL property is `className`, which returns the class list as a normalized string.

In Selenium 3, `getAttribute()` returned the HTML attribute value which could differ in whitespace or format.

If you need the exact HTML attribute as written in the markup, use `getDomAttribute("class")`. For the live JavaScript property, use `getDomProperty("className")` or `getAttribute("class")`.

Most of the time the difference is subtle (extra whitespace), but for attributes like `value` on inputs, the difference is significant: `getAttribute("value")` returns the current typed value, `getDomAttribute("value")` returns the initial default from HTML.

---

## 16. Comprehensive Scenarios

**Q49. You're tasked with verifying a file download through a Selenium Grid setup. The file downloads to the Grid node, not your local machine. How do you handle this?**

Selenium Grid 4 approach:
1. Enable downloads: add `se:downloadsEnabled: true` to capabilities
2. After triggering the download in the test, use the Grid download endpoint:
   ```
   GET /session/{sessionId}/se/files
   ```
   This lists downloaded files. Then retrieve:
   ```
   POST /session/{sessionId}/se/files
   {"name": "report.pdf"}
   ```
   Returns the file content as a zip.

Alternative: set the download directory via browser preferences to a known path on the node, then use a shared volume (Docker volume mount) accessible from the test runner.

For `<input type="file">` uploads on Grid: use `LocalFileDetector` to stream the file from the client:
```java
((RemoteWebDriver) driver).setFileDetector(new LocalFileDetector());
```

---

**Q50. You need to test that a real-time notification appears within 3 seconds of another user's action. How do you test this with Selenium?**

Strategy: use `executeAsyncScript` with a MutationObserver or WebSocket listener to wait for the notification DOM change:

```java
String script =
    "var callback = arguments[arguments.length - 1];" +
    "var observer = new MutationObserver(function(mutations) {" +
    "  var notification = document.querySelector('.notification');" +
    "  if (notification && notification.textContent.includes('New message')) {" +
    "    observer.disconnect();" +
    "    callback(notification.textContent);" +
    "  }" +
    "});" +
    "observer.observe(document.body, {childList: true, subtree: true});";

driver.manage().timeouts().scriptTimeout(Duration.ofSeconds(5));
String result = (String) js.executeAsyncScript(script);
```

Alternative: trigger the other user's action via API (not a second browser), then use an explicit wait for the notification element in the first browser.

This is one HTTP roundtrip that waits server-side (in the browser) rather than polling via multiple Selenium commands.

---

> **Study tip:** For each scenario, practice explaining your debugging thought process out loud — interviewers value systematic troubleshooting over memorized answers.