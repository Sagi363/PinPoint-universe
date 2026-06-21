# PDR Foreground Service: Implementation Guide

**Status**: Ready for Development  
**Platform**: Android 12+ (API 31+)  
**Architecture**: Moderate (Sensor Streaming + ZUPT + Client Kalman)  
**Estimated Timeline**: 12 hours

---

## 1. Project Structure

```
android/pdr/
├── src/main/
│   ├── kotlin/com/plangrid/android/pdr/
│   │   ├── service/
│   │   │   ├── PDRForegroundService.kt
│   │   │   ├── SensorAcquisitionThread.kt
│   │   │   ├── PDRProcessor.kt
│   │   │   └── PDRBinder.kt (or AIDL)
│   │   ├── sensor/
│   │   │   ├── SensorSnapshot.kt
│   │   │   ├── ProcessedSensor.kt
│   │   │   ├── SensorPipeline.kt
│   │   │   └── CalibrationManager.kt
│   │   ├── pdr/
│   │   │   ├── PDRUpdate.kt
│   │   │   ├── StepDetector.kt
│   │   │   ├── ZeroVelocityDetector.kt
│   │   │   └── HeadingIntegrator.kt
│   │   └── client/
│   │       └── PDRClient.kt (example consumer)
│   ├── aidl/com/plangrid/android/pdr/
│   │   └── IPDRService.aidl (if using AIDL)
│   └── AndroidManifest.xml
├── src/test/
│   └── kotlin/com/plangrid/android/pdr/
│       ├── SensorPipelineTest.kt
│       ├── ZeroVelocityDetectorTest.kt
│       └── PDRProcessorTest.kt
└── build.gradle.kts
```

---

## 2. Core Data Classes

### 2.1 SensorSnapshot.kt

```kotlin
package com.plangrid.android.pdr.sensor

import kotlin.math.sqrt

data class SensorSnapshot(
    val timestampNs: Long,                  // System.nanoTime()
    val accelX: Float,
    val accelY: Float,
    val accelZ: Float,
    val gyroX: Float,
    val gyroY: Float,
    val gyroZ: Float,
    val magX: Float?,
    val magY: Float?,
    val magZ: Float?,
    val pressure: Float?,                   // hPa; optional
    val temperature: Float?,                // °C; optional
) {
    val accelMagnitude: Float
        get() = sqrt(accelX * accelX + accelY * accelY + accelZ * accelZ)
    
    val gyroMagnitude: Float
        get() = sqrt(gyroX * gyroX + gyroY * gyroY + gyroZ * gyroZ)
    
    val magMagnitude: Float?
        get() = if (magX != null && magY != null && magZ != null) {
            sqrt(magX * magX + magY * magY + magZ * magZ)
        } else null
    
    companion object {
        fun from(event: android.hardware.SensorEvent): SensorSnapshot {
            return when (event.sensor.type) {
                android.hardware.Sensor.TYPE_ACCELEROMETER -> {
                    SensorSnapshot(
                        timestampNs = event.timestamp,
                        accelX = event.values[0],
                        accelY = event.values[1],
                        accelZ = event.values[2],
                        gyroX = 0f, gyroY = 0f, gyroZ = 0f,
                        magX = null, magY = null, magZ = null,
                        pressure = null, temperature = null
                    )
                }
                // ... other sensors
                else -> error("Unsupported sensor type: ${event.sensor.type}")
            }
        }
    }
}
```

### 2.2 ProcessedSensor.kt

