# DataMeter - الكود الكامل

## `settings.gradle.kts`

```kotlin
pluginManagement { repositories { google(); mavenCentral(); gradlePluginPortal() } }
dependencyResolutionManagement {
    repositoriesMode.set(RepositoriesMode.FAIL_ON_PROJECT_REPOS)
    repositories { google(); mavenCentral() }
}
rootProject.name = "DataMeter"
include(":app")
```

## `build.gradle.kts`

```kotlin
plugins {
    id("com.android.application") version "8.5.2" apply false
    id("org.jetbrains.kotlin.android") version "1.9.24" apply false
}
```

## `gradle.properties`

```properties
org.gradle.jvmargs=-Xmx2g
android.useAndroidX=true
kotlin.code.style=official
```

## `app/build.gradle.kts`

```kotlin
plugins {
    id("com.android.application")
    id("org.jetbrains.kotlin.android")
}

android {
    namespace = "com.ahmed.datameter"
    compileSdk = 34
    defaultConfig {
        applicationId = "com.ahmed.datameter"
        minSdk = 26
        targetSdk = 34
        versionCode = 1
        versionName = "1.0"
    }
    buildTypes { release { isMinifyEnabled = false } }
    compileOptions {
        sourceCompatibility = JavaVersion.VERSION_17
        targetCompatibility = JavaVersion.VERSION_17
    }
    kotlinOptions { jvmTarget = "17" }
    buildFeatures { compose = true }
    composeOptions { kotlinCompilerExtensionVersion = "1.5.14" }
}

dependencies {
    implementation(platform("androidx.compose:compose-bom:2024.06.00"))
    implementation("androidx.compose.ui:ui")
    implementation("androidx.compose.material3:material3")
    implementation("androidx.activity:activity-compose:1.9.0")
    implementation("androidx.work:work-runtime-ktx:2.9.0")
}
```

## `app/src/main/AndroidManifest.xml`

```xml
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools">

    <!-- لا توجد صلاحية INTERNET عمدًا: التطبيق لا يستطيع إرسال أي بيانات خارج الجهاز -->
    <uses-permission android:name="android.permission.PACKAGE_USAGE_STATS" tools:ignore="ProtectedPermissions" />
    <uses-permission android:name="android.permission.ACCESS_NETWORK_STATE" />
    <uses-permission android:name="android.permission.ACCESS_WIFI_STATE" />
    <uses-permission android:name="android.permission.ACCESS_FINE_LOCATION" />
    <uses-permission android:name="android.permission.ACCESS_BACKGROUND_LOCATION" />
    <uses-permission android:name="android.permission.READ_PHONE_STATE" />
    <uses-permission android:name="android.permission.RECEIVE_BOOT_COMPLETED" />

    <application
        android:allowBackup="false"
        android:label="DataMeter"
        android:supportsRtl="true"
        android:usesCleartextTraffic="false"
        android:theme="@android:style/Theme.Material.Light.NoActionBar">

        <activity android:name=".MainActivity" android:exported="true">
            <intent-filter>
                <action android:name="android.intent.action.MAIN" />
                <category android:name="android.intent.category.LAUNCHER" />
            </intent-filter>
        </activity>

        <receiver android:name=".NetReceiver" android:exported="false">
            <intent-filter>
                <action android:name="android.intent.action.BOOT_COMPLETED" />
            </intent-filter>
        </receiver>
    </application>
</manifest>
```

## `app/src/main/java/com/ahmed/datameter/Db.kt`

