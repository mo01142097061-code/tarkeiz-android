tarkeiz-android/
├── .github/
│   └── workflows/
│       └── android.yml
└── focusapp/
    ├── settings.gradle.kts
    ├── build.gradle.kts
    ├── gradle.properties
    └── app/
        ├── build.gradle.kts
        └── src/
            └── main/
                ├── AndroidManifest.xml
                ├── java/com/tarkeiz/focus/
                │   ├── MainActivity.kt
                │   ├── AppBlockService.kt
                │   └── BlockedActivity.kt
                └── res/
                    ├── values/styles.xml
                    └── xml/accessibility_service_config.xmlpluginManagement {
    repositories {
        google()
        mavenCentral()
        gradlePluginPortal()
    }
}

dependencyResolutionManagement {
    repositoriesMode.set(
        RepositoriesMode.FAIL_ON_PROJECT_REPOS
    )
    repositories {
        google()
        mavenCentral()
    }
}

rootProject.name = "Tarkeiz"
include(":app")plugins {
    id("com.android.application")
    id("org.jetbrains.kotlin.android")
}

android {
    namespace = "com.tarkeiz.focus"
    compileSdk = 35

    defaultConfig {
        applicationId = "com.tarkeiz.focus"
        minSdk = 26
        targetSdk = 35
        versionCode = 1
        versionName = "1.0"
    }
}plugins {
    id("com.android.application") version "8.7.3" apply false
    id("org.jetbrains.kotlin.android") version "2.0.21" apply false
}org.gradle.jvmargs=-Xmx2048m -Dfile.encoding=UTF-8
android.useAndroidX=true
kotlin.code.style=official<resources>
    <style
        name="AppTheme"
        parent="android:style/Theme.Material.Light.NoActionBar">

        <item name="android:fontFamily">sans</item>
        <item name="android:colorAccent">#2563EB</item>
        <item name="android:statusBarColor">#102A43</item>
        <item name="android:navigationBarColor">#102A43</item>
        <item name="android:windowLightStatusBar">false</item>

    </style>
</resources><manifest xmlns:android="http://schemas.android.com/apk/res/android">

    <uses-permission
        android:name="android.permission.QUERY_ALL_PACKAGES" />

    <application
        android:allowBackup="false"
        android:label="تركيز"
        android:theme="@style/AppTheme"
        android:supportsRtl="true">

        <activity
            android:name=".BlockedActivity"
            android:exported="false"
            android:excludeFromRecents="true" />

        <activity
            android:name=".MainActivity"
            android:exported="true">

            <intent-filter>
                <action android:name="android.intent.action.MAIN" />
                <category android:name="android.intent.category.LAUNCHER" />
            </intent-filter>

        </activity>

        <service
            android:name=".AppBlockService"
            android:label="تركيز - حظر التطبيقات"
            android:permission="android.permission.BIND_ACCESSIBILITY_SERVICE"
            android:exported="true">

            <intent-filter>
                <action
                    android:name="android.accessibilityservice.AccessibilityService" />
            </intent-filter>

            <meta-data
                android:name="android.accessibilityservice"
                android:resource="@xml/accessibility_service_config" />

        </service>

    </application>
</manifest><accessibility-service
    xmlns:android="http://schemas.android.com/apk/res/android"
    android:accessibilityEventTypes="typeWindowStateChanged|typeWindowsChanged"
    android:accessibilityFeedbackType="feedbackGeneric"
    android:notificationTimeout="100"
    android:canRetrieveWindowContent="false"
    android:description="يراقب التطبيق اسم التطبيق المفتوح أثناء جلسة التركيز ليمنع التطبيقات غير المسموح بها." />package com.tarkeiz.focus

import android.accessibilityservice.AccessibilityService
import android.content.Intent
import android.view.accessibility.AccessibilityEvent

class AppBlockService : AccessibilityService() {

    private var lastBlockedPackage: String? = null