```kotlin
package com.plangrid.android.pdr.sensor

data class ProcessedSensor(
    val timestamp: Long,
    val dt: Float,                           // Seconds since last sample
    val gravityVector: FloatArray,          // [gx, gy, gz]; m/s²
    val linearAccel: FloatArray,            // Accel - gravity; m/s²
    val rotationRate: FloatArray,           // Gyro after bias removal; rad/s
    val deviceHeading: Float,                // Degrees; relative to gravity plane
    val magHeadingIfReliable: Float?,       // Degrees; nullable if unreliable
    val accelVariance: Float,               // For zero-velocity detection
    val gyroVariance: Float,
) {
    override fun equals(other: Any?): Boolean {
        if (this === other) return true
        if (other !is ProcessedSensor) return false
        
        if (timestamp != other.timestamp) return false
        if (dt != other.dt) return false
        if (!gravityVector.contentEquals(other.gravityVector)) return false
        if (!linearAccel.contentEquals(other.linearAccel)) return false
        if (!rotationRate.contentEquals(other.rotationRate)) return false
        if (deviceHeading != other.deviceHeading) return false
        if (magHeadingIfReliable != other.magHeadingIfReliable) return false
        
        return true
    }
    
    override fun hashCode(): Int {
        var result = timestamp.hashCode()
        result = 31 * result + dt.hashCode()
        result = 31 * result + gravityVector.contentHashCode()
        result = 31 * result + linearAccel.contentHashCode()
        result = 31 * result + rotationRate.contentHashCode()
        result = 31 * result + deviceHeading.hashCode()
        result = 31 * result + (magHeadingIfReliable?.hashCode() ?: 0)
        return result
    }
}
```

### 2.3 PDRUpdate.kt

```kotlin
package com.plangrid.android.pdr

data class PDRUpdate(
    val timestamp: Long,                    // System time when computed
    val x: Float,                           // Meters from origin
    val y: Float,
    val heading: Float,                     // Degrees (0-360)
    val confidence: Float,                  // 0.0-1.0; indicates filter uncertainty
    val stepCount: Int,                     // Cumulative steps detected
    val velocity: Float,                    // m/s; estimated speed
    val zeroVelocityDetected: Boolean,     // Whether ZUPT is active
    val magReliable: Boolean,               // Whether magnetometer can be trusted
    val estimatedAccuracy: Float,           // Meters; 1-sigma horizontal error estimate
) {
    init {
        require(confidence in 0f..1f) { "Confidence must be in [0, 1]" }
        require(heading in 0f..360f) { "Heading must be in [0, 360]" }
    }
}
```

---

## 3. Sensor Acquisition & Processing

### 3.1 SensorAcquisitionThread.kt

```kotlin
package com.plangrid.android.pdr.service

import android.hardware.Sensor
import android.hardware.SensorEvent
import android.hardware.SensorEventListener
import android.hardware.SensorManager
import android.os.HandlerThread
import com.plangrid.android.pdr.sensor.SensorSnapshot
import java.util.concurrent.LinkedBlockingQueue
import java.util.concurrent.atomic.AtomicBoolean

class SensorAcquisitionThread(
    private val sensorManager: SensorManager,
) : HandlerThread("PDRSensorAcquisition"), SensorEventListener {
    
    private val isStarted = AtomicBoolean(false)
    
    // Ring buffer to avoid allocation in sensor callback
    private val sensorBuffer = LinkedBlockingQueue<SensorSnapshot>(capacity = 400)  // 2 sec @ 200 Hz
    
    private var accelerometer: Sensor? = null
    private var gyroscope: Sensor? = null
    private var magnetometer: Sensor? = null
    private var barometer: Sensor? = null
    
    fun startSampling() {
        if (isStarted.getAndSet(true)) return
        
        val handler = android.os.Handler(looper)
        
        accelerometer = sensorManager.getDefaultSensor(Sensor.TYPE_ACCELEROMETER)
        gyroscope = sensorManager.getDefaultSensor(Sensor.TYPE_GYROSCOPE)
        magnetometer = sensorManager.getDefaultSensor(Sensor.TYPE_MAGNETIC_FIELD)
        barometer = sensorManager.getDefaultSensor(Sensor.TYPE_PRESSURE)
        
        accelerometer?.let {
            sensorManager.registerListener(this, it, SensorManager.SENSOR_DELAY_GAME, handler)
        }
        gyroscope?.let {
            sensorManager.registerListener(this, it, SensorManager.SENSOR_DELAY_GAME, handler)
        }
        magnetometer?.let {
            sensorManager.registerListener(this, it, SensorManager.SENSOR_DELAY_UI, handler)
        }
        barometer?.let {
            sensorManager.registerListener(this, it, SensorManager.SENSOR_DELAY_NORMAL, handler)
        }
    }
    
    override fun onSensorChanged(event: SensorEvent?) {
        event ?: return
        
        // Note: In production, accumulate multiple sensor types into composite snapshot
        // For now, queue individual events; ProcessingPipeline groups them
        val snapshot = SensorSnapshot.from(event)
        sensorBuffer.offer(snapshot)  // Non-blocking
    }
    
    override fun onAccuracyChanged(sensor: Sensor?, accuracy: Int) {
        // Log accuracy changes for debugging
    }
    
    fun pollBuffer(): List<SensorSnapshot> {
        val batch = mutableListOf<SensorSnapshot>()
        sensorBuffer.drainTo(batch)
        return batch
    }
    
    fun stopSampling() {
        if (!isStarted.getAndSet(false)) return
        sensorManager.unregisterListener(this)
        sensorBuffer.clear()
    }
    
    fun shutdown() {
        stopSampling()
        quitSafely()
    }
}
```

