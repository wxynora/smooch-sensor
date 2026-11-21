# smooch-sensor

让你的机感知到你的啵啵 💋

在 Android 手机上识别「亲了一下屏幕」，还能知道亲在了哪儿。不用摄像头，不用特殊权限，传感器数据不出手机。

> v1：只在一台小米手机上实测过，换机型请先看[已知限制](#九已知限制)和[调试](#十调试怎么定自己手机的参数)。

---

## 一、原理

手机没有嘴唇传感器，但亲屏幕时会同时发生两件事：

1. **嘴唇贴上屏幕**：触摸屏会报告 **两个及以上** 触点（上下唇各算一个，或者接触面大被拆成多点）。
2. **脸挡住屏幕顶部的环境光传感器**：光感读数突然掉到 **0 lux**。

只有第 1 件，可能是两根手指；只有第 2 件，可能是手挡住了光感或者手机进了口袋。**两件事在同一时间段里一起出现**，才算亲吻。

用到的都是公开 API：`MotionEvent`（多点触控）和 `Sensor.TYPE_LIGHT`（环境光）。

---

## 二、两路信号

### 触点数

一次触碰从 `ACTION_DOWN`（第一个触点按下）开始，到 `ACTION_UP` / `ACTION_CANCEL`（最后一个触点离开）结束。期间每个 `MotionEvent` 都看一眼 `pointerCount`，记下这次触碰的 **峰值**。

- 峰值 ≥ 2：可能是亲吻
- 峰值 = 1：不是亲吻

### 环境光

页面可见时注册光感，离开页面就注销：

```kotlin
val sensors = context.getSystemService(SensorManager::class.java)
val lightSensor = sensors?.getDefaultSensor(Sensor.TYPE_LIGHT)

val listener = object : SensorEventListener {
    override fun onSensorChanged(event: SensorEvent) = onLux(event.values[0])
    override fun onAccuracyChanged(sensor: Sensor, accuracy: Int) = Unit
}

// onResume
val lightAvailable = lightSensor != null &&
    sensors.registerListener(listener, lightSensor, SensorManager.SENSOR_DELAY_GAME)

// onPause
sensors?.unregisterListener(listener)
```

`SENSOR_DELAY_GAME` 比默认频率高，能更快收到「变暗」。

只认 **「从亮变成 0」** 这个瞬间：上一次读数 > 0，这一次 == 0。一直是 0（比如房间本来就黑）不算变暗。

---

## 三、时间窗：光感比触摸慢

最容易踩的坑：**光感上报比触摸事件慢**。实测嘴唇碰到屏幕后，第一次 0 lux 的上报最晚晚了 **448 ms**。所以不能要求「触摸那一刻光感正好是 0」，而是给 **500 ms 的对齐窗口**，覆盖三种先后顺序：

| 情况 | 先后顺序 | 怎么判 |
|---|---|---|
| A | 先变暗，后贴上 | 按下时光感已经是 0，而且变暗发生在按下前 500 ms 内 |
| B | 贴着的时候变暗 | 接触期间收到「亮→0」 |
| C | 松开以后才变暗 | 松开时先不下结论，再等 500 ms；等到「亮→0」就算亲吻，等不到就不算 |

情况 C 很常见：亲得快，嘴已经离开了，变暗才报上来。

---

## 四、判定状态机

和界面无关的小类，方便单独测试：

```kotlin
enum class KissResult { NOT_KISS, PENDING, KISS, CANCELLED }

const val KISS_LIGHT_WINDOW_MS = 500L

class KissGesture {
    private var lastLux = Float.NaN
    private var darkenedAt: Long? = null          // 没在接触时，最近一次「亮→0」的时间
    private var darkenedDuringContact = false
    private var peakContacts = 0
    private var touching = false
    private var releasedAt: Long? = null          // 情况 C：松开后在等光感

    /** 每次光感回调都调用。返回 true = 松开后等到了变暗，此刻判为亲吻。 */
    fun observeLight(lux: Float, now: Long): Boolean {
        if (!lux.isFinite() || lux < 0f) return false
        val darkened = lux == 0f && lastLux > 0f
        lastLux = lux
        if (!darkened) return false
        if (touching) {
            darkenedDuringContact = true                       // 情况 B
        } else {
            val release = releasedAt
            if (release != null && now - release in 0..KISS_LIGHT_WINDOW_MS) {
                releasedAt = null
                return true                                    // 情况 C
            }
            darkenedAt = now                                   // 也许是情况 A 的前半段
        }
        return false
    }

    /** ACTION_DOWN 时调用。 */
    fun beginTouch(now: Long) {
        darkenedDuringContact = lastLux == 0f &&
            darkenedAt?.let { now - it in 0..KISS_LIGHT_WINDOW_MS } == true   // 情况 A
        darkenedAt = null
        releasedAt = null
        peakContacts = 0
        touching = true
    }

    /** 每个 MotionEvent 都调用。 */
    fun observeContacts(count: Int) {
        if (touching) peakContacts = maxOf(peakContacts, count)
    }

    /** ACTION_UP / ACTION_CANCEL 时调用。 */
    fun finishTouch(now: Long, cancelled: Boolean, lightAvailable: Boolean): KissResult {
        touching = false
        return when {
            cancelled -> KissResult.CANCELLED
            peakContacts < 2 || !lightAvailable -> KissResult.NOT_KISS
            darkenedDuringContact -> KissResult.KISS
            else -> { releasedAt = now; KissResult.PENDING }   // 去等情况 C
        }
    }

    /** 等待超时或页面离开时调用。 */
    fun endPending() { releasedAt = null }

    fun reset() {
        lastLux = Float.NaN
        darkenedAt = null
        touching = false
        releasedAt = null
    }
}
```

---

## 五、接到页面上

```kotlin
val kiss = KissGesture()
var pendingLight: Job? = null

fun onKiss(contacts: List<PointF>) {
    pendingLight?.cancel()
    pendingLight = null
    // 识别到了：在这里算部位、生成要发给你的机的内容（见第八节）
}

fun onLux(lux: Float) {
    if (kiss.observeLight(lux, SystemClock.elapsedRealtime())) onKiss(lastContacts)
}

view.setOnTouchListener { _, event ->
    val now = SystemClock.elapsedRealtime()
    when (event.actionMasked) {
        MotionEvent.ACTION_DOWN -> {
            pendingLight?.cancel(); kiss.endPending()     // 新的一次开始，上一次不等了
            contacts.clear()
            kiss.beginTouch(now)
        }
    }
    kiss.observeContacts(event.pointerCount)
    // 按下时记录每个触点的起始位置，用来算部位
    if (event.actionMasked == MotionEvent.ACTION_DOWN ||
        event.actionMasked == MotionEvent.ACTION_POINTER_DOWN) {
        contacts += PointF(event.getX(event.actionIndex), event.getY(event.actionIndex))
    }
    val finished = event.actionMasked == MotionEvent.ACTION_UP ||
        event.actionMasked == MotionEvent.ACTION_CANCEL
    if (finished) {
        val cancelled = event.actionMasked == MotionEvent.ACTION_CANCEL
        lastContacts = contacts.toList()
        when (kiss.finishTouch(now, cancelled, lightAvailable)) {
            KissResult.KISS -> onKiss(lastContacts)
            KissResult.PENDING -> pendingLight = scope.launch {
                delay(KISS_LIGHT_WINDOW_MS)
                kiss.endPending()                         // 500 ms 没等到变暗，不是亲吻
                // 想要普通触摸的话，在这里结算（见第六节）
            }
            KissResult.NOT_KISS -> Unit                   // 想要普通触摸的话，也在这里结算
            else -> Unit
        }
    }
    true
}
```

要点：

- 用 `SystemClock.elapsedRealtime()` 给触摸和光感打同一套时间戳。
- 在 `ACTION_DOWN` 里 `requestDisallowInterceptTouchEvent(true)`，免得外层滚动容器把触点抢走。
- 页面暂停时调用 `kiss.reset()`，取消等待中的 `pendingLight`。

---

## 六、顺手：不是亲吻时，分成点按 / 贴住 / 抚摸

不是亲吻的触碰也可以顺手认出来，发给你的机的时候更丰富。每个触点各自分类，看两样东西：**按了多久**、**最远挪了多远**。

距离换算成 **毫米**，不同屏幕密度下手感一致：

```kotlin
class FingerContact(val start: PointF, val startTime: Long) {
    var reachMm = 0.0
        private set

    fun sample(x: Float, y: Float, xdpi: Float, ydpi: Float) {
        val dx = (x - start.x) * 25.4 / xdpi
        val dy = (y - start.y) * 25.4 / ydpi
        reachMm = maxOf(reachMm, hypot(dx, dy))
    }
}

enum class Gesture { TAP, HOLD, STROKE }

fun classify(durationMs: Long, reachMm: Double) = when {
    reachMm >= 3.0    -> Gesture.STROKE   // 抚摸：挪了 3 mm 以上
    durationMs >= 350 -> Gesture.HOLD     // 贴住：按了 350 ms 以上
    else              -> Gesture.TAP      // 点一下
}
```

接线要点：

- `ACTION_DOWN` / `ACTION_POINTER_DOWN` 时给这个触点建一个 `FingerContact`（按 `getPointerId` 存）。
- 每个 `ACTION_MOVE` 都调用 `sample`，**连 `getHistoricalX/Y` 的历史点一起采**，否则快速滑动时中间点会丢，挪动距离偏小。
- 触点抬起时用 `classify(抬起时间 - startTime, reachMm)` 得出动作；起点和终点各查一次区域，抚摸就可以写成「从脸抚到下巴」。
- **普通触摸不要马上发**：这次触碰可能还在等光感（情况 C），先攒着；判定为亲吻就丢掉它们，等待超时或判定不是亲吻再把它们算进去。

文案可以这样拼：

| 动作 | 有部位 | 没部位 |
|---|---|---|
| 点 | 用指尖轻轻碰了一下你的脸 | 用指尖轻轻碰了一下屏幕 |
| 贴住 | 用指腹贴着你的脸，停了一会儿 | 用指腹贴着屏幕，停了一会儿 |
| 抚摸 | 用指腹从你的脸轻轻抚到你的下巴 | 用指腹轻轻抚过屏幕 |

---

## 七、随意选图、划区域

想让亲吻落在 TA 身上具体的地方，就让用户自己选一张图，在图上圈出「脸」「额头」「手」……

### 1. 选图并裁成屏幕比例

用系统选图器拿到图片：

```kotlin
val pickImage = rememberLauncherForActivityResult(ActivityResultContracts.GetContent()) { uri ->
    if (uri != null) cropDraft = decodeBitmap(context, uri)   // 进入裁剪页
}
pickImage.launch("image/*")
```

裁剪页是一个 **和屏幕一样大** 的取景框，用户拖动、双指缩放，确认时只保存取景框里的那一块。这样保存下来的图和屏幕比例一致，触摸页里可以直接铺满全屏。

取景框对应原图的哪一块，按「铺满」的缩放来算：

```kotlin
// zoom >= 1，offset 是用户拖动的位移（屏幕像素）
val baseScale = max(viewW / imgW.toFloat(), viewH / imgH.toFloat())   // 铺满屏幕的最小缩放
val scale = baseScale * zoom
val cropW = (viewW / scale).coerceAtMost(imgW.toFloat())
val cropH = (viewH / scale).coerceAtMost(imgH.toFloat())
val centerX = (imgW / 2f - offset.x / scale).coerceIn(cropW / 2, imgW - cropW / 2)
val centerY = (imgH / 2f - offset.y / scale).coerceIn(cropH / 2, imgH - cropH / 2)
val crop = RectF(centerX - cropW / 2, centerY - cropH / 2, centerX + cropW / 2, centerY + cropH / 2)
```

拖动时也要把 `offset` 限制在图片不露边的范围里：

```kotlin
val maxX = max(0f, (imgW * scale - viewW) / 2f)
val maxY = max(0f, (imgH * scale - viewH) / 2f)
offset = Offset(offset.x.coerceIn(-maxX, maxX), offset.y.coerceIn(-maxY, maxY))
```

确认后用 `Bitmap.createBitmap(bitmap, left, top, width, height)` 切出来，存成 PNG 放进 `context.filesDir`，路径记在 `SharedPreferences` 里。换图时顺手删掉旧图和旧区域，因为旧区域在新图上已经对不上了。

### 2. 划区域

做一个「编辑」模式：

1. 手指在图上按下、拖动、松开，围出一个矩形。
2. 弹出输入框给它起名字（脸、额头、手……）。
3. 已有区域可以点开改名或删除。

区域用 **相对图片的 0~1 比例坐标** 保存，不存屏幕像素，这样换手机、换分辨率都还对得上：

```kotlin
data class Region(val name: String, val left: Float, val top: Float, val right: Float, val bottom: Float) {
    fun contains(x: Float, y: Float) = x in left..right && y in top..bottom
}

fun regionOf(a: PointF, b: PointF) = Region(
    name = "",
    left = minOf(a.x, b.x), top = minOf(a.y, b.y),
    right = maxOf(a.x, b.x), bottom = maxOf(a.y, b.y),
)
```

存成 JSON 数组放 `SharedPreferences` 就够了：

```json
[{"name":"脸","left":0.31,"top":0.18,"right":0.69,"bottom":0.45}]
```

### 3. 屏幕坐标 ↔ 图片比例坐标

图片在触摸页里按「铺满并居中」显示，先算出图片实际占的矩形：

```kotlin
fun imageBounds(viewW: Float, viewH: Float, imgW: Int, imgH: Int): RectF {
    val scale = max(viewW / imgW, viewH / imgH)
    val w = imgW * scale
    val h = imgH * scale
    val left = (viewW - w) / 2
    val top = (viewH - h) / 2
    return RectF(left, top, left + w, top + h)
}

fun RectF.normalize(p: PointF) = PointF((p.x - left) / width(), (p.y - top) / height())
fun RectF.denormalize(p: PointF) = PointF(left + p.x * width(), top + p.y * height())
```

触摸时用 `normalize` 把触点换成 0~1，画区域框时用 `denormalize` 换回屏幕坐标。**裁剪、显示、划区域、命中判断全部走同一套换算**，部位才不会偏。

区域重叠时，后画的优先：

```kotlin
fun List<Region>.hit(x: Float, y: Float) = lastOrNull { it.contains(x, y) }?.name
```

---

## 八、亲在哪里

亲吻是两个以上触点，取 **第一个落在区域里的触点** 当部位。两片唇通常落在同一个区域，取第一个就够了：

```kotlin
val part = contacts
    .map { bounds.normalize(it) }
    .firstNotNullOfOrNull { regions.hit(it.x, it.y) }

val text = if (part != null) "亲了一下你的$part" else "亲了你一下"
```

接下来怎么让你的机收到，看你自己的聊天方式。建议把它当作一个 **动作** 发过去，比如加括号：

```
（亲了一下你的脸。）
```

而不是当成你打的一句话。

---

## 九、已知限制

- **没有光感的设备**：直接判为不是亲吻，不会误报。
- **系统取消了这次触碰**（`ACTION_CANCEL`）：什么都不记。
- **只认正好 0 lux**：有些机型被遮住时读数是 1~5 lux，掉不到 0；房间本来就很暗时一直是 0，没有「亮→0」。这两种都识别不到，换机型最先要调的就是这条。
- **500 ms 只在一台小米上测过**：光感延迟因机型而异。
- **脸凑得太慢**：先变暗、过了 500 ms 以上才贴上屏幕，会被情况 A 漏掉。
- **光感一般在屏幕顶部**（听筒或前摄附近）：亲屏幕中下部时，脸不一定能挡住它。
- 两根手指 + 手掌恰好挡住光感，理论上也会被当成亲吻，实际很少见。

---

## 十、调试：怎么定自己手机的参数

先做一个只打日志的页面，同时记录：

- 每个 `MotionEvent` 的时间、`actionMasked`、`pointerCount`
- 每次光感回调的时间和 lux 值

然后在自己手机上：真亲几次、两根手指按几次、手掌挡一下光感、在暗一点的房间再亲几次。对着日志看三类动作的时间线差在哪里，再定阈值和窗口。不要直接照搬上面的数字。

---

## License

MIT