```kotlin
package com.ahmed.datameter

import android.content.Context
import android.database.sqlite.SQLiteDatabase
import android.database.sqlite.SQLiteOpenHelper

/** قاعدة بيانات محلية بسيطة: استهلاك يومي لكل (نوع، اسم). */
class Db private constructor(ctx: Context) : SQLiteOpenHelper(ctx, "usage.db", null, 1) {

    override fun onCreate(db: SQLiteDatabase) {
        db.execSQL(
            "CREATE TABLE usage(day INTEGER NOT NULL, kind TEXT NOT NULL, label TEXT NOT NULL, " +
                "bytes INTEGER NOT NULL, PRIMARY KEY(day, kind, label))"
        )
    }

    override fun onUpgrade(db: SQLiteDatabase, o: Int, n: Int) {}

    fun add(day: Long, kind: String, label: String, bytes: Long) {
        if (bytes <= 0) return
        val db = writableDatabase
        db.execSQL(
            "UPDATE usage SET bytes = bytes + ? WHERE day = ? AND kind = ? AND label = ?",
            arrayOf<Any?>(bytes, day, kind, label)
        )
        db.execSQL(
            "INSERT OR IGNORE INTO usage(day, kind, label, bytes) VALUES(?,?,?,?)",
            arrayOf<Any?>(day, kind, label, bytes)
        )
    }

    fun sums(kind: String, from: Long, to: Long): List<Pair<String, Long>> {
        val out = ArrayList<Pair<String, Long>>()
        readableDatabase.rawQuery(
            "SELECT label, SUM(bytes) FROM usage WHERE kind = ? AND day BETWEEN ? AND ? " +
                "GROUP BY label ORDER BY 2 DESC",
            arrayOf(kind, from.toString(), to.toString())
        ).use { c -> while (c.moveToNext()) out.add(c.getString(0) to c.getLong(1)) }
        return out
    }

    companion object {
        @Volatile private var inst: Db? = null
        fun get(ctx: Context): Db = inst ?: synchronized(this) {
            inst ?: Db(ctx.applicationContext).also { inst = it }
        }
    }
}
```

## `app/src/main/java/com/ahmed/datameter/Net.kt`