### 3.2 SensorPipeline.kt (Calibration + Processing)

```kotlin
package com.plangrid.android.pdr.sensor

import kotlin.math.sqrt

class SensorPipeline(
    private val calibration: CalibrationManager = CalibrationManager(),
) {
    
    private var lastTimestampNs = 0L
    private val gravityLowPass = FloatArray(3) { 0f }
    private val gravityAlpha = 0.05f  // τ = 0.5 sec at 200 Hz
    
    // Latest measurements (updated per onSensorChanged call)
    private var latestAccel = FloatArray(3)
    private var latestGyro = FloatArray(3)
    private var latestMag = FloatArray(3)
    
    // Variance tracking (for ZUPT)
    private val accelVarianceWindow = FloatArray(20)  // 100 ms window
    private val gyroVarianceWindow = FloatArray(20)
    private var varWindowIdx = 0
    
    fun process(snapshots: List<SensorSnapshot>): ProcessedSensor? {
        if (snapshots.isEmpty()) return null
        
        // Merge latest readings
        for (snap in snapshots) {
            when {
                snap.accelX != 0f || snap.accelY != 0f || snap.accelZ != 0f ->
                    latestAccel = floatArrayOf(snap.accelX, snap.accelY, snap.accelZ)
                
                snap.gyroX != 0f || snap.gyroY != 0f || snap.gyroZ != 0f ->
                    latestGyro = floatArrayOf(snap.gyroX, snap.gyroY, snap.gyroZ)
                
                snap.magX != null ->
                    latestMag = floatArrayOf(snap.magX, snap.magY!!, snap.magZ!!)
            }
        }
        
        val lastTimestamp = lastTimestampNs
        val currentTimestamp = snapshots.last().timestampNs
        lastTimestampNs = currentTimestamp
        
        val dt = if (lastTimestamp == 0L) 0.005f else (currentTimestamp - lastTimestamp) / 1e9f
        
        // 1. Gravity extraction via low-pass filter
        updateGravityEstimate(latestAccel)
        
        // 2. Remove gravity from accel
        val linearAccel = FloatArray(3)
        for (i in 0..2) {
            linearAccel[i] = latestAccel[i] - gravityLowPass[i]
        }
        
        // 3. Remove bias from gyro
        val gyroCalibrated = calibration.removeGyroBias(latestGyro)
        
        // 4. Compute heading (from gravity + gyro integration)
        val heading = computeHeading(gravityLowPass, gyroCalibrated)
        
        // 5. Check magnetometer reliability
        val magReliable = isMagnetometerReliable(latestMag, heading)
        val magHeading = if (magReliable) computeMagHeading(latestMag) else null
        
        // 6. Track variance for ZUPT
        val accelVar = updateVarianceTracking(latestAccel, accelVarianceWindow)
        val gyroVar = updateVarianceTracking(gyroCalibrated, gyroVarianceWindow)
        
        return ProcessedSensor(
            timestamp = currentTimestamp,
            dt = dt,
            gravityVector = gravityLowPass.copyOf(),
            linearAccel = linearAccel,
            rotationRate = gyroCalibrated,
            deviceHeading = heading,
            magHeadingIfReliable = magHeading,
            accelVariance = accelVar,
            gyroVariance = gyroVar,
        )
    }
    
    private fun updateGravityEstimate(accel: FloatArray) {
        for (i in 0..2) {
            gravityLowPass[i] = gravityAlpha * accel[i] + (1 - gravityAlpha) * gravityLowPass[i]
        }
    }
    
    private fun computeHeading(gravity: FloatArray, gyro: FloatArray): Float {
        // Simplified: use gravity to infer up-axis, integrate gyro for heading
        val gravityMag = sqrt(gravity[0] * gravity[0] + gravity[1] * gravity[1] + gravity[2] * gravity[2])
        val normalized = FloatArray(3) { i -> gravity[i] / gravityMag }
        
        // Heading from accel alone is ambiguous; primarily use gyro Z
        // In production: compute full rotation matrix and extract yaw angle
        return (headingAccumulator + gyro[2] * 57.3f) % 360f  // rad/s -> deg/s
    }
    
    private fun isMagnetometerReliable(mag: FloatArray, expectedHeading: Float): Boolean {
        val magMag = sqrt(mag[0] * mag[0] + mag[1] * mag[1] + mag[2] * mag[2])
        
        // Expect Earth's magnetic field: 25-65 µT
        return magMag in 20f..70f
    }
    
    private fun computeMagHeading(mag: FloatArray): Float {
        // atan2 between East and North components
        val heading = kotlin.math.atan2(mag[1], mag[0]) * 57.3f
        return (heading + 360f) % 360f
    }
    
    private fun updateVarianceTracking(values: FloatArray, window: FloatArray): Float {
        val magnitude = sqrt(values[0] * values[0] + values[1] * values[1] + values[2] * values[2])
        window[varWindowIdx] = magnitude
        varWindowIdx = (varWindowIdx + 1) % window.size
        
        val mean = window.average().toFloat()
        val variance = window.map { (it - mean) * (it - mean) }.average().toFloat()
        return variance
    }
    
    private var headingAccumulator = 0f
}
```

