# -*- coding: utf-8 -*-
"""
شغّل هذا الملف مرة واحدة، فينشئ مشروع أندرويد كاملاً اسمه WalkieTalkie بجانبه.
"""
import os

FILES = {}
FILES['settings.gradle.kts'] = r'''pluginManagement { repositories { google(); mavenCentral(); gradlePluginPortal() } }
dependencyResolutionManagement { repositories { google(); mavenCentral() } }
rootProject.name = "WalkieTalkie"
include(":app")
'''
FILES['build.gradle.kts'] = r'''plugins {
    id("com.android.application") version "8.5.2" apply false
    id("org.jetbrains.kotlin.android") version "1.9.24" apply false
}
'''
FILES['gradle.properties'] = r'''android.useAndroidX=true
org.gradle.jvmargs=-Xmx2g
'''
FILES['gradle/wrapper/gradle-wrapper.properties'] = r'''distributionBase=GRADLE_USER_HOME
distributionPath=wrapper/dists
distributionUrl=https\://services.gradle.org/distributions/gradle-8.7-bin.zip
zipStoreBase=GRADLE_USER_HOME
zipStorePath=wrapper/dists
'''
FILES['app/build.gradle.kts'] = r'''plugins { id("com.android.application"); id("org.jetbrains.kotlin.android") }
android {
    namespace = "com.example.walkie"
    compileSdk = 34
    defaultConfig { applicationId = "com.example.walkie"; minSdk = 26; targetSdk = 34; versionCode = 1; versionName = "1.0" }
    compileOptions { sourceCompatibility = JavaVersion.VERSION_17; targetCompatibility = JavaVersion.VERSION_17 }
    kotlinOptions { jvmTarget = "17" }
}
dependencies { implementation("com.google.android.gms:play-services-nearby:19.3.0") }
'''
FILES['app/src/main/AndroidManifest.xml'] = r'''<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android">
    <uses-permission android:name="android.permission.RECORD_AUDIO"/>
    <uses-permission android:name="android.permission.ACCESS_WIFI_STATE"/>
    <uses-permission android:name="android.permission.CHANGE_WIFI_STATE"/>
    <uses-permission android:name="android.permission.BLUETOOTH" android:maxSdkVersion="30"/>
    <uses-permission android:name="android.permission.BLUETOOTH_ADMIN" android:maxSdkVersion="30"/>
    <uses-permission android:name="android.permission.ACCESS_COARSE_LOCATION" android:maxSdkVersion="32"/>
    <uses-permission android:name="android.permission.ACCESS_FINE_LOCATION" android:maxSdkVersion="32"/>
    <uses-permission android:name="android.permission.BLUETOOTH_ADVERTISE"/>
    <uses-permission android:name="android.permission.BLUETOOTH_CONNECT"/>
    <uses-permission android:name="android.permission.BLUETOOTH_SCAN"/>
    <uses-permission android:name="android.permission.NEARBY_WIFI_DEVICES" android:usesPermissionFlags="neverForLocation"/>
    <application android:label="لاسلكي" android:theme="@android:style/Theme.Material.Light.NoActionBar">
        <activity android:name=".MainActivity" android:exported="true">
            <intent-filter>
                <action android:name="android.intent.action.MAIN"/>
                <category android:name="android.intent.category.LAUNCHER"/>
            </intent-filter>
        </activity>
    </application>
</manifest>
'''
FILES['app/src/main/java/com/example/walkie/MainActivity.kt'] = r'''package com.example.walkie

import android.Manifest
import android.app.Activity
import android.content.pm.PackageManager
import android.graphics.Color
import android.media.*
import android.os.Build
import android.os.Bundle
import android.os.ParcelFileDescriptor
import android.view.Gravity
import android.view.MotionEvent
import android.view.View
import android.widget.*
import com.google.android.gms.nearby.Nearby
import com.google.android.gms.nearby.connection.*
import java.io.IOException
import java.util.concurrent.ConcurrentHashMap
import kotlin.concurrent.thread

class MainActivity : Activity() {
    private val serviceId = "com.example.walkie.channel"
    private val rate = 16000
    private val myName = "u" + (100000..999999).random()
    private val peers = ConcurrentHashMap.newKeySet<String>()
    private lateinit var client: ConnectionsClient
    private lateinit var status: TextView
    private lateinit var talkBtn: Button
    @Volatile private var talking = false

    override fun onCreate(b: Bundle?) {
        super.onCreate(b)
        client = Nearby.getConnectionsClient(this)
        setContentView(R.layout.activity_main)
        status = findViewById(R.id.status)
        talkBtn = findViewById(R.id.talkBtn)
        talkBtn.setOnTouchListener { v, e ->
            when (e.action) {
                MotionEvent.ACTION_DOWN -> { startTalking(); v.setBackgroundColor(Color.parseColor("#C62828")); true }
                MotionEvent.ACTION_UP, MotionEvent.ACTION_CANCEL -> { talking = false; v.setBackgroundColor(Color.parseColor("#2E7D32")); v.performClick(); true }
                else -> false
            }
        }
        if (missingPermissions().isEmpty()) startNetwork() else requestPermissions(missingPermissions().toTypedArray(), 1)
    }

    private fun neededPermissions(): List<String> {
        val l = mutableListOf(Manifest.permission.RECORD_AUDIO)
        if (Build.VERSION.SDK_INT >= 31) l += listOf(Manifest.permission.BLUETOOTH_ADVERTISE, Manifest.permission.BLUETOOTH_CONNECT, Manifest.permission.BLUETOOTH_SCAN)
        else l += Manifest.permission.ACCESS_FINE_LOCATION
        if (Build.VERSION.SDK_INT >= 33) l += Manifest.permission.NEARBY_WIFI_DEVICES
        return l
    }
    private fun missingPermissions() = neededPermissions().filter { checkSelfPermission(it) != PackageManager.PERMISSION_GRANTED }

    override fun onRequestPermissionsResult(c: Int, p: Array<out String>, r: IntArray) {
        if (missingPermissions().isEmpty()) startNetwork() else status.text = "الصلاحيات مطلوبة لعمل التطبيق"
    }

    private fun updateStatus() = runOnUiThread {
        status.text = if (peers.isEmpty()) "جارٍ البحث عن أجهزة قريبة..." else "متصل بـ ${peers.size} جهاز"
    }

    // ===== الاتصال: واي فاي أو بلوتوث تلقائياً =====
    private val lifecycle = object : ConnectionLifecycleCallback() {
        override fun onConnectionInitiated(id: String, info: ConnectionInfo) { client.acceptConnection(id, payloadCb) }
        override fun onConnectionResult(id: String, r: ConnectionResolution) { if (r.status.isSuccess) { peers.add(id); updateStatus() } }
        override fun onDisconnected(id: String) { peers.remove(id); updateStatus() }
    }
    private val discoveryCb = object : EndpointDiscoveryCallback() {
        override fun onEndpointFound(id: String, info: DiscoveredEndpointInfo) {
            // طرف واحد فقط يبادر بالاتصال لتفادي التعارض
            if (myName < info.endpointName) client.requestConnection(myName, id, lifecycle)
        }
        override fun onEndpointLost(id: String) {}
    }

    private fun startNetwork() {
        val s = Strategy.P2P_CLUSTER
        client.startAdvertising(myName, serviceId, lifecycle, AdvertisingOptions.Builder().setStrategy(s).build())
        client.startDiscovery(serviceId, discoveryCb, DiscoveryOptions.Builder().setStrategy(s).build())
    }

    // ===== استقبال الصوت =====
    private val payloadCb = object : PayloadCallback() {
        override fun onPayloadReceived(id: String, p: Payload) {
            val input = p.asStream()?.asInputStream() ?: return
            thread {
                val min = AudioTrack.getMinBufferSize(rate, AudioFormat.CHANNEL_OUT_MONO, AudioFormat.ENCODING_PCM_16BIT)
                val track = AudioTrack.Builder()
                    .setAudioAttributes(AudioAttributes.Builder().setUsage(AudioAttributes.USAGE_MEDIA).setContentType(AudioAttributes.CONTENT_TYPE_SPEECH).build())
                    .setAudioFormat(AudioFormat.Builder().setEncoding(AudioFormat.ENCODING_PCM_16BIT).setSampleRate(rate).setChannelMask(AudioFormat.CHANNEL_OUT_MONO).build())
                    .setBufferSizeInBytes(min * 2).setTransferMode(AudioTrack.MODE_STREAM).build()
                track.play()
                val buf = ByteArray(min)
                try { while (true) { val n = input.read(buf); if (n < 0) break; track.write(buf, 0, n) } } catch (_: IOException) {}
                finally { track.stop(); track.release(); input.close() }
            }
        }
        override fun onPayloadTransferUpdate(id: String, u: PayloadTransferUpdate) {}
    }

    // ===== إرسال الصوت =====
    private fun startTalking() {
        if (talking || peers.isEmpty()) return
        talking = true
        val pipe = ParcelFileDescriptor.createPipe()
        client.sendPayload(peers.toList(), Payload.fromStream(pipe[0]))
        thread {
            val min = AudioRecord.getMinBufferSize(rate, AudioFormat.CHANNEL_IN_MONO, AudioFormat.ENCODING_PCM_16BIT)
            val rec = AudioRecord(MediaRecorder.AudioSource.VOICE_COMMUNICATION, rate, AudioFormat.CHANNEL_IN_MONO, AudioFormat.ENCODING_PCM_16BIT, min * 2)
            val out = ParcelFileDescriptor.AutoCloseOutputStream(pipe[1])
            val buf = ByteArray(min)
            rec.startRecording()
            try { while (talking) { val n = rec.read(buf, 0, buf.size); if (n > 0) out.write(buf, 0, n) } } catch (_: IOException) {}
            finally { rec.stop(); rec.release(); out.close() }
        }
    }

    override fun onDestroy() { talking = false; client.stopAllEndpoints(); client.stopAdvertising(); client.stopDiscovery(); super.onDestroy() }
}
'''
FILES['app/src/main/res/layout/activity_main.xml'] = r'''<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical"
    android:layoutDirection="rtl"
    android:padding="24dp"
    android:background="#FFFFFF">

    <TextView
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:gravity="center"
        android:text="لاسلكي"
        android:textColor="#1B5E20"
        android:textSize="28sp"
        android:textStyle="bold" />

    <TextView
        android:id="@+id/status"
        android:layout_width="match_parent"
        android:layout_height="0dp"
        android:layout_weight="1"
        android:gravity="center"
        android:text="جارٍ البحث عن أجهزة قريبة..."
        android:textColor="#424242"
        android:textSize="18sp" />

    <Button
        android:id="@+id/talkBtn"
        android:layout_width="match_parent"
        android:layout_height="0dp"
        android:layout_weight="2"
        android:background="#2E7D32"
        android:text="اضغط للتحدث"
        android:textColor="#FFFFFF"
        android:textSize="26sp" />
</LinearLayout>
'''

base = os.path.join(os.path.dirname(os.path.abspath(__file__)), "WalkieTalkie")
for path, content in FILES.items():
    full = os.path.join(base, *path.split("/"))
    os.makedirs(os.path.dirname(full), exist_ok=True)
    with open(full, "w", encoding="utf-8", newline="\n") as fh:
        fh.write(content)
print("تم إنشاء المشروع في:", base)
print("افتحه الآن من Android Studio: File > Open")