```kotlin
package com.ahmed.datameter

import android.Manifest
import android.app.AppOpsManager
import android.app.PendingIntent
import android.app.usage.NetworkStatsManager
import android.content.BroadcastReceiver
import android.content.Context
import android.content.Intent
import android.content.pm.PackageManager
import android.net.ConnectivityManager
import android.net.NetworkCapabilities
import android.net.NetworkRequest
import android.net.wifi.WifiManager
import android.os.Build
import android.os.Process
import android.telephony.SubscriptionManager
import android.telephony.TelephonyManager
import androidx.work.Worker
import androidx.work.WorkerParameters
import java.time.LocalDate

/** قراءة الأرقام من النظام نفسه (NetworkStatsManager). */
object UsageReader {
    @Suppress("DEPRECATION")
    fun hasAccess(ctx: Context): Boolean {
        val ops = ctx.getSystemService(AppOpsManager::class.java)
        val mode = if (Build.VERSION.SDK_INT >= 29)
            ops.unsafeCheckOpNoThrow(AppOpsManager.OPSTR_GET_USAGE_STATS, Process.myUid(), ctx.packageName)
        else
            ops.checkOpNoThrow(AppOpsManager.OPSTR_GET_USAGE_STATS, Process.myUid(), ctx.packageName)
        return mode == AppOpsManager.MODE_ALLOWED
    }

    @Suppress("HardwareIds", "MissingPermission")
    private fun legacySub(ctx: Context): String? = try {
        ctx.getSystemService(TelephonyManager::class.java).subscriberId
    } catch (e: Exception) { null }

    /** إجمالي (تحميل + رفع) للنوع المطلوب خلال الفترة. */
    @Suppress("DEPRECATION")
    fun total(ctx: Context, type: Int, start: Long, end: Long): Long = try {
        val nsm = ctx.getSystemService(NetworkStatsManager::class.java)
        val sub = if (type == ConnectivityManager.TYPE_MOBILE && Build.VERSION.SDK_INT < 29) legacySub(ctx) else null
        val b = nsm.querySummaryForDevice(type, sub, start, end)
        b.rxBytes + b.txBytes
    } catch (e: Exception) { 0L }
}

object Net {
    /** اسم الخط الحالي المستخدم للداتا. */
    fun simLabel(ctx: Context): String = try {
        val id = SubscriptionManager.getDefaultDataSubscriptionId()
        if (id == SubscriptionManager.INVALID_SUBSCRIPTION_ID) "SIM"
        else {
            val info = ctx.getSystemService(SubscriptionManager::class.java).getActiveSubscriptionInfo(id)
            if (info == null) "SIM" else {
                val name = info.displayName?.toString().orEmpty()
                "SIM ${info.simSlotIndex + 1}" + if (name.isNotBlank()) " - $name" else ""
            }
        }
    } catch (e: SecurityException) { "SIM" } catch (e: Exception) { "SIM" }

    /** اسم شبكة الواي فاي الحالية، أو null لو مش متصل بواي فاي. */
    @Suppress("DEPRECATION")
    fun currentSsid(ctx: Context): String? {
        val cm = ctx.getSystemService(ConnectivityManager::class.java)
        val onWifi = cm.allNetworks.any {
            cm.getNetworkCapabilities(it)?.hasTransport(NetworkCapabilities.TRANSPORT_WIFI) == true
        }
        if (!onWifi) return null
        val wm = ctx.getSystemService(Context.WIFI_SERVICE) as WifiManager
        val name = try { wm.connectionInfo?.ssid?.trim('"') } catch (e: Exception) { null }
        return if (name.isNullOrBlank() || name == "<unknown ssid>") "Wi-Fi (اسم غير معروف)" else name
    }
}

/**
 * عند كل تغيّر في الشبكة نحسب الفرق (delta) من آخر لقطة وننسبه للخط/الشبكة اللي كانت شغالة.
 */
object Tracker {
    private val lock = Any()
    private var lastRun = 0L

    fun snapshot(ctx: Context) {
        synchronized(lock) {
            if (!UsageReader.hasAccess(ctx)) return
            val now = System.currentTimeMillis()
            if (now - lastRun < 2000) return
            lastRun = now
            val app = ctx.applicationContext
            val p = app.getSharedPreferences("tracker", Context.MODE_PRIVATE)
            @Suppress("DEPRECATION")
            val mobile = UsageReader.total(app, ConnectivityManager.TYPE_MOBILE, 0, now)
            @Suppress("DEPRECATION")
            val wifi = UsageReader.total(app, ConnectivityManager.TYPE_WIFI, 0, now)
            val db = Db.get(app)
            val day = LocalDate.now().toEpochDay()
            val lastM = p.getLong("m", -1L)
            val lastW = p.getLong("w", -1L)
            val simNow = Net.simLabel(app)
            val ssidNow = Net.currentSsid(app)
            if (lastM >= 0 && mobile > lastM) db.add(day, "SIM", p.getString("sim", null) ?: simNow, mobile - lastM)
            if (lastW >= 0 && wifi > lastW) db.add(day, "WIFI", p.getString("ssid", null) ?: "Wi-Fi (اسم غير معروف)", wifi - lastW)
            p.edit().putLong("m", mobile).putLong("w", wifi)
                .putString("sim", simNow).putString("ssid", ssidNow).apply()
        }
    }
}

/** تسجيل مراقبة الشبكة عن طريق PendingIntent: شغّال حتى لو التطبيق مقفول، ومن غير Service دايم. */
object Watcher {
    const val ACTION = "com.ahmed.datameter.NET_EVENT"

    fun register(ctx: Context) {
        val app = ctx.applicationContext
        val cm = app.getSystemService(ConnectivityManager::class.java)
        listOf(NetworkCapabilities.TRANSPORT_WIFI to 1, NetworkCapabilities.TRANSPORT_CELLULAR to 2)
            .forEach { (transport, code) ->
                val req = NetworkRequest.Builder().addTransportType(transport).build()
                val pi = PendingIntent.getBroadcast(
                    app, code,
                    Intent(app, NetReceiver::class.java).setAction(ACTION),
                    PendingIntent.FLAG_UPDATE_CURRENT or PendingIntent.FLAG_MUTABLE
                )
                try { cm.registerNetworkCallback(req, pi) } catch (e: Exception) { }
            }
    }
}

class NetReceiver : BroadcastReceiver() {
    override fun onReceive(ctx: Context, intent: Intent) {
        val pending = goAsync()
        val app = ctx.applicationContext
        Thread {
            try {
                if (intent.action == Intent.ACTION_BOOT_COMPLETED) Watcher.register(app)
                Tracker.snapshot(app)
            } finally { pending.finish() }
        }.start()
    }
}

/** شبكة أمان: لقطة كل 15 دقيقة. */
class SnapWorker(c: Context, p: WorkerParameters) : Worker(c, p) {
    override fun doWork(): Result {
        Watcher.register(applicationContext)
        Tracker.snapshot(applicationContext)
        return Result.success()
    }
}
```