### 3.3 CalibrationManager.kt

```kotlin
package com.plangrid.android.pdr.sensor

import android.content.Context
import android.content.SharedPreferences

class CalibrationManager(private val context: Context? = null) {
    
    private var gyroBias = FloatArray(3)
    private var accelBias = FloatArray(3)
    private var magOffset = FloatArray(3)
    private var magScale = floatArrayOf(1f, 1f, 1f)
    
    init {
        context?.let { loadCalibration() }
    }
    
    fun removeGyroBias(rawGyro: FloatArray): FloatArray {
        return FloatArray(3) { i -> rawGyro[i] - gyroBias[i] }
    }
    
    fun removeAccelBias(rawAccel: FloatArray): FloatArray {
        return FloatArray(3) { i -> rawAccel[i] - accelBias[i] }
    }
    
    fun calibrateMagnetometer(rawMag: FloatArray): FloatArray {
        return FloatArray(3) { i ->
            magScale[i] * (rawMag[i] - magOffset[i])
        }
    }
    
    fun updateGyroBiasFromZeroVelocity(gyroReadings: List<FloatArray>) {
        // Average gyro readings during detected zero-velocity window
        val meanBias = FloatArray(3)
        for (reading in gyroReadings) {
            for (i in 0..2) meanBias[i] += reading[i]
        }
        for (i in 0..2) meanBias[i] /= gyroReadings.size
        
        // Low-pass update (don't trust single window)
        for (i in 0..2) {
            gyroBias[i] = 0.1f * meanBias[i] + 0.9f * gyroBias[i]
        }
        
        context?.let { saveCalibration() }
    }
    
    private fun loadCalibration() {
        val prefs = context!!.getSharedPreferences("pdr_calibration", Context.MODE_PRIVATE)
        gyroBias = FloatArray(3) { i ->
            prefs.getFloat("gyro_bias_$i", 0f)
        }
        // ... load other calibrations
    }
    
    private fun saveCalibration() {
        val prefs = context!!.getSharedPreferences("pdr_calibration", Context.MODE_PRIVATE)
        prefs.edit().apply {
            gyroBias.forEachIndexed { i, bias ->
                putFloat("gyro_bias_$i", bias)
            }
            apply()
        }
    }
}
```

