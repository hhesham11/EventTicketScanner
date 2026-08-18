package com.eventtickets.scanner

import android.Manifest
import android.content.Context
import android.content.pm.PackageManager
import android.media.AudioManager
import android.media.ToneGenerator
import android.os.Bundle
import android.os.Vibrator
import android.os.VibrationEffect
import android.view.ViewGroup
import android.widget.*
import androidx.activity.ComponentActivity
import androidx.camera.core.*
import androidx.camera.lifecycle.ProcessCameraProvider
import androidx.camera.view.PreviewView
import androidx.core.app.ActivityCompat
import androidx.core.content.ContextCompat
import com.google.mlkit.vision.barcode.BarcodeScanning
import com.google.mlkit.vision.common.InputImage
import okhttp3.*
import okhttp3.MediaType.Companion.toMediaType
import okhttp3.RequestBody.Companion.toRequestBody
import org.json.JSONObject
import java.io.IOException
import java.util.concurrent.Executors

class MainActivity : ComponentActivity() {
    private lateinit var preview: PreviewView
    private lateinit var status: TextView
    private lateinit var server: EditText
    private lateinit var eventId: EditText
    private lateinit var gateId: EditText
    private lateinit var key: EditText
    private lateinit var manualCode: EditText
    private val client = OkHttpClient()
    private val executor = Executors.newSingleThreadExecutor()
    private val prefs by lazy { getSharedPreferences("scanner", Context.MODE_PRIVATE) }
    @Volatile private var busy = false
    private var lastCode = ""
    private var lastScanAt = 0L
    private var tone: ToneGenerator? = null
    private val CAMERA_PERMISSION_CODE = 5

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        try {
            tone = ToneGenerator(AudioManager.STREAM_NOTIFICATION, 100)
        } catch (e: Exception) {
            tone = null
        }
        buildUi()
        if (ContextCompat.checkSelfPermission(this, Manifest.permission.CAMERA) != PackageManager.PERMISSION_GRANTED) {
            ActivityCompat.requestPermissions(this, arrayOf(Manifest.permission.CAMERA), CAMERA_PERMISSION_CODE)
        } else startCamera()
    }

    override fun onRequestPermissionsResult(requestCode: Int, permissions: Array<out String>, grantResults: IntArray) {
        super.onRequestPermissionsResult(requestCode, permissions, grantResults)
        if (requestCode == CAMERA_PERMISSION_CODE) {
            if (grantResults.isNotEmpty() && grantResults[0] == PackageManager.PERMISSION_GRANTED) {
                startCamera()
            } else {
                status.text = "Camera permission denied. Enable it in Settings to scan."
            }
        }
    }

    private fun buildUi() {
        val root = LinearLayout(this).apply { orientation = LinearLayout.VERTICAL; setPadding(16,16,16,16) }
        val title = TextView(this).apply { text = "🎟 Event Ticket Scanner"; textSize = 24f; setPadding(0,0,0,12) }
        server = EditText(this).apply { hint = "Server URL  http://192.168.1.30:5000"; setSingleLine() }
        eventId = EditText(this).apply { hint = "Event ID"; inputType = 2; setSingleLine(); setText(prefs.getString("eventId", "")) }
        gateId = EditText(this).apply { hint = "Gate ID"; inputType = 2; setSingleLine(); setText(prefs.getString("gateId", "")) }
        key = EditText(this).apply { hint = "Scanner key"; setSingleLine(); setText(prefs.getString("key", "")) }
        server.setText(prefs.getString("server", ""))
        val save = Button(this).apply { text = "Save Settings"; setOnClickListener { saveSettings(); toast("Settings saved") } }
        val test = Button(this).apply { text = "Test Server"; setOnClickListener { testServer() } }
        status = TextView(this).apply { text = "Ready — point the camera at a QR code"; textSize = 20f; setPadding(0,12,0,12) }
        manualCode = EditText(this).apply { hint = "Manual ticket number / QR value"; setSingleLine() }
        val manual = Button(this).apply { text = "Check Ticket"; setOnClickListener { scan(manualCode.text.toString()) } }
        preview = PreviewView(this).apply { layoutParams = LinearLayout.LayoutParams(ViewGroup.LayoutParams.MATCH_PARENT, 0, 1f) }
        root.addView(title); root.addView(server); root.addView(eventId); root.addView(gateId); root.addView(key); root.addView(save); root.addView(test); root.addView(status); root.addView(manualCode); root.addView(manual); root.addView(preview)
        setContentView(root)
    }

    private fun saveSettings() {
        prefs.edit().putString("server", server.text.toString().trim().trimEnd('/')).putString("eventId", eventId.text.toString()).putString("gateId", gateId.text.toString()).putString("key", key.text.toString()).apply()
    }

    private fun testServer() {
        val url = server.text.toString().trim().trimEnd('/') + "/api/scan/check"
        if (url.startsWith("/")) return
        status.text = "Testing server..."
        val body = JSONObject().put("code", "TEST").put("eventId", eventId.text.toString().toIntOrNull() ?: 0).put("gateId", gateId.text.toString().toIntOrNull() ?: 0).put("scannerKey", key.text.toString()).toString()
        val req = Request.Builder().url(url).post(body.toRequestBody("application/json".toMediaType())).build()
        client.newCall(req).enqueue(object: Callback {
            override fun onFailure(call: Call, e: IOException) = runOnUiThread { status.text = "SERVER ERROR\n${e.message}" }
            override fun onResponse(call: Call, response: Response) { runOnUiThread { status.text = if (response.code == 401) "Server reachable — scanner key rejected" else "Server reachable — HTTP ${response.code}" } }
        })
    }

    private fun startCamera() {
        val future = ProcessCameraProvider.getInstance(this)
        future.addListener({
            val provider = future.get()
            val previewUse = Preview.Builder().build().also { it.setSurfaceProvider(preview.surfaceProvider) }
            val analysis = ImageAnalysis.Builder().setBackpressureStrategy(ImageAnalysis.STRATEGY_KEEP_ONLY_LATEST).build()
            analysis.setAnalyzer(executor) { proxy ->
                val media = proxy.image
                if (media == null) { proxy.close(); return@setAnalyzer }
                val image = InputImage.fromMediaImage(media, proxy.imageInfo.rotationDegrees)
                BarcodeScanning.getClient().process(image).addOnSuccessListener { codes ->
                    val value = codes.firstOrNull()?.rawValue ?: return@addOnSuccessListener
                    val now = System.currentTimeMillis()
                    if (!busy && (value != lastCode || now - lastScanAt > 3000)) { lastCode = value; lastScanAt = now; busy = true; scan(value) }
                }.addOnCompleteListener { proxy.close() }
            }
            provider.unbindAll()
            provider.bindToLifecycle(this, CameraSelector.DEFAULT_BACK_CAMERA, previewUse, analysis)
        }, ContextCompat.getMainExecutor(this))
    }

    private fun scan(code: String) {
        val clean = code.trim(); if (clean.isEmpty()) return
        saveSettings(); runOnUiThread { status.text = "Checking..." }
        val url = server.text.toString().trim().trimEnd('/') + "/api/scan/check"
        val body = JSONObject().put("code", clean).put("eventId", eventId.text.toString().toIntOrNull() ?: 0).put("gateId", gateId.text.toString().toIntOrNull() ?: 0).put("scannerKey", key.text.toString()).toString()
        val req = Request.Builder().url(url).post(body.toRequestBody("application/json".toMediaType())).build()
        client.newCall(req).enqueue(object: Callback {
            override fun onFailure(call: Call, e: IOException) { runOnUiThread { status.text = "CONNECTION ERROR\n${e.message}"; tone?.startTone(ToneGenerator.TONE_PROP_NACK, 250) }; busy = false }
            override fun onResponse(call: Call, response: Response) {
                val json = JSONObject(response.body?.string() ?: "{}")
                val st = json.optString("status", "INVALID")
                val text = "${json.optString("message", "Unknown")}\n${json.optString("guest", "")}\n${json.optString("ticket", "")}\nTable ${json.optString("table", "-")} · Seat ${json.optString("seat", "-")}"
                runOnUiThread {
                    status.text = text
                    if (st == "VALID") { tone?.startTone(ToneGenerator.TONE_PROP_ACK, 200); vibrate() }
                    else tone?.startTone(ToneGenerator.TONE_PROP_NACK, 300)
                }
                Thread.sleep(1600); busy = false
            }
        })
    }

    private fun vibrate() { val v = getSystemService(Vibrator::class.java); v?.vibrate(VibrationEffect.createOneShot(120, VibrationEffect.DEFAULT_AMPLITUDE)) }
    private fun toast(s: String) = Toast.makeText(this, s, Toast.LENGTH_SHORT).show()
    override fun onDestroy() { super.onDestroy(); executor.shutdown(); tone?.release() }
}