## `app/src/main/java/com/ahmed/datameter/MainActivity.kt`

```kotlin
package com.ahmed.datameter

import android.Manifest
import android.content.Context
import android.content.Intent
import android.content.pm.PackageManager
import android.net.ConnectivityManager
import android.os.Build
import android.os.Bundle
import android.provider.Settings
import androidx.activity.ComponentActivity
import androidx.activity.compose.setContent
import androidx.activity.result.contract.ActivityResultContracts
import androidx.compose.foundation.layout.*
import androidx.compose.foundation.lazy.LazyColumn
import androidx.compose.material3.*
import androidx.compose.runtime.*
import androidx.compose.ui.Modifier
import androidx.compose.ui.platform.LocalContext
import androidx.compose.ui.platform.LocalLayoutDirection
import androidx.compose.ui.unit.LayoutDirection
import androidx.compose.ui.unit.dp
import androidx.core.content.ContextCompat
import androidx.work.ExistingPeriodicWorkPolicy
import androidx.work.PeriodicWorkRequestBuilder
import androidx.work.WorkManager
import kotlinx.coroutines.Dispatchers
import kotlinx.coroutines.withContext
import java.time.Instant
import java.time.LocalDate
import java.time.ZoneId
import java.time.ZoneOffset
import java.util.Locale
import java.util.concurrent.TimeUnit

data class Report(
    val mobile: Long, val wifi: Long,
    val sims: List<Pair<String, Long>>, val ssids: List<Pair<String, Long>>
)

@Suppress("DEPRECATION")
fun compute(ctx: Context, from: LocalDate, to: LocalDate): Report {
    val z = ZoneId.systemDefault()
    val s = from.atStartOfDay(z).toInstant().toEpochMilli()
    val e = minOf(System.currentTimeMillis(), to.plusDays(1).atStartOfDay(z).toInstant().toEpochMilli())
    val db = Db.get(ctx)
    return Report(
        UsageReader.total(ctx, ConnectivityManager.TYPE_MOBILE, s, e),
        UsageReader.total(ctx, ConnectivityManager.TYPE_WIFI, s, e),
        db.sums("SIM", from.toEpochDay(), to.toEpochDay()),
        db.sums("WIFI", from.toEpochDay(), to.toEpochDay())
    )
}

fun fmt(b: Long): String {
    val u = arrayOf("B", "KB", "MB", "GB", "TB")
    var v = b.toDouble(); var i = 0
    while (v >= 1024 && i < 4) { v /= 1024; i++ }
    return if (i == 0) "$b B" else String.format(Locale.US, "%.2f %s", v, u[i])
}

fun missingRuntime(ctx: Context): List<String> {
    val need = mutableListOf(Manifest.permission.ACCESS_FINE_LOCATION, Manifest.permission.READ_PHONE_STATE)
    if (Build.VERSION.SDK_INT >= 29) need.add(Manifest.permission.ACCESS_BACKGROUND_LOCATION)
    return need.filter { ContextCompat.checkSelfPermission(ctx, it) != PackageManager.PERMISSION_GRANTED }
}

class MainActivity : ComponentActivity() {
    private var tick by mutableIntStateOf(0)

    private val permsLauncher = registerForActivityResult(ActivityResultContracts.RequestMultiplePermissions()) {
        tick++; askBackground()
    }
    private val bgLauncher = registerForActivityResult(ActivityResultContracts.RequestPermission()) { tick++ }

    private fun askBackground() {
        if (Build.VERSION.SDK_INT >= 29 &&
            ContextCompat.checkSelfPermission(this, Manifest.permission.ACCESS_FINE_LOCATION) == PackageManager.PERMISSION_GRANTED &&
            ContextCompat.checkSelfPermission(this, Manifest.permission.ACCESS_BACKGROUND_LOCATION) != PackageManager.PERMISSION_GRANTED
        ) bgLauncher.launch(Manifest.permission.ACCESS_BACKGROUND_LOCATION)
    }

    private fun askPerms() = permsLauncher.launch(
        arrayOf(Manifest.permission.ACCESS_FINE_LOCATION, Manifest.permission.READ_PHONE_STATE)
    )

    private fun openUsage() = startActivity(Intent(Settings.ACTION_USAGE_ACCESS_SETTINGS))

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        Watcher.register(this)
        WorkManager.getInstance(this).enqueueUniquePeriodicWork(
            "snap", ExistingPeriodicWorkPolicy.KEEP,
            PeriodicWorkRequestBuilder<SnapWorker>(15, TimeUnit.MINUTES).build()
        )
        setContent {
            MaterialTheme {
                CompositionLocalProvider(LocalLayoutDirection provides LayoutDirection.Rtl) {
                    Surface(Modifier.fillMaxSize()) {
                        Screen(tick, { openUsage() }, { askPerms() })
                    }
                }
            }
        }
    }

    override fun onResume() { super.onResume(); tick++ }
}

private fun utcDate(ms: Long): LocalDate = Instant.ofEpochMilli(ms).atZone(ZoneOffset.UTC).toLocalDate()

@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun Screen(tick: Int, onUsage: () -> Unit, onPerms: () -> Unit) {
    val ctx = LocalContext.current.applicationContext
    var period by remember { mutableIntStateOf(0) }
    var cFrom by remember { mutableStateOf(LocalDate.now()) }
    var cTo by remember { mutableStateOf(LocalDate.now()) }
    var refresh by remember { mutableIntStateOf(0) }
    var report by remember { mutableStateOf<Report?>(null) }
    var picker by remember { mutableStateOf(false) }

    val today = LocalDate.now()
    val (from, to) = when (period) {
        0 -> today to today
        1 -> today.withDayOfMonth(1) to today
        else -> cFrom to minOf(cTo, today)
    }

    LaunchedEffect(period, cFrom, cTo, refresh, tick) {
        report = withContext(Dispatchers.IO) {
            Tracker.snapshot(ctx)
            compute(ctx, from, to)
        }
    }

    LazyColumn(
        Modifier.fillMaxSize().systemBarsPadding().padding(16.dp),
        verticalArrangement = Arrangement.spacedBy(12.dp)
    ) {
        item { Text("عدّاد الاستهلاك", style = MaterialTheme.typography.headlineMedium) }
        if (!UsageReader.hasAccess(ctx))
            item { Warn("لازم تفعّل «الوصول لبيانات الاستخدام» للتطبيق", "فتح الإعدادات", onUsage) }
        if (missingRuntime(ctx).isNotEmpty())
            item { Warn("محتاج صلاحيات الموقع والهاتف (لقراءة اسم الواي فاي واسم الخط)، واختار «السماح طوال الوقت» للموقع", "منح الصلاحيات", onPerms) }
        item {
            Row(horizontalArrangement = Arrangement.spacedBy(8.dp)) {
                FilterChip(selected = period == 0, onClick = { period = 0 }, label = { Text("اليوم") })
                FilterChip(selected = period == 1, onClick = { period = 1 }, label = { Text("الشهر") })
                FilterChip(selected = period == 2, onClick = { period = 2; picker = true }, label = { Text("فترة مخصصة") })
            }
        }
        if (period == 2) item { OutlinedButton(onClick = { picker = true }) { Text("$cFrom  ←  $cTo") } }
        report?.let { r ->
            item { Block("داتا الموبايل — الإجمالي", r.mobile, r.sims, "حسب الخط") }
            item { Block("الواي فاي — الإجمالي", r.wifi, r.ssids, "حسب الشبكة") }
        }
        item {
            Text(
                "الإجمالي بيتقرأ من النظام نفسه (دقيق ومن قبل التثبيت). تفاصيل كل خط وكل شبكة بتبدأ من وقت أول تشغيل للتطبيق.",
                style = MaterialTheme.typography.bodySmall
            )
        }
        item { OutlinedButton(onClick = { refresh++ }) { Text("تحديث") } }
    }

    if (picker) {
        val st = rememberDateRangePickerState()
        DatePickerDialog(
            onDismissRequest = { picker = false },
            confirmButton = {
                TextButton(onClick = {
                    val a = st.selectedStartDateMillis
                    val b = st.selectedEndDateMillis ?: a
                    if (a != null && b != null) { cFrom = utcDate(a); cTo = utcDate(b) }
                    picker = false
                }) { Text("تم") }
            },
            dismissButton = { TextButton(onClick = { picker = false }) { Text("إلغاء") } }
        ) { DateRangePicker(state = st, modifier = Modifier.height(500.dp)) }
    }
}

@Composable
fun Warn(msg: String, btn: String, onClick: () -> Unit) {
    Card(colors = CardDefaults.cardColors(containerColor = MaterialTheme.colorScheme.errorContainer)) {
        Column(Modifier.padding(16.dp), verticalArrangement = Arrangement.spacedBy(8.dp)) {
            Text(msg)
            Button(onClick = onClick) { Text(btn) }
        }
    }
}

@Composable
fun Block(title: String, total: Long, items: List<Pair<String, Long>>, sub: String) {
    Card(Modifier.fillMaxWidth()) {
        Column(Modifier.padding(16.dp), verticalArrangement = Arrangement.spacedBy(6.dp)) {
            Text(title, style = MaterialTheme.typography.titleMedium)
            Text(fmt(total), style = MaterialTheme.typography.headlineSmall)
            if (items.isNotEmpty()) {
                HorizontalDivider()
                Text(sub, style = MaterialTheme.typography.labelLarge)
                items.forEach {
                    Row(Modifier.fillMaxWidth(), horizontalArrangement = Arrangement.SpaceBetween) {
                        Text(it.first, Modifier.weight(1f)); Text(fmt(it.second))
                    }
                }
            }
        }
    }
}
```

## `.github/workflows/build.yml`

```yaml
name: Build APK
on:
  workflow_dispatch:
  push:
    branches: [main]
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: 17
      - uses: gradle/actions/setup-gradle@v4
        with:
          gradle-version: "8.7"
      - run: gradle assembleDebug --no-daemon
      - uses: actions/upload-artifact@v4
        with:
          name: DataMeter-apk
          path: app/build/outputs/apk/debug/app-debug.apk
```

## `README.md`

```markdown
# DataMeter
طريقة الحصول على الـ APK (من غير ما تثبّت حاجة على الكمبيوتر):
1. اعمل مستودع (Repository) خاص على GitHub وارفع محتوى المجلد كله (بما فيه مجلد .github).
2. من تبويب Actions اختار "Build APK" ثم Run workflow.
3. بعد دقائق نزّل الملف DataMeter-apk من قسم Artifacts، وفك الضغط، وثبّت app-debug.apk.
بديل: افتح المجلد في Android Studio واختار Build > Build APK(s).
بعد التثبيت: فعّل "الوصول لبيانات الاستخدام" ومنح صلاحيات الموقع (السماح طوال الوقت) والهاتف.
```