---

## 4. PDR Computation

### 4.1 ZeroVelocityDetector.kt

```kotlin
package com.plangrid.android.pdr.pdr

import com.plangrid.android.pdr.sensor.ProcessedSensor

class ZeroVelocityDetector(
    private val accelThreshold: Float = 0.1f,
    private val gyroThreshold: Float = 0.01f,
    private val windowSize: Int = 100,  // 0.5 sec @ 200 Hz
) {
    
    private val accelBuffer = FloatArray(windowSize)
    private val gyroBuffer = FloatArray(windowSize)
    private var bufferIdx = 0
    private var windowsFull = 0
    
    fun onProcessedSensor(processed: ProcessedSensor): Boolean {
        // Add to buffers
        accelBuffer[bufferIdx] = processed.accelVariance
        gyroBuffer[bufferIdx] = processed.gyroVariance
        bufferIdx = (bufferIdx + 1) % windowSize
        
        if (bufferIdx == 0) windowsFull++
        
        // Need full window before detection
        if (windowsFull < 1) return false
        
        // Check if all recent samples are below threshold
        val allAccelLow = accelBuffer.all { it < accelThreshold }
        val allGyroLow = gyroBuffer.all { it < gyroThreshold }
        
        return allAccelLow && allGyroLow
    }
    
    fun reset() {
        bufferIdx = 0
        windowsFull = 0
    }
}
```

### 4.2 StepDetector.kt

```kotlin
package com.plangrid.android.pdr.pdr

import com.plangrid.android.pdr.sensor.ProcessedSensor
import kotlin.math.sqrt

class StepDetector(
    private val accelThreshold: Float = 1.5f,  // m/s²
    private val minStepInterval: Float = 0.4f,  // seconds
) {
    
    private var lastStepTimeNs = 0L
    private var lastStepLength = 0.7f
    
    fun onProcessedSensor(processed: ProcessedSensor): StepDetected? {
        val timeSinceLastStep = (processed.timestamp - lastStepTimeNs) / 1e9f
        
        // Check acceleration magnitude
        val accelMag = sqrt(
            processed.linearAccel[0] * processed.linearAccel[0] +
            processed.linearAccel[1] * processed.linearAccel[1] +
            processed.linearAccel[2] * processed.linearAccel[2]
        )
        
        // Simple peak detection: accel spike in Z (vertical)
        if (accelMag > accelThreshold && timeSinceLastStep > minStepInterval) {
            lastStepTimeNs = processed.timestamp
            lastStepLength = estimateStepLength()
            
            return StepDetected(
                timestamp = processed.timestamp,
                length = lastStepLength,
                heading = processed.deviceHeading,
            )
        }
        
        return null
    }
    
    private fun estimateStepLength(): Float {
        // Anthropometric model: L = 0.4 + 0.4 * height_m
        // Assume average user height = 1.7m
        return 0.7f
    }
    
    data class StepDetected(
        val timestamp: Long,
        val length: Float,
        val heading: Float,
    )
}
```

### 4.3 PDRProcessor.kt (Main State Machine)

