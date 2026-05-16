# Selenium WebDriver with Java — Focused Study Plan

---

## Phase 1: Selenium Architecture & Internals

### 1.1 Selenium Ecosystem Overview
- Selenium IDE — record-and-playback browser extension
- Selenium WebDriver — programmatic browser automation
- Selenium Grid — distributed remote execution
- Selenium Manager — built-in driver/browser management (since 4.6+)

### 1.2 WebDriver Architecture Deep Dive
- Client bindings (Java) → W3C WebDriver Protocol (HTTP/JSON) → browser driver executable → browser
- Browser drivers: ChromeDriver, GeckoDriver, msedgedriver, safaridriver
- Each Selenium command = one HTTP request to the driver server
- Driver server starts a local HTTP server, listens for commands, translates to browser-native automation
- **Hidden topic:** W3C WebDriver Protocol vs legacy JSON Wire Protocol — Selenium 4 defaults to W3C; legacy mode removed; behavioral differences in capabilities, actions, error codes
- **Hidden topic:** the driver binary is a standalone HTTP server — understanding this explains why `driver.quit()` kills a process, why timeouts occur on network issues, and why remote execution works the same as local
- **Hidden topic:** session creation lifecycle — `POST /session` with capabilities → capability matching → browser launch → session ID returned → all subsequent commands use that session ID
- **Hidden topic:** command execution model — synchronous HTTP request/response per command; the client blocks until the driver responds or times out

### 1.3 WebDriver Interface Hierarchy
- `WebDriver` → top-level browser control
- `WebElement` → element interaction
- `SearchContext` → parent of both `WebDriver` and `WebElement` (`findElement`, `findElements`)
- `JavascriptExecutor` → JS execution
- `TakesScreenshot` → screen capture
- `HasCapabilities` → read capabilities
- `HasAuthentication`, `HasDevTools`, `HasCdp` — Selenium 4 interfaces
- **Hidden topic:** `SearchContext` is the shared interface — `WebElement.findElement()` scopes searches to that element's subtree, not the whole DOM
- **Hidden topic:** casting `WebDriver` to sub-interfaces (`(JavascriptExecutor) driver`) — fails at runtime if the driver implementation doesn't support it (rare with standard drivers, common with custom wrappers)

### 1.4 Capabilities & Options System
- `MutableCapabilities`, `AbstractDriverOptions`
- `ChromeOptions`, `FirefoxOptions`, `EdgeOptions`, `SafariOptions` all extend `AbstractDriverOptions`
- Standard W3C capabilities: `browserName`, `browserVersion`, `platformName`, `acceptInsecureCerts`, `pageLoadStrategy`, `proxy`, `timeouts`, `unhandledPromptBehavior`
- Vendor-specific capabilities: `goog:chromeOptions`, `moz:firefoxOptions`, `ms:edgeOptions`
- `merge()` method for combining capabilities
- **Hidden topic:** W3C capability namespacing — custom capabilities MUST use a vendor prefix with `:` (e.g., `cloud:options`); unprefixed custom caps are rejected
- **Hidden topic:** `DesiredCapabilities` is deprecated in Selenium 4 — use browser-specific `Options` classes instead; `DesiredCapabilities` still works but loses type safety
- **Hidden topic:** capability matching on Grid — the Grid matches session requests to nodes based on capabilities; a typo in capability names causes silent mismatches

---

## Phase 2: Browser Setup & Configuration

### 2.1 Driver Management
- Manual driver download and `System.setProperty("webdriver.chrome.driver", path)`
- `WebDriverManager` (Boni Garcia) — auto-detects browser version, downloads matching driver
- Selenium Manager (built-in since 4.6+) — automatic driver AND browser management, no external dependency
- **Hidden topic:** Selenium Manager also manages browser binaries — it can download Chrome for Testing or Firefox if the installed version doesn't match
- **Hidden topic:** Selenium Manager resolution order: check PATH → check default install locations → download if needed; set `SE_MANAGER_LOG=DEBUG` environment variable to see resolution steps
- **Hidden topic:** `URI.create(url).toURL()` instead of deprecated `new URL(String)` when constructing `RemoteWebDriver` URLs (Java 20+)
- **Hidden topic:** classpath conflicts — having both `WebDriverManager` and Selenium Manager active can cause version mismatches if they resolve different driver versions

### 2.2 ChromeDriver & ChromeOptions
- Basic: `new ChromeDriver()`, `new ChromeDriver(options)`
- Arguments: `addArguments("--headless=new")`, `--disable-gpu`, `--no-sandbox`, `--window-size=1920,1080`, `--disable-extensions`, `--incognito`, `--start-maximized`
- Experimental options: `setExperimentalOption("prefs", prefsMap)` — download directory, disable password manager, disable popup blocking
- Extensions: `addExtensions(File extensionCrx)`
- Binary location: `setBinary("/path/to/chrome")` — use a specific Chrome installation
- Exclude switches: `setExperimentalOption("excludeSwitches", List.of("enable-automation"))` — removes "Chrome is being controlled" banner
- **Hidden topic:** `--headless=new` vs old `--headless` — the new headless mode (Chrome 109+) uses the full browser engine; old headless was a separate implementation with different rendering behavior
- **Hidden topic:** `--remote-allow-origins=*` — needed when Chrome and ChromeDriver have CORS-like restrictions on DevTools connections; common in CI environments
- **Hidden topic:** `detach` option — `options.setExperimentalOption("detach", true)` keeps the browser open after `driver.quit()` for manual debugging
- **Hidden topic:** `--disable-dev-shm-usage` — critical for Docker/Linux to avoid Chrome crashes due to small `/dev/shm`
- **Hidden topic:** Chrome profile — `addArguments("--user-data-dir=/path", "--profile-directory=Profile 1")` loads an existing profile with saved cookies, extensions, etc.

### 2.3 GeckoDriver & FirefoxOptions
- Basic: `new FirefoxDriver()`, `new FirefoxDriver(options)`
- Profile management: `FirefoxProfile` with custom preferences
- Preferences: `addPreference("browser.download.folderList", 2)`, `addPreference("browser.download.dir", path)`
- Headless: `addArguments("-headless")`
- Binary: `setBinary(new FirefoxBinary(path))`
- **Hidden topic:** Firefox preferences use a different naming convention than Chrome arguments — they are `about:config` key-value pairs
- **Hidden topic:** GeckoDriver logging — `System.setProperty("webdriver.gecko.driver.log", "geckodriver.log")` for debugging driver-level issues
- **Hidden topic:** Firefox downloads require setting both `browser.download.folderList` and `browser.download.dir`, plus MIME type handling via `browser.helperApps.neverAsk.saveToDisk`

### 2.4 EdgeDriver & EdgeOptions
- Chromium-based Edge — shares most ChromeOptions behaviors
- `new EdgeDriver(options)` with `EdgeOptions`
- Same `addArguments()`, `setExperimentalOption()` as Chrome
- **Hidden topic:** Edge sometimes requires separate `msedgedriver` even though it's Chromium — version pairing matters

### 2.5 SafariDriver
- `new SafariDriver()` — no external driver binary; `safaridriver` ships with macOS
- Must enable: Safari → Develop → Allow Remote Automation
- **Hidden topic:** Safari has no headless mode, no CDP support, no network interception, limited options — test what you can, note the gaps
- **Hidden topic:** Safari Technology Preview — available as a separate driver for testing against upcoming Safari features

### 2.6 Page Load Strategy
- `NORMAL` (default) — waits for `load` event (all resources including images, stylesheets)
- `EAGER` — waits for `DOMContentLoaded` (DOM parsed, but resources may still load)
- `NONE` — returns immediately after navigation starts
- Set via: `options.setPageLoadStrategy(PageLoadStrategy.EAGER)`
- **Hidden topic:** `EAGER` is often the best balance for SPAs — DOM is ready for interaction, no waiting for slow images/analytics
- **Hidden topic:** `NONE` requires manual readiness checks (poll `document.readyState` or wait for specific elements) — fastest but most error-prone
- **Hidden topic:** page load strategy affects `driver.get()`, `navigate().to()`, `click()` on links, form submissions — any navigation action