    override fun onAccessibilityEvent(
        event: AccessibilityEvent?
    ) {

        if (
            event == null ||
            (
                event.eventType !=
                    AccessibilityEvent.TYPE_WINDOW_STATE_CHANGED &&
                event.eventType !=
                    AccessibilityEvent.TYPE_WINDOWS_CHANGED
            )
        ) {
            return
        }

        val pkg = event.packageName?.toString() ?: return

        val prefs = getSharedPreferences(
            "tarkeiz",
            MODE_PRIVATE
        )

        val endAt = prefs.getLong("endAt", 0L)

        if (endAt <= System.currentTimeMillis()) {

            if (endAt != 0L) {
                prefs.edit().putLong("endAt", 0L).apply()
            }

            lastBlockedPackage = null
            return
        }

        val allowed = prefs.getStringSet(
            "allowed",
            emptySet()
        ) ?: emptySet()

        if (
            pkg == packageName ||
            allowed.contains(pkg)
        ) {
            lastBlockedPackage = null
            return
        }

        if (lastBlockedPackage != pkg) {

            lastBlockedPackage = pkg

            val intent = Intent(
                this,
                BlockedActivity::class.java
            ).apply {

                addFlags(
                    Intent.FLAG_ACTIVITY_NEW_TASK or
                        Intent.FLAG_ACTIVITY_CLEAR_TOP or
                        Intent.FLAG_ACTIVITY_NO_HISTORY
                )

                putExtra("endAt", endAt)
            }

            startActivity(intent)
        }
    }

    override fun onInterrupt() {}
}package com.tarkeiz.focus

import android.accessibilityservice.AccessibilityServiceInfo
import android.app.Activity
import android.app.AlertDialog
import android.content.Context
import android.content.Intent
import android.graphics.Color
import android.graphics.Typeface
import android.os.Bundle
import android.provider.Settings
import android.view.Gravity
import android.view.ViewGroup
import android.view.accessibility.AccessibilityManager
import android.widget.*

class MainActivity : Activity() {

    private val prefs by lazy {
        getSharedPreferences("tarkeiz", Context.MODE_PRIVATE)
    }

    private val selected = linkedSetOf<String>()
    private var durationMinutes = 60
    private lateinit var status: TextView

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        selected.addAll(
            prefs.getStringSet("allowed", emptySet()) ?: emptySet()
        )
        selected.add(packageName)

        val root = LinearLayout(this).apply {
            orientation = LinearLayout.VERTICAL
            setPadding(28, 28, 28, 24)
            setBackgroundColor(Color.rgb(245, 248, 252))
            layoutDirection = LinearLayout.LayoutDirection.RTL
        }

        val title = TextView(this).apply {
            text = "تركيز"
            textSize = 32f
            setTextColor(Color.rgb(16, 42, 67))
            typeface = Typeface.DEFAULT_BOLD
            gravity = Gravity.CENTER
        }

        val subtitle = TextView(this).apply {
            text = "وقتك أغلى من أن يضيع بلا هدف"
            textSize = 16f
            setTextColor(Color.DKGRAY)
            gravity = Gravity.CENTER
            setPadding(0, 8, 0, 24)
        }

        status = TextView(this).apply {
            textSize = 14f
            setTextColor(Color.rgb(90, 100, 115))
            gravity = Gravity.CENTER
            setPadding(0, 0, 0, 16)
        }

        val durationLabel = TextView(this).apply {
            text = "اختر مدة التركيز"
            textSize = 18f
            setTextColor(Color.BLACK)
        }

        val duration = Spinner(this)

        val labels = listOf(
            "30 دقيقة",
            "ساعة واحدة",
            "ساعتان",
            "3 ساعات",
            "5 ساعات",
            "8 ساعات"
        )

        val values = listOf(30, 60, 120, 180, 300, 480)

        duration.adapter = ArrayAdapter(
            this,
            android.R.layout.simple_spinner_dropdown_item,
            labels
        )

        duration.setSelection(1)

        duration.onItemSelectedListener =
            object : AdapterView.OnItemSelectedListener {

                override fun onNothingSelected(
                    parent: AdapterView<*>?
                ) {}

                override fun onItemSelected(
                    parent: AdapterView<*>?,
                    view: android.view.View?,
                    position: Int,
                    id: Long
                ) {
                    durationMinutes = values[position]
                }
            }

        val choose = button(
            "اختيار التطبيقات التعليمية المسموح بها"
        ) {
            showAppPicker()
        }