```kotlin
package com.plangrid.android.pdr.service

import com.plangrid.android.pdr.PDRUpdate
import com.plangrid.android.pdr.pdr.StepDetector
import com.plangrid.android.pdr.pdr.ZeroVelocityDetector
import com.plangrid.android.pdr.sensor.CalibrationManager
import com.plangrid.android.pdr.sensor.ProcessedSensor
import kotlinx.coroutines.sync.Mutex
import kotlinx.coroutines.sync.withLock
import kotlin.math.cos
import kotlin.math.sin

class PDRProcessor(
    private val calibration: CalibrationManager,
) {
    
    private val stateMutex = Mutex()
    
    // Position state
    private var x = 0f
    private var y = 0f
    private var heading = 0f
    
    // Velocity state
    private var vx = 0f
    private var vy = 0f
    
    // Detection modules
    private val stepDetector = StepDetector()
    private val zeroVelocityDetector = ZeroVelocityDetector()
    
    // Statistics
    private var stepCount = 0
    private var confidence = 1f
    
    suspend fun update(processed: ProcessedSensor): PDRUpdate {
        return stateMutex.withLock {
            // 1. Detect zero-velocity (ZUPT)
            val zeroVelDetected = zeroVelocityDetector.onProcessedSensor(processed)
            
            // 2. Detect step
            val step = stepDetector.onProcessedSensor(processed)
            
            // 3. Integrate heading (gyro)
            heading = (heading + processed.rotationRate[2] * processed.dt * 57.3f) % 360f
            
            // 4. Update velocity and position
            if (zeroVelDetected) {
                // ZUPT: freeze velocity
                vx = 0f
                vy = 0f
                // Update gyro bias from stationary reading
                calibration.updateGyroBiasFromZeroVelocity(emptyList())
            } else if (step != null) {
                // Dead reckoning from step
                val dx = step.length * cos(heading.toRadians())
                val dy = step.length * sin(heading.toRadians())
                x += dx
                y += dy
                stepCount++
            }
            
            // 5. Confidence management
            confidence = if (zeroVelDetected) 0.95f else 0.85f - stepCount * 0.001f
            confidence = confidence.coerceIn(0f, 1f)
            
            PDRUpdate(
                timestamp = processed.timestamp,
                x = x,
                y = y,
                heading = heading,
                confidence = confidence,
                stepCount = stepCount,
                velocity = 0f,  // Simplified for moderate approach
                zeroVelocityDetected = zeroVelDetected,
                magReliable = processed.magHeadingIfReliable != null,
                estimatedAccuracy = 0.5f + stepCount * 0.01f,  // Rough estimate
            )
        }
    }
    
    private fun Float.toRadians(): Float = this * 3.14159f / 180f
}
```

---

## 5. Foreground Service

### 5.1 PDRForegroundService.kt