### 2.7 Timeouts Configuration
- `driver.manage().timeouts().implicitlyWait(Duration.ofSeconds(n))` — element lookup polling
- `driver.manage().timeouts().pageLoadTimeout(Duration.ofSeconds(n))` — max time for page navigation
- `driver.manage().timeouts().scriptTimeout(Duration.ofSeconds(n))` — max time for `executeAsyncScript()`
- Default values: implicit = 0, page load = 300s, script = 30s
- **Hidden topic:** `setScriptTimeout` affects ONLY `executeAsyncScript()`, not `executeScript()` — the sync version has no timeout (it blocks indefinitely if JS hangs)
- **Hidden topic:** setting implicit wait to 0 and using explicit waits everywhere is the recommended pattern

### 2.8 Proxy Configuration
- `Proxy` class: `setHttpProxy()`, `setSslProxy()`, `setNoProxy()`, `setSocksProxy()`
- Assign to options: `options.setProxy(proxy)`
- Proxy types: `MANUAL`, `PAC`, `AUTODETECT`, `SYSTEM`, `DIRECT`
- **Hidden topic:** `SYSTEM` uses the OS proxy settings — useful for corporate environments
- **Hidden topic:** proxy affects all traffic including driver-to-browser communication if misconfigured — use `setNoProxy("localhost,127.0.0.1")` to exclude local addresses

---

## Phase 3: Locator Strategies (Deep Dive)

### 3.1 Basic Locators
- `By.id("elementId")` — fastest, most reliable; IDs should be unique
- `By.name("elementName")` — common for form fields
- `By.className("className")` — matches a single class; fails if the value contains spaces (multiple classes)
- `By.tagName("tag")` — broad, usually returns many elements
- `By.linkText("Exact Link Text")` — only works on `<a>` elements, exact match
- `By.partialLinkText("Partial")` — substring match on `<a>` elements
- **Hidden topic:** `By.className("foo bar")` throws `InvalidSelectorException` — it doesn't support compound classes; use CSS selector `.foo.bar` instead
- **Hidden topic:** `By.id` and `By.name` use `getElementById` / `getElementsByName` internally — if there are duplicate IDs (invalid HTML), behavior is browser-dependent