        val permission = button("تفعيل صلاحية الحظر") {
            startActivity(
                Intent(Settings.ACTION_ACCESSIBILITY_SETTINGS)
            )
        }

        val start = button("ابدأ جلسة التركيز") {
            startFocus()
        }

        val stop = button("إنهاء الجلسة") {
            prefs.edit().putLong("endAt", 0L).apply()
            refreshStatus()

            Toast.makeText(
                this,
                "انتهت جلسة التركيز",
                Toast.LENGTH_SHORT
            ).show()
        }

        val note = TextView(this).apply {
            text = """
                مهم: يجب تفعيل خدمة «تركيز - حظر التطبيقات»
                من إعدادات إمكانية الوصول.

                اختر التطبيقات التعليمية التي تريد إبقاءها متاحة.
                قد تختلف قوة الحظر حسب إصدار أندرويد
                والشركة المصنّعة.
            """.trimIndent()

            textSize = 13f
            setTextColor(Color.rgb(90, 100, 115))
            setPadding(4, 20, 4, 0)
        }

        listOf(
            title,
            subtitle,
            status,
            durationLabel,
            duration,
            choose,
            permission,
            start,
            stop,
            note
        ).forEach { view ->

            root.addView(
                view,
                LinearLayout.LayoutParams(
                    ViewGroup.LayoutParams.MATCH_PARENT,
                    if (view is Button) 56
                    else ViewGroup.LayoutParams.WRAP_CONTENT
                ).apply {
                    bottomMargin = if (view is Button) 10 else 4
                }
            )
        }

        setContentView(
            ScrollView(this).apply {
                addView(root)
            }
        )

        refreshStatus()
    }

    private fun button(
        label: String,
        action: () -> Unit
    ) = Button(this).apply {

        text = label
        textSize = 15f
        isAllCaps = false

        setOnClickListener {
            action()
        }
    }

    private fun showAppPicker() {

        val pm = packageManager

        val apps = pm.getInstalledApplications(0)
            .filter {
                pm.getLaunchIntentForPackage(it.packageName) != null &&
                    it.packageName != packageName
            }
            .sortedBy {
                pm.getApplicationLabel(it)
                    .toString()
                    .lowercase()
            }

        val names = apps.map {
            pm.getApplicationLabel(it).toString()
        }.toTypedArray()

        val checks = BooleanArray(apps.size) {
            selected.contains(apps[it].packageName)
        }

        AlertDialog.Builder(this)
            .setTitle("التطبيقات المسموح بها")
            .setMultiChoiceItems(names, checks) {
                    _, which, checked ->

                if (checked) {
                    selected.add(apps[which].packageName)
                } else {
                    selected.remove(apps[which].packageName)
                }
            }
            .setPositiveButton("حفظ") { _, _ ->

                selected.add(packageName)

                prefs.edit()
                    .putStringSet("allowed", selected.toSet())
                    .apply()

                Toast.makeText(
                    this,
                    "تم حفظ التطبيقات المسموح بها",
                    Toast.LENGTH_SHORT
                ).show()
            }
            .setNegativeButton("إلغاء", null)
            .show()
    }

    private fun startFocus() {

        val manager = getSystemService(
            ACCESSIBILITY_SERVICE
        ) as AccessibilityManager

        val enabled =
            manager.getEnabledAccessibilityServiceList(
                AccessibilityServiceInfo.FEEDBACK_ALL_MASK
            ).any {
                it.resolveInfo.serviceInfo.packageName == packageName
            }

        if (!enabled) {

            AlertDialog.Builder(this)
                .setTitle("يلزم تفعيل الصلاحية")
                .setMessage(
                    "افتح إعدادات إمكانية الوصول، ثم فعّل خدمة " +
                        "«تركيز - حظر التطبيقات» حتى يستطيع التطبيق " +
                        "منع فتح التطبيقات غير المسموح بها."
                )
                .setPositiveButton("فتح الإعدادات") { _, _ ->
                    startActivity(
                        Intent(Settings.ACTION_ACCESSIBILITY_SETTINGS)
                    )
                }
                .setNegativeButton("لاحقًا", null)
                .show()

            return
        }

        prefs.edit()
            .putLong(
                "endAt",
                System.currentTimeMillis() +
                    durationMinutes * 60_000L
            )
            .putStringSet("allowed", selected.toSet())
            .apply()

        refreshStatus()

        Toast.makeText(
            this,
            "بدأت جلسة التركيز",
            Toast.LENGTH_SHORT
        ).show()
    }

    private fun refreshStatus() {

        val end = prefs.getLong("endAt", 0L)
        val remaining = end - System.currentTimeMillis()

        status.text =
            if (remaining > 0) {
                "جلسة نشطة • المتبقي " +
                    "${remaining / 3_600_000} س و " +
                    "${(remaining / 60_000) % 60} د"
            } else {
                "لا توجد جلسة نشطة"
            }
    }
}package com.tarkeiz.focus