```kotlin
package com.plangrid.android.pdr.service

import android.app.Notification
import android.app.NotificationChannel
import android.app.NotificationManager
import android.app.Service
import android.content.Context
import android.content.Intent
import android.hardware.SensorManager
import android.os.Binder
import android.os.IBinder
import androidx.core.app.NotificationCompat
import com.plangrid.android.pdr.PDRUpdate
import kotlinx.coroutines.CoroutineScope
import kotlinx.coroutines.Dispatchers
import kotlinx.coroutines.Job
import kotlinx.coroutines.cancel
import kotlinx.coroutines.flow.MutableSharedFlow
import kotlinx.coroutines.flow.SharedFlow
import kotlinx.coroutines.launch

class PDRForegroundService : Service() {
    
    companion object {
        private const val NOTIFICATION_ID = 42
        private const val NOTIFICATION_CHANNEL = "pdr_channel"
        
        const val ACTION_START = "com.plangrid.android.pdr.START"
        const val ACTION_STOP = "com.plangrid.android.pdr.STOP"
    }
    
    private val binder = PDRBinder()
    private val sensorManager by lazy {
        getSystemService(Context.SENSOR_SERVICE) as SensorManager
    }
    
    private lateinit var sensorAcq: SensorAcquisitionThread
    private lateinit var sensorPipeline: SensorPipeline
    private lateinit var pdrProcessor: PDRProcessor
    
    private val pdrFlow = MutableSharedFlow<PDRUpdate>(replay = 1)
    
    private val serviceScope = CoroutineScope(Dispatchers.Default + Job())
    
    override fun onCreate() {
        super.onCreate()
        
        // Create notification channel
        createNotificationChannel()
        
        // Initialize components
        sensorAcq = SensorAcquisitionThread(sensorManager)
        sensorAcq.start()
        
        val calibration = CalibrationManager(this)
        sensorPipeline = SensorPipeline(calibration)
        pdrProcessor = PDRProcessor(calibration)
    }
    
    override fun onStartCommand(intent: Intent?, flags: Int, startId: Int): Int {
        when (intent?.action) {
            ACTION_START -> {
                startPDRTracking()
            }
            ACTION_STOP -> {
                stopPDRTracking()
            }
        }
        
        startForeground(NOTIFICATION_ID, buildNotification())
        return START_STICKY
    }
    
    private fun startPDRTracking() {
        sensorAcq.startSampling()
        
        serviceScope.launch {
            while (true) {
                val sensors = sensorAcq.pollBuffer()
                if (sensors.isEmpty()) {
                    kotlinx.coroutines.delay(5)
                    continue
                }
                
                val processed = sensorPipeline.process(sensors) ?: continue
                val update = pdrProcessor.update(processed)
                
                pdrFlow.emit(update)
            }
        }
    }
    
    private fun stopPDRTracking() {
        sensorAcq.stopSampling()
    }
    
    private fun buildNotification(): Notification {
        return NotificationCompat.Builder(this, NOTIFICATION_CHANNEL)
            .setSmallIcon(android.R.drawable.ic_menu_mylocation)
            .setContentTitle("PDR Navigation Active")
            .setContentText("Tracking your position...")
            .setPriority(NotificationCompat.PRIORITY_LOW)
            .setForegroundServiceBehavior(NotificationCompat.FOREGROUND_SERVICE_IMMEDIATE)
            .build()
    }
    
    private fun createNotificationChannel() {
        val channel = NotificationChannel(
            NOTIFICATION_CHANNEL,
            "PDR Navigation",
            NotificationManager.IMPORTANCE_LOW
        )
        val manager = getSystemService(Context.NOTIFICATION_SERVICE) as NotificationManager
        manager.createNotificationChannel(channel)
    }
    
    override fun onBind(intent: Intent?): IBinder = binder
    
    override fun onDestroy() {
        super.onDestroy()
        sensorAcq.shutdown()
        serviceScope.cancel()
    }
    
    // Binder for client connections
    inner class PDRBinder : Binder() {
        fun getPDRUpdates(): SharedFlow<PDRUpdate> = pdrFlow
        
        fun getService(): PDRForegroundService = this@PDRForegroundService
    }
}
```

### 5.2 AndroidManifest.xml Configuration

```xml
<manifest>
    <!-- Permissions -->
    <uses-permission android:name="android.permission.ACCESS_FINE_LOCATION" />
    <uses-permission android:name="android.permission.ACCESS_COARSE_LOCATION" />
    <uses-permission android:name="android.permission.BODY_SENSORS" />
    <uses-permission android:name="android.permission.POST_NOTIFICATIONS" />
    <uses-permission android:name="android.permission.FOREGROUND_SERVICE" />
    
    <application>
        <!-- Service declaration -->
        <service
            android:name=".pdr.service.PDRForegroundService"
            android:foregroundServiceType="location"
            android:enabled="true"
            android:exported="false" />
    </application>
</manifest>
```

---

## 6. Client Usage Example

### 6.1 PDRClient.kt