### 3.2 CSS Selectors
- Element: `div`, `input`, `a`
- ID: `#myId`
- Class: `.myClass`, `.class1.class2` (AND)
- Attribute: `[type='text']`, `[href^='https']` (starts-with), `[href$='.pdf']` (ends-with), `[href*='partial']` (contains), `[data-testid]` (attribute existence)
- Combinators: descendant (space), child (`>`), adjacent sibling (`+`), general sibling (`~`)
- Pseudo-classes: `:first-child`, `:last-child`, `:nth-child(n)`, `:nth-of-type(n)`, `:not(selector)`, `:enabled`, `:disabled`, `:checked`
- Multiple selectors: `selector1, selector2` (OR)
- **Hidden topic:** `:nth-child(n)` vs `:nth-of-type(n)` — `nth-child` counts ALL sibling elements regardless of type; `nth-of-type` counts only siblings of the same tag; mixing them up causes wrong element selection
- **Hidden topic:** CSS cannot select by text content — there is no `:contains()` in standard CSS; use XPath for text-based selection
- **Hidden topic:** CSS cannot traverse upward (no parent selector in Selenium's CSS engine) — use XPath `parent::` or `ancestor::` for that
- **Hidden topic:** escaping special characters — IDs or classes with `.`, `:`, `[` need escaping: `#my\.id` or `[id='my.id']`

### 3.3 XPath
- Absolute: `/html/body/div[1]/form/input` — fragile, breaks on any DOM change
- Relative: `//input[@id='email']` — starts search from anywhere
- Axes: `parent::`, `ancestor::`, `child::`, `descendant::`, `following-sibling::`, `preceding-sibling::`, `following::`, `preceding::`, `self::`
- Functions: `contains(@attr, 'value')`, `starts-with(@attr, 'value')`, `text()`, `normalize-space()`, `not()`, `last()`, `position()`, `string-length()`, `concat()`, `translate()`
- Boolean operators: `and`, `or`
- Indexing: `(//div[@class='item'])[3]` — third match globally; `//div[@class='item'][3]` — third child within each parent
- **Hidden topic:** `text()` vs `.` — `text()` returns only direct text nodes; `.` (dot) returns the concatenated text of the element and all descendants; `//div[.='Full Text']` is often more reliable than `//div[text()='Full Text']`
- **Hidden topic:** `normalize-space()` — strips leading/trailing whitespace and collapses internal whitespace; essential for matching text that has `\n`, `\t`, or multiple spaces in the source HTML
- **Hidden topic:** XPath indexing is 1-based, not 0-based — `[1]` is the first element
- **Hidden topic:** `//div[@class='exact']` does exact match — won't match `class="exact other"`; use `contains(@class, 'exact')` for partial match, but this can over-match (`class="inexact"` also matches)
- **Hidden topic:** XPath `|` operator — `//input | //textarea` selects elements matching either expression
- **Hidden topic:** `translate()` for case-insensitive matching — `//div[translate(text(),'ABCDEFGHIJKLMNOPQRSTUVWXYZ','abcdefghijklmnopqrstuvwxyz')='hello']`

### 3.4 Relative Locators (Selenium 4)
- `RelativeLocator.with(By.tagName("input")).above(element)` — element above another
- `below(element)`, `toLeftOf(element)`, `toRightOf(element)`, `near(element)`
- Chaining: `.above(el1).toRightOf(el2)`
- Accepts both `WebElement` and `By` as arguments
- **Hidden topic:** relative locators use `getRect()` (bounding box) for position calculation — invisible or zero-size elements produce incorrect results
- **Hidden topic:** `near()` default distance is 50px — pass `near(element, 100)` for larger search radius
- **Hidden topic:** performance — relative locators execute JavaScript internally to compute positions; slower than direct CSS/XPath
- **Hidden topic:** they can return unexpected elements in complex/overlapping layouts — always verify the returned element

### 3.5 Shadow DOM
- Open shadow roots: `element.getShadowRoot()` (Selenium 4.1+)
- Then find elements within: `shadowRoot.findElement(By.cssSelector("..."))`
- **Hidden topic:** only CSS selectors work inside shadow roots — XPath does not work across shadow boundaries
- **Hidden topic:** nested shadow DOMs — must traverse each shadow root level by level
- **Hidden topic:** closed shadow roots are inaccessible via `getShadowRoot()` — use `executeScript("return arguments[0].shadowRoot", element)` as a workaround (works for open only; closed shadows require JS injection at page load)
- **Hidden topic:** `::shadow` and `/deep/` CSS combinators are deprecated and removed — `getShadowRoot()` is the only supported path

### 3.6 Locator Strategy Best Practices
- Priority order: `id` > `data-testid` / `data-qa` > CSS selector > XPath
- Prefer user-facing attributes: `name`, `placeholder`, `role`, `aria-label`
- Avoid: positional XPath, auto-generated IDs, deeply nested paths, index-dependent locators
- **Hidden topic:** request `data-testid` attributes from developers — it's a standard practice in modern frontend development; React Testing Library promotes it
- **Hidden topic:** dynamic class names (CSS modules, Tailwind JIT) — class names like `btn-abc12` change on every build; never use them as locators
- **Hidden topic:** `findElements()` (plural) returns an empty list when nothing matches (no exception) — use it to check element existence without try-catch

---

## Phase 4: Element Interactions

### 4.1 Basic Element Methods
- `click()` — clicks the center of the element; element must be visible and not obscured
- `sendKeys(CharSequence... keys)` — types text; appends to existing value; dispatches keydown/keypress/keyup events
- `clear()` — clears input/textarea; fires `change` event
- `submit()` — submits the parent form; deprecated behavior in some contexts
- `getText()` — returns visible (rendered) text; returns `""` for hidden elements
- `getAttribute(String name)` — in Selenium 4, returns the IDL property value (JavaScript property)
- `getDomAttribute(String name)` — returns the HTML attribute value as written in the markup
- `getDomProperty(String name)` — returns the current JavaScript property value
- `getCssValue(String property)` — returns computed CSS value (e.g., `"rgba(0, 0, 0, 1)"` for color)
- `getTagName()` — returns the element's HTML tag
- `getRect()` — returns `Rectangle` with x, y, width, height
- `getAriaRole()`, `getAccessibleName()` — accessibility properties (Selenium 4)
- `isDisplayed()`, `isEnabled()`, `isSelected()`
- `getScreenshotAs(OutputType)` — element-level screenshot (Selenium 4)
- **Hidden topic:** `getAttribute("value")` returns the current value for `<input>` (IDL property); `getDomAttribute("value")` returns the initial HTML attribute — this distinction trips people up when checking whether a field was programmatically updated
- **Hidden topic:** `getText()` returns `""` for `<input>` elements — use `getAttribute("value")` or `getDomProperty("value")` instead
- **Hidden topic:** `isDisplayed()` checks CSS visibility, display, opacity AND checks if the element has a size > 0 — an element can be in the DOM but return `false` for many reasons
- **Hidden topic:** `sendKeys()` does NOT clear existing text — always call `clear()` first, or select-all then type: `sendKeys(Keys.chord(Keys.CONTROL, "a"), "new text")`

### 4.2 Stale Element Reference
- Occurs when the DOM element has been destroyed and re-created (page reload, AJAX update, SPA re-render)
- The WebElement reference becomes invalid — any method call throws `StaleElementReferenceException`
- Patterns to handle:
  - Re-locate the element before each interaction
  - Wrap interactions in retry logic
  - Use `ExpectedConditions.stalenessOf(element)` to wait for staleness, then re-find
- **Hidden topic:** React/Angular/Vue frequently re-render components — element references go stale even when the UI looks unchanged; this is the #1 cause of flaky tests in SPAs
- **Hidden topic:** storing `WebElement` in instance variables is risky for dynamic pages — store the `By` locator instead and find fresh each time
- **Hidden topic:** `@CacheLookup` in Page Factory makes staleness worse — the cached reference never refreshes

### 4.3 Element Interactability
- Selenium checks before `click()`/`sendKeys()`: element must be visible, enabled, and not obscured by another element
- `ElementClickInterceptedException` — another element is covering the target (overlays, sticky headers, cookie banners)
- `ElementNotInteractableException` — element exists in DOM but is hidden, zero-size, or otherwise not interactable
- **Hidden topic:** scrolling into view happens automatically before interaction — but only vertical scrolling; horizontal scroll issues can still cause failures
- **Hidden topic:** fixed/sticky elements (nav bars, cookie banners) can intercept clicks — scroll extra pixels or close the overlay first
- **Hidden topic:** `element.click()` clicks the element center — if the center is covered (e.g., a label overlapping an input), it fails even if edges are visible

---

## Phase 5: Waits & Synchronization

### 5.1 Implicit Waits
- `driver.manage().timeouts().implicitlyWait(Duration.ofSeconds(10))`
- Applies globally to all `findElement`/`findElements` calls
- Polls the DOM until the element appears or timeout expires
- Returns immediately if element is found
- **Hidden topic:** implicit wait only waits for element presence in the DOM — not for visibility, clickability, or any other condition
- **Hidden topic:** implicit wait affects `findElements()` too — it waits the full duration before returning an empty list if count > 0 is expected

### 5.2 Explicit Waits
- `WebDriverWait wait = new WebDriverWait(driver, Duration.ofSeconds(15))`
- `wait.until(ExpectedConditions.condition(locator))`
- Common `ExpectedConditions`:
  - `presenceOfElementLocated(By)` — in DOM (may be hidden)
  - `visibilityOfElementLocated(By)` — in DOM AND displayed
  - `visibilityOf(WebElement)` — already-found element becomes visible
  - `elementToBeClickable(By)` — visible AND enabled
  - `invisibilityOfElementLocated(By)` — gone from DOM or hidden
  - `textToBePresentInElement(WebElement, text)` — element contains expected text
  - `textToBePresentInElementValue(By, text)` — input value contains text
  - `frameToBeAvailableAndSwitchToIt(By)` — frame exists and switches to it
  - `alertIsPresent()` — alert dialog appears
  - `numberOfElementsToBe(By, count)` — exact element count
  - `numberOfElementsToBeMoreThan(By, count)` / `toBeLessThan`
  - `stalenessOf(WebElement)` — element becomes stale (useful for waiting for page updates)
  - `urlContains(String)`, `urlToBe(String)`, `titleContains(String)`, `titleIs(String)`
  - `attributeToBe(By, attr, value)`, `attributeContains(By, attr, value)`
  - `elementToBeSelected(By)` / `elementSelectionStateToBe(By, boolean)`
- **Hidden topic:** `presenceOfElementLocated` ≠ `visibilityOfElementLocated` — an element can be present but hidden (`display:none`); using presence when you need visibility leads to `ElementNotInteractableException`
- **Hidden topic:** `elementToBeClickable` checks `isDisplayed() && isEnabled()` — but it does NOT check if the element is obscured; you can still get `ElementClickInterceptedException`

### 5.3 Fluent Waits
- `FluentWait<WebDriver>` — more configurable than `WebDriverWait`
- `withTimeout(Duration)` — max wait time
- `pollingEvery(Duration)` — how often to check the condition
- `ignoring(Exception.class)` — exceptions to swallow during polling (e.g., `NoSuchElementException`, `StaleElementReferenceException`)
- `withMessage(String)` — custom timeout message
- **Hidden topic:** `WebDriverWait` IS a `FluentWait<WebDriver>` — it's a convenience subclass with default polling of 500ms
- **Hidden topic:** the `ignoring()` clause is critical — without `ignoring(StaleElementReferenceException.class)`, a stale reference during polling immediately throws instead of retrying
- **Hidden topic:** you can pass any `Function<WebDriver, T>` to `until()` — not limited to `ExpectedConditions`; write lambdas for complex conditions

### 5.4 Custom Expected Conditions
- Lambda-based: `wait.until(driver -> driver.findElement(By.id("x")).getText().length() > 0)`
- Class-based: implement `ExpectedCondition<T>` for reusable conditions
- Return type determines behavior: returning `null` or `false` means "keep waiting"; any other value means "condition met"
- **Hidden topic:** the return value of `until()` IS the value returned by the condition — for `ExpectedCondition<WebElement>`, you get the element back; no need to `findElement` again
- **Hidden topic:** combine conditions with `ExpectedConditions.and()`, `or()`, `not()` — e.g., `and(visibilityOfElementLocated(by), elementToBeClickable(by))`

### 5.5 Why NOT to Mix Implicit and Explicit Waits
- Implicit wait sets a baseline polling timeout on ALL find operations
- Explicit wait internally calls `findElement` which inherits the implicit timeout
- Example: implicit = 10s, explicit = 15s → each polling attempt inside the explicit wait takes up to 10s → total effective wait can be 10s × (15s / 0.5s polling) = much larger than expected
- **Hidden topic:** the official Selenium docs explicitly warn against mixing — use implicit wait = 0 and explicit waits for all synchronization
- **Hidden topic:** `findElements` inside an explicit wait with implicit > 0 is especially problematic — it waits the full implicit duration before returning an empty list on each poll

---

## Phase 6: Window, Frame & Alert Handling

### 6.1 Window and Tab Management
- `driver.getWindowHandle()` — returns the current window's handle (unique string)
- `driver.getWindowHandles()` — returns `Set<String>` of all open window/tab handles
- `driver.switchTo().window(handle)` — switch context to a specific window/tab
- `driver.switchTo().newWindow(WindowType.TAB)` — opens new tab and switches to it (Selenium 4)
- `driver.switchTo().newWindow(WindowType.WINDOW)` — opens new window and switches to it (Selenium 4)
- `driver.close()` — closes the current window/tab only
- `driver.quit()` — closes all windows and ends the WebDriver session
- **Hidden topic:** `getWindowHandles()` returns a `Set` — iteration order is NOT guaranteed across browsers; never rely on `.toArray()[1]` being the "new" window
- **Hidden topic:** correct pattern: store handle before action → perform action that opens new window → diff the handle sets → switch to the new handle
- **Hidden topic:** after `close()`, the driver's focus is undefined — you must `switchTo().window(previousHandle)` explicitly or you'll get `NoSuchWindowException` on the next command
- **Hidden topic:** `driver.close()` on the last remaining window without calling `quit()` leaves the WebDriver process running as an orphan

### 6.2 Frames and iFrames
- `driver.switchTo().frame(int index)` — by zero-based index (fragile)
- `driver.switchTo().frame(String nameOrId)` — by `name` or `id` attribute of the frame element
- `driver.switchTo().frame(WebElement frameElement)` — by the frame/iframe WebElement itself (most reliable)
- `driver.switchTo().defaultContent()` — returns to the top-level document
- `driver.switchTo().parentFrame()` — goes up one frame level (Selenium 4 made this reliable)
- **Hidden topic:** all element locators are scoped to the current frame context — an element inside an iframe is invisible from the main frame; you MUST switch first
- **Hidden topic:** nested iframes — must switch into each level sequentially; `defaultContent()` jumps all the way out, `parentFrame()` goes up one level
- **Hidden topic:** `ExpectedConditions.frameToBeAvailableAndSwitchToIt(By)` — combines waiting for the frame to load AND switching in one call
- **Hidden topic:** cross-origin iframes — Selenium can switch into them and interact with elements, but `executeScript()` inside a cross-origin frame hits browser same-origin policy violations

### 6.3 Alerts, Confirms, and Prompts
- `Alert alert = driver.switchTo().alert()`
- `alert.getText()` — reads the alert message
- `alert.accept()` — clicks OK / Accept
- `alert.dismiss()` — clicks Cancel / Dismiss
- `alert.sendKeys(String text)` — types into a prompt dialog
- **Hidden topic:** always wait for the alert first: `wait.until(ExpectedConditions.alertIsPresent())` — switching to a non-existent alert throws `NoAlertPresentException`
- **Hidden topic:** JavaScript `alert()`, `confirm()`, `prompt()` are true browser dialogs — Selenium handles these; but sweet alerts, modals, toasts are HTML elements — interact with them normally
- **Hidden topic:** `UnhandledAlertBehaviour` capability — controls what happens if a command is issued while an alert is open: `ACCEPT`, `DISMISS`, `IGNORE`, `ACCEPT_AND_NOTIFY`, `DISMISS_AND_NOTIFY`
- **Hidden topic:** Chrome blocks `alert()` in some contexts (e.g., `beforeunload` during navigation) — test behavior may differ from manual interaction

---

## Phase 7: Actions API (Advanced Interactions)

### 7.1 Mouse Actions
- `Actions actions = new Actions(driver)`
- `actions.moveToElement(element)` — hover; moves to element center
- `actions.moveToElement(element, xOffset, yOffset)` — hover with offset from center
- `actions.click()`, `click(element)` — click at current position or on element
- `actions.doubleClick()`, `doubleClick(element)`
- `actions.contextClick()`, `contextClick(element)` — right-click
- `actions.clickAndHold(element)` → `moveToElement(target)` → `release()` — drag and drop manual chain
- `actions.dragAndDrop(source, target)` — convenience method
- `actions.moveByOffset(x, y)` — move relative to current cursor position
- **Hidden topic:** `dragAndDrop()` fails on HTML5 drag-and-drop elements — the HTML5 Drag and Drop API uses different events (`dragstart`, `dragenter`, `drop`) that Selenium's Actions don't fully trigger; use a JavaScript-based workaround
- **Hidden topic:** `moveToElement()` in Selenium 4 (W3C) moves to the element's CENTER; in legacy mode it moved to the top-left corner — this changes hover and offset calculations
- **Hidden topic:** `moveByOffset()` offsets are from the CURRENT mouse position, not from the element — track where the cursor is or use `moveToElement` first

### 7.2 Keyboard Actions
- `actions.keyDown(Keys.SHIFT)` → `sendKeys("text")` → `keyUp(Keys.SHIFT)` — types "TEXT"
- `actions.sendKeys(element, "text")` — sends keys to specific element
- `actions.sendKeys(Keys.ENTER)`, `Keys.TAB`, `Keys.ESCAPE`, `Keys.BACK_SPACE`, `Keys.DELETE`
- `Keys.chord(Keys.CONTROL, "a")` — select all (static utility, not an action)
- Key combinations in actions: `keyDown(Keys.CONTROL).sendKeys("c").keyUp(Keys.CONTROL)` — Ctrl+C
- **Hidden topic:** `Keys.CONTROL` vs `Keys.COMMAND` — use `Keys.COMMAND` on macOS, `Keys.CONTROL` on Windows/Linux; or detect OS and choose dynamically
- **Hidden topic:** `keyDown` without `keyUp` leaves the key pressed — subsequent actions will have that modifier active; always release keys
- **Hidden topic:** some applications use `keydown` event only (not `keypress` or `keyup`) — `sendKeys` on the element may work differently than Actions `keyDown`/`keyUp`

### 7.3 Scroll Actions (Selenium 4)
- `actions.scrollToElement(element)` — scrolls the element into the viewport
- `actions.scrollByAmount(deltaX, deltaY)` — scroll by pixel amounts
- `actions.scrollFromOrigin(WheelInput.ScrollOrigin.fromElement(element), 0, 300)` — scroll from an element's position
- `actions.scrollFromOrigin(WheelInput.ScrollOrigin.fromViewport(x, y), 0, 500)` — scroll from a viewport position
- **Hidden topic:** these scroll actions use the Wheel input source in W3C — they simulate actual mouse wheel events, which may trigger JavaScript scroll handlers
- **Hidden topic:** `scrollToElement` scrolls the element to the viewport center — not the top; offset if you need the element at a specific viewport position

### 7.4 Pen / Pointer Input (Selenium 4)
- Pointer input sources: `MOUSE`, `TOUCH`, `PEN`
- Pen-specific: pressure, tilt angles, twist
- Touch gestures: tap, long press (via pointer actions)
- **Hidden topic:** pen/touch input is mainly useful for testing on devices or emulators — desktop browsers generally respond to mouse events for these

### 7.5 Action Chains & Performance
- All action methods return the `Actions` object — chain them: `actions.moveToElement(el).click().sendKeys("text").perform()`
- `perform()` — executes the entire chain; MUST be called or nothing happens
- `build().perform()` — `build()` creates the `Action` object without executing; `perform()` calls `build()` internally
- `Actions.pause(Duration.ofMillis(500))` — insert a delay in the middle of an action chain
- **Hidden topic:** each `perform()` call sends the entire action sequence as a single W3C request — it's more efficient than separate commands
- **Hidden topic:** `pause()` is the correct way to add delays in action chains — NOT `Thread.sleep()` between actions
- **Hidden topic:** a new `Actions(driver)` instance starts with a clean state — actions from previous chains don't carry over

---

## Phase 8: JavaScript Executor

### 8.1 Synchronous Execution
- `JavascriptExecutor js = (JavascriptExecutor) driver`
- `js.executeScript("return document.title;")` — returns String
- `js.executeScript("arguments[0].click();", element)` — pass WebElement as argument
- `js.executeScript("arguments[0].scrollIntoView(true);", element)` — scroll into view
- `js.executeScript("arguments[0].setAttribute('style', 'border: 2px solid red');", element)` — highlight
- `js.executeScript("return arguments[0].value;", inputElement)` — read input value
- `js.executeScript("arguments[0].value = arguments[1];", inputElement, "new value")` — set input value directly
- `js.executeScript("return document.readyState;")` — check page load state (`"loading"`, `"interactive"`, `"complete"`)
- Return type mapping: JS string → Java `String`, JS number → Java `Long`/`Double`, JS boolean → Java `Boolean`, JS array → Java `List`, JS object → Java `Map`, JS null → Java `null`, JS element → Java `WebElement`
- **Hidden topic:** JS `click()` bypasses Selenium's visibility and interactability checks — it can click hidden elements that `WebElement.click()` would reject; useful for unblocking, but it masks real UI bugs
- **Hidden topic:** setting `.value` via JS does NOT fire `input`, `change`, or `keyup` events — React/Angular controlled inputs won't update their state; dispatch events manually: `arguments[0].dispatchEvent(new Event('input', { bubbles: true }))`

### 8.2 Asynchronous Execution
- `js.executeAsyncScript(script, args...)` — for operations that need a callback (AJAX, timers, animations)
- The last argument injected into the script is the callback: `arguments[arguments.length - 1]`
- Must call the callback to signal completion: `var done = arguments[arguments.length-1]; setTimeout(function() { done('result'); }, 2000);`
- Subject to `scriptTimeout` — throws `ScriptTimeoutException` if callback not invoked in time
- **Hidden topic:** if you forget to call the callback, the script hangs until `scriptTimeout` — always structure async scripts with the callback invocation
- **Hidden topic:** use `executeAsyncScript` for waiting on custom async operations: AJAX calls completing, animations finishing, WebSocket messages arriving

### 8.3 Common JavaScript Patterns in Selenium
```
// Scroll to bottom of page
js.executeScript("window.scrollTo(0, document.body.scrollHeight);");

// Remove element (e.g., overlay blocking clicks)
js.executeScript("arguments[0].remove();", overlayElement);

// Get all attributes of an element
js.executeScript("var attrs = {}; for (var a of arguments[0].attributes) { attrs[a.name] = a.value; } return attrs;", element);

// Wait for jQuery AJAX to complete
js.executeAsyncScript("var done = arguments[0]; (function check() { if (jQuery.active === 0) done(); else setTimeout(check, 100); })();");

// Check if element is in viewport
js.executeScript("var r = arguments[0].getBoundingClientRect(); return r.top >= 0 && r.left >= 0 && r.bottom <= window.innerHeight && r.right <= window.innerWidth;", element);
```
- **Hidden topic:** `executeScript` runs in the page's JavaScript context — you have access to the page's global variables, jQuery, Angular internals, React devtools, etc.
- **Hidden topic:** be cautious with `arguments[0].remove()` — it permanently modifies the DOM; the element won't come back until a page reload or re-render

---

## Phase 9: Handling Specific UI Components

### 9.1 Select Dropdowns (`<select>`)
- `Select select = new Select(driver.findElement(By.id("dropdown")))`
- `select.selectByVisibleText("Option Text")`
- `select.selectByValue("optionValue")`
- `select.selectByIndex(2)` — zero-based
- `select.getOptions()` — `List<WebElement>` of all `<option>` elements
- `select.getFirstSelectedOption()` — currently selected option
- `select.getAllSelectedOptions()` — for multi-select
- `select.deselectAll()`, `deselectByVisibleText()`, `deselectByValue()`, `deselectByIndex()` — multi-select only
- `select.isMultiple()` — check if multi-select
- **Hidden topic:** `Select` ONLY works on native `<select>` HTML elements — custom dropdowns (React Select, Material UI, Ant Design, Bootstrap) are `<div>`-based; interact by clicking the container, then clicking the option `<li>` / `<div>`
- **Hidden topic:** `UnexpectedTagNameException` — thrown if you wrap a non-`<select>` element with `new Select(element)`

### 9.2 Custom Dropdowns (Non-`<select>`)
- Click the dropdown trigger element to open the list
- Wait for the option list to become visible
- Find and click the desired option
- Common pattern: `driver.findElement(By.cssSelector(".dropdown-trigger")).click()` → `wait.until(visibilityOfElementLocated(By.cssSelector(".dropdown-options")))` → click option
- **Hidden topic:** some custom dropdowns use virtual scrolling — not all options are in the DOM at once; you may need to scroll within the dropdown list to load more options

### 9.3 Date Pickers
- Strategy 1: type directly into the input if it accepts keyboard input — `sendKeys("12/25/2025")`
- Strategy 2: interact with the calendar UI — click month/year navigators, then click the day
- Strategy 3: set the value via JavaScript — `executeScript("arguments[0].value = '2025-12-25'", dateInput)`
- **Hidden topic:** many date pickers are `readonly` inputs — `sendKeys()` won't work; use JS to remove `readonly`, set value, and dispatch change event
- **Hidden topic:** date format varies by locale — `MM/DD/YYYY` vs `DD/MM/YYYY`; match the application's expected format

### 9.4 File Upload
- Standard: `driver.findElement(By.cssSelector("input[type='file']")).sendKeys(absoluteFilePath)`
- **Hidden topic:** hidden file inputs — use JS to make visible first: `executeScript("arguments[0].style.display = 'block'; arguments[0].style.visibility = 'visible';", fileInput)`
- **Hidden topic:** some frameworks wrap `<input type='file'>` in custom components — locate the actual `<input>` in the DOM, not the styled button
- **Hidden topic:** multiple files: `sendKeys("C:\\path1.txt\nC:\\path2.txt")` — newline-separated paths (browser-dependent)
- **Hidden topic:** remote execution (Grid) — file must exist on the machine running the browser, not the client; use `LocalFileDetector`: `((RemoteWebDriver) driver).setFileDetector(new LocalFileDetector())`

### 9.5 File Download
- Configure browser to download without dialog:
  - Chrome: `prefs.put("download.default_directory", path)` + `prefs.put("download.prompt_for_download", false)`
  - Firefox: `profile.setPreference("browser.download.dir", path)` + `profile.setPreference("browser.helperApps.neverAsk.saveToDisk", "application/pdf,application/octet-stream,..."`
- Verify download: poll the directory for the expected file with a wait
- **Hidden topic:** `.crdownload` (Chrome) / `.part` (Firefox) — temporary file extensions during download; wait until the file no longer has these extensions
- **Hidden topic:** on Grid/remote, file download goes to the remote machine — use `se:downloadsEnabled` capability and the Grid download endpoint to retrieve files

### 9.6 Tables
- Locate table: `driver.findElement(By.cssSelector("table"))`
- Rows: `table.findElements(By.tagName("tr"))`
- Cells: `row.findElements(By.tagName("td"))` or `By.tagName("th")`
- Reading specific cell: `//table//tr[3]/td[2]`
- Dynamic table: iterate rows, match by column content
- **Hidden topic:** `<thead>`, `<tbody>`, `<tfoot>` sections — `//table/tr` fails if rows are inside `<tbody>`; use `//table//tr` or `//table/tbody/tr`
- **Hidden topic:** virtual/paginated tables — only visible rows are in the DOM; must paginate or scroll to access all data

### 9.7 Tooltips, Hovering, and Context Menus
- Tooltip: `actions.moveToElement(element).perform()` → wait for tooltip element → read text
- Context menu: `actions.contextClick(element).perform()` → interact with the context menu (may be HTML or native browser menu)
- **Hidden topic:** native browser context menus cannot be interacted with via Selenium — only custom HTML context menus; use `Keys.ESCAPE` to dismiss native ones

---

## Phase 10: Screenshots & Visual Capture

### 10.1 Full Page / Viewport Screenshot
- `File screenshot = ((TakesScreenshot) driver).getScreenshotAs(OutputType.FILE)`
- `Files.copy(screenshot.toPath(), Path.of("screenshot.png"))`
- `OutputType.BYTES` → `byte[]` (for attaching to reports)
- `OutputType.BASE64` → Base64-encoded string
- **Hidden topic:** standard `TakesScreenshot` captures only the visible viewport — not the full scrollable page; Chrome, Firefox, and Edge may differ

### 10.2 Element Screenshot
- `File elementShot = element.getScreenshotAs(OutputType.FILE)` — captures only the element's bounding box
- **Hidden topic:** element screenshot clips to the viewport — if part of the element is off-screen, that portion is cut off

### 10.3 Full-Page Screenshot via CDP
- Chrome: `((ChromeDriver) driver).getDevTools()` → send `Page.captureScreenshot` with `captureBeyondViewport: true`
- Firefox: `((FirefoxDriver) driver).getFullPageScreenshotAs(OutputType.FILE)` — Firefox has native full-page support
- **Hidden topic:** CDP full-page screenshot renders the entire page including off-screen content — useful for regression testing but file sizes can be large

### 10.4 Screenshot Timing & Best Practices
- Take screenshots AFTER wait conditions are met, not before
- Screenshot on test failure: capture in `@AfterMethod` when `ITestResult.FAILURE`
- Name screenshots with test name + timestamp for traceability
- **Hidden topic:** overlapping screenshots (toast messages, loading spinners) can cause false visual diffs — wait for transient UI elements to disappear before capturing

---

## Phase 11: Chrome DevTools Protocol (CDP) & WebDriver BiDi

### 11.1 CDP Basics
- `DevTools devTools = ((HasDevTools) driver).getDevTools()`
- `devTools.createSession()` — opens a CDP session
- `devTools.send(command)` — execute CDP commands
- `devTools.addListener(event, handler)` — subscribe to CDP events
- Works with Chrome, Edge (Chromium-based) — NOT Firefox or Safari

### 11.2 Network Domain
- `devTools.send(Network.enable(Optional.empty(), Optional.empty(), Optional.empty()))` — enable network tracking
- `devTools.addListener(Network.requestWillBeSent(), request -> { ... })` — observe outgoing requests
- `devTools.addListener(Network.responseReceived(), response -> { ... })` — observe responses
- `devTools.send(Network.emulateNetworkConditions(false, 200, 1000, 500, Optional.empty()))` — simulate slow network (offline, latency, download/upload throughput)
- **Hidden topic:** `Fetch.enable()` vs `Network.enable()` — `Fetch` intercepts and can modify/block requests; `Network` only observes; use `Fetch` for mocking, `Network` for logging
- **Hidden topic:** `NetworkInterceptor` (Selenium's high-level wrapper) simplifies request mocking — `new NetworkInterceptor(driver, route -> ...)`

### 11.3 Geolocation Mocking
- `devTools.send(Emulation.setGeolocationOverride(Optional.of(lat), Optional.of(lng), Optional.of(accuracy)))`
- **Hidden topic:** only affects `navigator.geolocation` — does not change IP-based geolocation

### 11.4 Device Emulation
- `devTools.send(Emulation.setDeviceMetricsOverride(width, height, deviceScaleFactor, mobile, ...))` — simulate mobile viewport
- `devTools.send(Emulation.setUserAgentOverride(userAgent, ...))` — change user-agent string
- **Hidden topic:** device emulation via CDP is more accurate than just resizing the window — it sets device pixel ratio, touch events, and user-agent

### 11.5 Console and Exception Capture
- `devTools.addListener(Log.entryAdded(), entry -> { ... })` — capture browser console logs
- `devTools.getDomains().events().addConsoleListener(consoleEvent -> { ... })` — Selenium convenience API
- `driver.manage().logs().get(LogType.BROWSER)` — legacy log retrieval (limited in W3C mode)
- **Hidden topic:** console error capture is valuable for detecting JavaScript errors during test execution — log them in test reports

### 11.6 Performance Metrics
- `devTools.send(Performance.enable(Optional.empty()))`
- `devTools.send(Performance.getMetrics())` — returns metrics like `JSHeapUsedSize`, `Documents`, `Nodes`, `LayoutCount`, etc.
- **Hidden topic:** these are runtime metrics — useful for detecting memory leaks or excessive DOM growth during tests

### 11.7 WebDriver BiDi (The Future)
- Cross-browser replacement for CDP — works on Chrome, Firefox, Edge, Safari (eventually)
- Event-driven protocol — subscribe to events instead of polling
- Selenium is progressively implementing BiDi support
- `driver.script().addConsoleMessageHandler()` — BiDi console listener
- `driver.script().addJavaScriptErrorHandler()` — BiDi JS error listener
- **Hidden topic:** BiDi is still evolving — not all CDP features have BiDi equivalents yet; use CDP for Chrome-specific needs, BiDi for cross-browser event handling
- **Hidden topic:** BiDi uses WebSocket (bidirectional) vs WebDriver's HTTP (request-response) — enables true event subscriptions without polling

---

## Phase 12: Selenium Grid

### 12.1 Grid 4 Architecture
- **Standalone** — all components in one process: `java -jar selenium-server-4.x.jar standalone`
- **Hub-Node** — hub receives requests, routes to nodes:
  - Hub: `java -jar selenium-server-4.x.jar hub`
  - Node: `java -jar selenium-server-4.x.jar node --hub http://hub:4444`
- **Fully Distributed** — each component runs independently: Router, Distributor, Session Map, Session Queue, Event Bus, Node
- Components:
  - **Router** — entry point, receives session requests
  - **Distributor** — assigns sessions to nodes based on capabilities
  - **Session Map** — tracks which session runs on which node
  - **Session Queue** — holds pending session requests
  - **Event Bus** — internal component communication
  - **Node** — runs browsers, executes commands
- **Hidden topic:** Grid 4 uses HTTPS for internal communication (optional, configurable) and supports observability via OpenTelemetry
- **Hidden topic:** `/graphql` endpoint — query grid status, sessions, nodes programmatically
- **Hidden topic:** Grid 4 session queue has configurable timeout (`--session-request-timeout`) — requests that wait too long are rejected
- **Hidden topic:** drain mode — `cURL POST /se/grid/distributor/node/{nodeId}/drain` gracefully stops a node after current sessions complete

### 12.2 RemoteWebDriver
- `RemoteWebDriver driver = new RemoteWebDriver(new URI("http://grid:4444").toURL(), options)`
- Same WebDriver API — tests run identically on local and remote
- Capabilities determine which browser/OS the Grid assigns
- `driver.getSessionId()` — unique session identifier
- **Hidden topic:** `LocalFileDetector` — `((RemoteWebDriver) driver).setFileDetector(new LocalFileDetector())` — required for file uploads when the file is on the client machine, not the Grid node
- **Hidden topic:** `RemoteWebDriver` augmenting — `new Augmenter().augment(driver)` adds interfaces like `HasDevTools` to remote driver instances

### 12.3 Docker-Based Grid
- Official images: `selenium/hub`, `selenium/node-chrome`, `selenium/node-firefox`, `selenium/node-edge`
- Standalone: `selenium/standalone-chrome`
- `docker-compose.yml` for multi-browser grid
- Video recording: `selenium/video:ffmpeg-*` sidecar containers
- Dynamic Grid: `selenium/standalone-docker` — spawns browser containers on demand
- **Hidden topic:** `--shm-size=2g` — MUST set for Chrome containers; default 64MB `/dev/shm` causes Chrome tab crashes
- **Hidden topic:** VNC access — each node exposes a VNC port (default 5900 + offset); use `selenium/node-chrome-debug` or connect to the VNC port for live debugging
- **Hidden topic:** resource limits — set Docker memory/CPU limits to prevent a single test from starving other containers

### 12.4 Cloud Grids
- BrowserStack, Sauce Labs, LambdaTest — managed Selenium Grid in the cloud
- Connect via `RemoteWebDriver` with the vendor's hub URL
- Vendor capabilities: `bstack:options`, `sauce:options`, `LT:Options` — must be W3C-namespaced
- **Hidden topic:** cloud grids have idle timeout (typically 60-90s) — long waits or `Thread.sleep` can cause the session to terminate
- **Hidden topic:** test status reporting — send pass/fail status via vendor API or capabilities after test completion for dashboard integration

---

## Phase 13: Cookies, Storage & Session Management

### 13.1 Cookie Operations
- `driver.manage().addCookie(new Cookie("name", "value"))` — add a cookie
- `driver.manage().getCookieNamed("name")` — get a specific cookie
- `driver.manage().getCookies()` — get all cookies (`Set<Cookie>`)
- `driver.manage().deleteCookieNamed("name")` — delete one
- `driver.manage().deleteAllCookies()` — delete all
- `Cookie` builder: `new Cookie.Builder("name", "value").domain(".example.com").path("/").isSecure(true).expiresOn(date).build()`
- **Hidden topic:** cookies are domain-scoped — `addCookie()` fails if you haven't navigated to that domain first; navigate to the domain, add cookies, then navigate to the target page
- **Hidden topic:** `SameSite` attribute — modern browsers block `SameSite=None` cookies without `Secure` flag; tests on `http://` may silently fail to set cookies
- **Hidden topic:** cookie size limit is ~4KB per cookie — exceeding it causes silent truncation in some browsers

### 13.2 Browser Storage via JavaScript
- `executeScript("return window.localStorage.getItem('key');")` — read localStorage
- `executeScript("window.localStorage.setItem('key', 'value');")` — write localStorage
- `executeScript("window.localStorage.removeItem('key');")` — remove
- `executeScript("window.localStorage.clear();")` — clear all
- Same pattern for `sessionStorage`
- **Hidden topic:** localStorage is origin-scoped (protocol + domain + port) — you must be on the correct origin before reading/writing
- **Hidden topic:** setting auth tokens in localStorage can skip login flows in tests — useful for speed, but ensure at least one test validates the actual login UI

### 13.3 Session Transfer Patterns
- Copy cookies from one driver session to another (or from API to browser)
- Copy localStorage/sessionStorage tokens
- **Hidden topic:** session fixation — be careful about reusing session identifiers across tests; always start with a clean session for security-sensitive tests

---

## Phase 14: Selenium Exceptions (Complete Reference)

### 14.1 Common Exceptions
- `NoSuchElementException` — element not found in the current DOM/frame context
- `StaleElementReferenceException` — element was in DOM but has been removed/re-rendered
- `ElementNotInteractableException` — element is in DOM but not visible/enabled for interaction
- `ElementClickInterceptedException` — another element is receiving the click instead (overlay, sticky header)
- `TimeoutException` — `WebDriverWait` condition not met within timeout; page load exceeded `pageLoadTimeout`
- `NoSuchWindowException` — target window/tab has been closed or doesn't exist
- `NoSuchFrameException` — frame not found by index, name, or element
- `NoAlertPresentException` — `switchTo().alert()` called with no alert present
- `InvalidSelectorException` — malformed CSS selector or XPath expression
- `JavascriptException` — error thrown by JS code inside `executeScript`
- `ScriptTimeoutException` — `executeAsyncScript` callback not invoked within `scriptTimeout`
- `SessionNotCreatedException` — browser/driver version mismatch, or browser failed to launch
- `WebDriverException` — base exception; catch-all for unexpected errors
- `UnexpectedTagNameException` — `new Select(element)` on a non-`<select>` element
- `InvalidArgumentException` — wrong argument type/value (e.g., negative timeout, malformed URL)
- `InsecureCertificateException` — SSL certificate rejected and `acceptInsecureCerts` not set
- `UnhandledAlertException` — a command was issued while a browser alert was open

### 14.2 Exception Handling Patterns
- **Retry pattern**: catch `StaleElementReferenceException`, re-locate, retry (with max attempts)
- **Wait-before-act**: use explicit waits to prevent `NoSuchElementException` and `ElementNotInteractableException`
- **Scroll-before-click**: scroll element into view before `click()` to prevent `ElementClickInterceptedException`
- **Hidden topic:** exception messages contain useful diagnostics — element info, expected state, actual state; always log the full message
- **Hidden topic:** `WebDriverException: unknown error: net::ERR_CONNECTION_REFUSED` — driver process crashed or network issue between client and driver; check if the browser/driver is still alive
- **Hidden topic:** intermittent `TimeoutException` in CI often means the environment is slower — increase timeouts for CI, or investigate resource contention

---

## Phase 15: Page Object Model & Page Factory

### 15.1 Page Object Model (POM)
- Each page/component → one Java class
- Web elements as private members with `By` locators
- Public methods represent user actions: `enterUsername()`, `clickLogin()`, `getErrorMessage()`
- Return page objects from methods for fluent chaining: `loginPage.enterUsername("u").enterPassword("p").clickLogin()` returns `DashboardPage`
- No assertions in page objects — assertions belong in test classes
- No WebDriver logic in test classes — all interactions go through page objects
- **Hidden topic:** page objects for components (not just pages) — `HeaderComponent`, `SidebarComponent`, `SearchResultsComponent`; compose them within page objects
- **Hidden topic:** avoid over-granular pages — one page object per logical page, not per section; too many classes create maintenance overhead
- **Hidden topic:** base page class — common methods (wait helpers, screenshot, element interaction wrappers) shared via inheritance or composition

### 15.2 Page Factory
- `@FindBy(id = "username")` → `private WebElement usernameInput;`
- `@FindBy(css = ".error-message")` → `private WebElement errorMessage;`
- `@FindBy(xpath = "//button[@type='submit']")` → `private WebElement submitButton;`
- `@FindBys({@FindBy(css = ".parent"), @FindBy(css = ".child")})` — AND logic (parent then child)
- `@FindAll({@FindBy(css = ".optionA"), @FindBy(css = ".optionB")})` — OR logic (matches either)
- `PageFactory.initElements(driver, this)` — initializes annotated fields with lazy proxy elements
- **Hidden topic:** elements are lazy proxies — `findElement` is called only when you interact with the element, NOT at `initElements` time; this means `initElements` never throws `NoSuchElementException`
- **Hidden topic:** `@CacheLookup` — caches the element reference after first use; ONLY safe for static elements (nav items, page title); NEVER use on dynamic content — causes `StaleElementReferenceException`
- **Hidden topic:** Page Factory does NOT support explicit waits natively — you can't add a `WebDriverWait` before the proxy resolves; you must use a separate `By` locator for waited interactions
- **Hidden topic:** `@FindBys` vs `@FindAll` — commonly confused; `@FindBys` = ALL conditions must match (AND), `@FindAll` = ANY condition can match (OR)
- **Hidden topic:** the Selenium project discourages Page Factory in favor of direct `By` locators + explicit waits — more control, easier debugging, no proxy magic

---

## Phase 16: Performance, Resource Management & Debugging

### 16.1 Selenium Performance Optimization
- Use CSS selectors over XPath when both are viable — slightly faster element resolution
- Reduce implicit wait to 0; use explicit waits only where needed
- Minimize `findElement` calls — if you need an element multiple times in quick succession, reuse the reference (safe on static pages)
- Use `PageLoadStrategy.EAGER` or `NONE` when you don't need full page loads
- Batch DOM reads with `executeScript()` — one HTTP roundtrip instead of N
- **Hidden topic:** each Selenium command is an HTTP request/response — on Grid or cloud, network latency compounds; 100 commands × 50ms latency = 5 seconds overhead
- **Hidden topic:** `findElements()` with implicit wait > 0 always waits the full duration if no elements are found — set implicit to 0 and use explicit waits for existence checks
- **Hidden topic:** headless browsers are 2-5x faster than headed mode — always use headless in CI pipelines

### 16.2 Resource Management
- `driver.quit()` — kills the browser process AND the driver server process; terminates the session
- `driver.close()` — closes only the current window; does NOT end the session unless it's the last window
- **Hidden topic:** not calling `quit()` leaves orphaned `chromedriver`/`geckodriver` processes — they accumulate in CI and eventually exhaust system resources
- **Hidden topic:** put `quit()` in a `finally` block or `@AfterMethod` — ensure it runs even when tests throw exceptions
- **Hidden topic:** `Runtime.getRuntime().addShutdownHook()` as a safety net — kills the driver on JVM exit; last resort, not a substitute for proper cleanup
- **Hidden topic:** on CI, add a cleanup step that kills all browser/driver processes: `taskkill /F /IM chromedriver.exe` (Windows) or `pkill -f chromedriver` (Linux)

### 16.3 Debugging Techniques
- Browser DevTools (F12) — inspect DOM, console, network while test runs (non-headless mode)
- `Thread.sleep()` for temporary debugging pauses — NEVER leave in production test code
- Screenshot on failure — capture the visual state at the point of failure
- Page source on failure — `driver.getPageSource()` to see the DOM at failure time
- Browser logs — `driver.manage().logs().get(LogType.BROWSER)` for JavaScript console errors
- Driver logs — `ChromeDriverService.Builder().withVerbose(true).withLogFile(file)` for protocol-level debugging
- **Hidden topic:** `--enable-logging --v=1` on ChromeDriver — produces verbose logs of every WebDriver command and response
- **Hidden topic:** HAR (HTTP Archive) files via BrowserMob Proxy or CDP network capture — record all network traffic for debugging failed tests
- **Hidden topic:** Selenium Grid has a `/status` endpoint and `/graphql` for querying session state — useful for debugging why tests hang on Grid

### 16.4 Handling Flaky Tests
- Root causes: timing issues, environment instability, test data dependencies, order-dependent tests
- Fix timing: replace `Thread.sleep` with explicit waits for specific conditions
- Fix test data: each test creates its own data; no shared state
- Fix order dependency: tests must be independently runnable
- Fix environment: stable test environments, consistent browser versions
- **Hidden topic:** retry mechanisms (TestNG `IRetryAnalyzer`, JUnit `@RepeatedTest`) are band-aids, not fixes — use them to stay green while investigating root causes
- **Hidden topic:** run flaky tests in a loop locally (`@Test(invocationCount = 50)` in TestNG) to reproduce timing issues

---

## Phase 17: Cross-Browser Testing

### 17.1 Browser Behavioral Differences
- **Chrome** — most widely used, best Selenium support, CDP access, Chrome for Testing builds
- **Firefox** — GeckoDriver, different rendering engine, native full-page screenshot support
- **Edge** — Chromium-based, nearly identical to Chrome but separate driver, Microsoft-specific capabilities
- **Safari** — limited: no headless, no CDP, no network interception, `safaridriver` only on macOS
- **Hidden topic:** `sendKeys()` event dispatch order differs between browsers — some frameworks rely on specific event sequences; test on target browsers
- **Hidden topic:** CSS rendering differences — `getCssValue()` returns different color formats (`rgb()` vs `rgba()`) or unit formats across browsers
- **Hidden topic:** screenshot dimensions/quality vary by browser — pixel-level visual comparison must account for browser-specific rendering
- **Hidden topic:** file download behavior differs — Chrome creates `.crdownload` temp files, Firefox creates `.part`; download verification must handle both

### 17.2 Driver Factory Pattern for Cross-Browser
- Accept browser name via config/parameter
- Instantiate the correct driver with browser-specific options
- `switch(browser) { case "chrome": return new ChromeDriver(chromeOptions); ... }`
- **Hidden topic:** abstract browser differences in the factory — e.g., headless argument is `--headless=new` for Chrome but `-headless` for Firefox
- **Hidden topic:** use `ThreadLocal<WebDriver>` when running browsers in parallel — each thread gets its own driver instance

---

## Phase 18: SSL, Authentication & Security Scenarios

### 18.1 SSL Certificate Handling
- `options.setAcceptInsecureCerts(true)` — W3C standard capability; handles self-signed and expired certs
- **Hidden topic:** `--ignore-certificate-errors` (Chrome flag) — broader than `acceptInsecureCerts`; ignores ALL certificate errors including revoked certificates; use only in test environments
- **Hidden topic:** corporate proxy with SSL inspection (MITM) — browser sees the proxy's certificate, not the server's; may need to add the corporate CA to the browser's trust store via profile

### 18.2 Basic HTTP Authentication
- `HasAuthentication` (Selenium 4): `((HasAuthentication) driver).register(() -> new UsernameAndPassword("user", "pass"))`
- Registers a handler that automatically fills basic auth dialogs
- **Hidden topic:** this only handles HTTP Basic/Digest auth — not form-based login, OAuth, SAML, or SSO
- **Hidden topic:** embedding credentials in the URL (`https://user:pass@example.com`) is deprecated in modern browsers and blocked by Chrome

### 18.3 Handling OAuth / SSO in Tests
- Strategy: use API calls to obtain tokens, inject them via cookies/localStorage
- Strategy: have a test-only bypass account with simplified auth
- Strategy: mock the auth provider in the test environment
- **Hidden topic:** never hardcode real credentials in test code — use environment variables, secrets managers, or encrypted config files

---

## Phase 19: Selenium Logging & Browser Logs

### 19.1 WebDriver Logging
- `LoggingPreferences prefs = new LoggingPreferences()`
- `prefs.enable(LogType.BROWSER, Level.ALL)` — JavaScript console logs
- `prefs.enable(LogType.DRIVER, Level.ALL)` — driver-level logs
- `prefs.enable(LogType.PERFORMANCE, Level.ALL)` — network/performance entries
- `options.setCapability("goog:loggingPrefs", prefs)` — Chrome
- `driver.manage().logs().get(LogType.BROWSER)` — retrieve log entries
- **Hidden topic:** W3C mode changed how logging works — some log types are only available in non-W3C mode; CDP/BiDi event listeners are the modern alternative
- **Hidden topic:** `LogType.PERFORMANCE` entries include network requests in HAR-like format — useful for performance analysis without external tools

### 19.2 Driver Service Logging
- `ChromeDriverService.Builder builder = new ChromeDriverService.Builder()`
- `builder.withVerbose(true).withLogFile(new File("chromedriver.log")).withLogOutput(System.out)`
- Pass the service to the driver: `new ChromeDriver(service, options)`
- **Hidden topic:** driver logs show the raw HTTP commands — invaluable for debugging when you suspect a Selenium client-side issue vs a driver-side issue

---

## Phase 20: Selenium 4 Migration & Version Differences

### 20.1 Key Changes from Selenium 3 to 4
- W3C WebDriver protocol is the default and only protocol
- `DesiredCapabilities` → browser-specific `Options` classes
- `FindsBy*` interfaces removed — use `findElement(By.x())` directly
- `getAttribute()` now returns the IDL property; `getDomAttribute()` added for HTML attributes
- `Actions` class rewritten for W3C compliance — offset calculations changed
- Relative locators added
- `newWindow()` API for tabs/windows
- Element screenshots
- CDP integration via `HasDevTools`
- `driver.switchTo().parentFrame()` works reliably
- Timeout setters use `Duration` instead of `long, TimeUnit`
- **Hidden topic:** Selenium 3 `Actions.moveToElement()` moved to top-left corner; Selenium 4 moves to element center — existing hover tests may break
- **Hidden topic:** Selenium 3 allowed non-W3C capabilities freely; Selenium 4 rejects unprefixed vendor caps — fix capability names during migration
- **Hidden topic:** Selenium 4 removed the built-in `EventFiringWebDriver` — replace with `EventFiringDecorator` and `WebDriverListener`

### 20.2 EventFiringDecorator (Selenium 4)
- `WebDriverListener` interface — implement before/after hooks for WebDriver events
- `beforeClick(WebElement)`, `afterClick(WebElement)`, `beforeFindElement(By)`, `afterGetText(WebElement, String)`, etc.
- `EventFiringDecorator decorator = new EventFiringDecorator(listener)`
- `WebDriver decoratedDriver = decorator.decorate(driver)`
- **Hidden topic:** replaces Selenium 3's `EventFiringWebDriver` and `WebDriverEventListener` — the new API is decorator-based rather than wrapper-based
- **Hidden topic:** listeners are useful for automatic logging, screenshot capture, element highlighting, and timing measurement — without modifying page object code

---

## Phase 21: Selenium Interview Deep Dive

### 21.1 Architecture Questions
- How does Selenium WebDriver communicate with the browser? (Client → HTTP → Driver → Browser)
- What is the W3C WebDriver Protocol? How is it different from JSON Wire Protocol?
- What is the role of ChromeDriver/GeckoDriver?
- How does Selenium Grid 4 distribute sessions across nodes?
- What is WebDriver BiDi and why does it matter?

### 21.2 Practical Selenium Questions
- `driver.close()` vs `driver.quit()` — what happens if you only call `close()`?
- `getAttribute()` vs `getDomAttribute()` vs `getDomProperty()` — when to use which?
- Implicit vs explicit vs fluent waits — what happens when you mix them?
- How do you handle stale element reference exceptions?
- How do you handle dynamic web elements with no stable locators?
- How do you interact with Shadow DOM elements?
- How does `findElement` differ when called on `WebDriver` vs `WebElement`?
- How do you upload files on Selenium Grid?
- What is `PageLoadStrategy` and how does each mode affect test performance?
- How do you capture network traffic with Selenium?

### 21.3 Scenario-Based Questions
- "Element is in the DOM but `click()` fails — what do you check?"
  → Visibility? Overlay? Frame context? Need to scroll? Need explicit wait for clickability?
- "Test passes locally but fails on Grid/CI — what do you investigate?"
  → Browser/driver version? Screen size? Timing differences? Missing fonts? Headless mode differences?
- "How do you verify a file was downloaded via Selenium?"
  → Set download directory, poll for file existence, check file size, verify content
- "How do you handle a page with multiple iframes?"
  → Switch frame by frame, interact, switch back; use `defaultContent()` vs `parentFrame()`
- "How do you test a page with lazy-loaded content?"
  → Scroll down incrementally, wait for elements to appear, repeat until all content is loaded

### 21.4 Coding Challenges
- Write a custom `ExpectedCondition` that waits for an element's text to change from a known value
- Write a method that safely clicks an element with retry logic for `StaleElementReferenceException`
- Write a utility that reads all rows from a dynamic HTML table into a `List<Map<String, String>>` (header → value)
- Write a method that waits for all AJAX calls to complete using `JavascriptExecutor`
- Write a driver factory that supports Chrome, Firefox, and Edge with configurable headless mode

---

## Quick Reference: Maven Dependency

```xml
<dependency>
    <groupId>org.seleniumhq.selenium</groupId>
    <artifactId>selenium-java</artifactId>
    <version>4.27.0</version>
</dependency>
```

---

## Study Schedule

| Phase | Duration | Priority |
|-------|----------|----------|
| Phase 1: Architecture & Internals | 3-4 days | Critical |
| Phase 2: Browser Setup & Config | 3-4 days | Critical |
| Phase 3: Locator Strategies | 1-2 weeks | Critical |
| Phase 4: Element Interactions | 1 week | Critical |
| Phase 5: Waits & Synchronization | 1 week | Critical |
| Phase 6: Windows, Frames, Alerts | 3-4 days | Critical |
| Phase 7: Actions API | 4-5 days | High |
| Phase 8: JavaScript Executor | 3-4 days | High |
| Phase 9: UI Components | 4-5 days | High |
| Phase 10: Screenshots | 1-2 days | Medium |
| Phase 11: CDP & BiDi | 3-4 days | Medium-High |
| Phase 12: Selenium Grid | 3-4 days | High |
| Phase 13: Cookies & Storage | 2-3 days | Medium |
| Phase 14: Exceptions | 2-3 days | High |
| Phase 15: POM & Page Factory | 3-4 days | Critical |
| Phase 16: Performance & Debugging | 3-4 days | High |
| Phase 17: Cross-Browser | 2-3 days | Medium-High |
| Phase 18: SSL & Authentication | 1-2 days | Medium |
| Phase 19: Logging | 1-2 days | Medium |
| Phase 20: Selenium 4 Migration | 2-3 days | Medium |
| Phase 21: Interview Deep Dive | Ongoing | Critical |

---

> **Every "hidden topic" marks knowledge that separates Selenium users from Selenium experts — the things that cause flaky tests, CI failures, and interview stumbles.**