import android.app.Activity
import android.graphics.Color
import android.graphics.Typeface
import android.os.Bundle
import android.view.Gravity
import android.widget.LinearLayout
import android.widget.TextView

class BlockedActivity : Activity() {

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        val end = intent.getLongExtra("endAt", 0L)

        val mins = (
            (end - System.currentTimeMillis())
                .coerceAtLeast(0L) / 60_000L
            )

        val root = LinearLayout(this).apply {
            orientation = LinearLayout.VERTICAL
            gravity = Gravity.CENTER
            setPadding(32, 32, 32, 32)
            setBackgroundColor(Color.rgb(16, 42, 67))
            layoutDirection = LinearLayout.LayoutDirection.RTL
        }

        root.addView(
            TextView(this).apply {
                text = "وقت التركيز"
                textSize = 30f
                typeface = Typeface.DEFAULT_BOLD
                setTextColor(Color.WHITE)
                gravity = Gravity.CENTER
            }
        )

        root.addView(
            TextView(this).apply {
                text =
                    "هذا التطبيق غير متاح أثناء جلسة التركيز.\n\n" +
                    "ارجع إلى هدفك، فكل دقيقة تصنع فرقًا.\n\n" +
                    "الوقت المتبقي تقريبًا: $mins دقيقة"

                textSize = 19f
                setTextColor(Color.rgb(225, 235, 245))
                gravity = Gravity.CENTER
                setPadding(0, 24, 0, 24)
            }
        )

        val back = TextView(this).apply {
            text = "العودة"
            textSize = 18f
            setTextColor(Color.WHITE)
            gravity = Gravity.CENTER
            setPadding(24, 16, 24, 16)
            setBackgroundColor(Color.rgb(37, 99, 235))

            setOnClickListener {
                finish()
            }
        }

        root.addView(back)
        setContentView(root)
    }

    override fun onResume() {
        super.onResume()

        val end = getSharedPreferences(
            "tarkeiz",
            MODE_PRIVATE
        ).getLong("endAt", 0L)

        if (end <= System.currentTimeMillis()) {
            finish()
        }
    }
}name: Build Tarkeiz APK

on:
  workflow_dispatch:
  push:
    branches:
      - main

permissions:
  contents: read

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout source
        uses: actions/checkout@v4

      - name: Set up JDK 17
        uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: '17'

      - name: Set up Android SDK
        uses: android-actions/setup-android@v3

      - name: Install Android SDK
        run: |
          sdkmanager "platforms;android-35" "build-tools;35.0.0"

      - name: Install Gradle
        uses: gradle/actions/setup-gradle@v4
        with:
          gradle-version: '8.9'

      - name: Build debug APK
        working-directory: focusapp
        run: gradle --no-daemon assembleDebug

      - name: Upload APK
        uses: actions/upload-artifact@v4
        with:
          name: Tarkeiz-debug-apk
          path: focusapp/app/build/outputs/apk/debug/app-debug.apk
          if-no-files-found: errorval stop = button("إنهاء الجلسة") {
    prefs.edit().putLong("endAt", 0L).apply()
    refreshStatus()
    Toast.makeText(
        this,
        "انتهت جلسة التركيز",
        Toast.LENGTH_SHORT
    ).show()
}listOf(
    title,
    subtitle,
    status,
    durationLabel,
    duration,
    choose,
    permission,
    start,
    note
).forEach { view ->
    // استخدم كود إضافة العناصر الموجود في الملف السابق
}