```kotlin
package com.plangrid.android.pdr.client

import android.content.Context
import android.content.Intent
import android.content.ServiceConnection
import android.os.IBinder
import com.plangrid.android.pdr.PDRUpdate
import com.plangrid.android.pdr.service.PDRForegroundService
import kotlinx.coroutines.flow.SharedFlow
import kotlinx.coroutines.flow.collect

class PDRClient(private val context: Context) {
    
    private var service: PDRForegroundService? = null
    private var bound = false
    
    private val connection = object : ServiceConnection {
        override fun onServiceConnected(name: android.content.ComponentName?, service: IBinder?) {
            this@PDRClient.service = (service as PDRForegroundService.PDRBinder).getService()
            bound = true
        }
        
        override fun onServiceDisconnected(name: android.content.ComponentName?) {
            bound = false
        }
    }
    
    fun startTracking() {
        val intent = Intent(context, PDRForegroundService::class.java).apply {
            action = PDRForegroundService.ACTION_START
        }
        context.startForegroundService(intent)
        context.bindService(intent, connection, Context.BIND_AUTO_CREATE)
    }
    
    suspend fun observePDRUpdates(onUpdate: suspend (PDRUpdate) -> Unit) {
        while (!bound) {
            kotlinx.coroutines.delay(10)
        }
        
        service?.let { svc ->
            val updates: SharedFlow<PDRUpdate> = (svc as? PDRForegroundService.PDRBinder)
                ?.getPDRUpdates()
                ?: return
            
            updates.collect { onUpdate(it) }
        }
    }
    
    fun stopTracking() {
        if (bound) {
            context.unbindService(connection)
            bound = false
        }
        
        val intent = Intent(context, PDRForegroundService::class.java).apply {
            action = PDRForegroundService.ACTION_STOP
        }
        context.stopService(intent)
    }
}
```

---

## 7. Testing Strategy

### 7.1 Unit Tests (SensorPipelineTest.kt)

```kotlin
package com.plangrid.android.pdr.sensor

import org.junit.Test
import kotlin.test.assertTrue

class SensorPipelineTest {
    
    @Test
    fun testGravityExtraction() {
        val pipeline = SensorPipeline()
        
        // Simulate stationary device (only gravity)
        val snapshot = SensorSnapshot(
            timestampNs = 1000000000L,
            accelX = 0f, accelY = 0f, accelZ = 9.81f,
            gyroX = 0f, gyroY = 0f, gyroZ = 0f,
            magX = null, magY = null, magZ = null,
            pressure = null, temperature = null
        )
        
        val processed = pipeline.process(listOf(snapshot))
        assertTrue(processed?.accelVariance ?: 0f < 0.1f, "Gravity should stabilize variance")
    }
    
    @Test
    fun testHeadingIntegration() {
        // Simulate rotation around Z-axis
        // Heading should increase over time
    }
}
```

---

## 8. Build Configuration

### 8.1 build.gradle.kts

```kotlin
plugins {
    id("com.android.library")
    kotlin("android")
}

android {
    namespace = "com.plangrid.android.pdr"
    compileSdk = 34
    
    defaultConfig {
        minSdk = 31  // Android 12+ for foreground service restrictions
    }
}

dependencies {
    // Coroutines for Flow
    implementation("org.jetbrains.kotlinx:kotlinx-coroutines-android:1.7.1")
    implementation("org.jetbrains.kotlinx:kotlinx-coroutines-core:1.7.1")
    
    // AndroidX
    implementation("androidx.core:core:1.10.0")
    
    // Testing
    testImplementation("junit:junit:4.13.2")
    testImplementation("org.jetbrains.kotlin:kotlin-test:1.9.0")
}
```

---

## 9. Next Steps for Phase 2

1. **Kalman Filter Implementation**: Move from simple integration to full UKF
2. **Map Integration**: Particle filter for constraint satisfaction
3. **Multi-floor Support**: Barometer-based floor detection
4. **Ground-truth Validation**: Instrumented walks with GPS verification

---

**End of Implementation Guide**
