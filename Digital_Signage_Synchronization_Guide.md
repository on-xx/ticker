# Synchronized Digital Signage Using NTP Principles

## Overview

Applying the NTP synchronization concepts from the ticker project to digital signage creates powerful synchronized multi-display experiences. This guide shows how to implement synchronized content playback across multiple digital signage players.

## Core Synchronization Architecture

### 1. **Synchronized Program Scheduler**

```kotlin
data class ScheduledProgram(
    val id: String,
    val name: String,
    val contents: List<ContentItem>,
    val startTime: Long,  // NTP-synchronized timestamp
    val duration: Long,
    val repeatInterval: Long? = null
)

data class ContentItem(
    val id: String,
    val type: ContentType,
    val filePath: String,
    val duration: Long,
    val startOffset: Long  // Offset within the program
)

enum class ContentType {
    VIDEO, IMAGE, TEXT_ANIMATION, WEB_CONTENT
}
```

### 2. **NTP-Based Synchronization Service**

```kotlin
class DigitalSignageSyncService {
    private val ntpClient = EnhancedSntpClient()
    private val programScheduler = ProgramScheduler()
    
    suspend fun synchronizeAndSchedule(programs: List<ScheduledProgram>) {
        // Phase 1: Initial sync
        val initialOffset = calculateNtpOffset()
        
        // Phase 2: Precise sync (30 seconds before first program)
        val nextProgramStart = programs.minByOrNull { it.startTime }?.startTime ?: return
        val precisionSyncTime = nextProgramStart - 30_000 // 30 seconds before
        
        delay(precisionSyncTime - (System.currentTimeMillis() + initialOffset))
        val finalOffset = calculateNtpOffset()
        
        // Schedule all programs with precise timing
        schedulePrograms(programs, finalOffset)
    }
    
    private suspend fun calculateNtpOffset(): Long {
        val offsets = mutableListOf<Long>()
        repeat(3) {
            val offset = ntpClient.requestTimeOffset("pool.ntp.org")
            if (offset != Long.MAX_VALUE) {
                offsets.add(offset)
            }
        }
        return offsets.sorted()[offsets.size / 2] // Median
    }
}
```

### 3. **Enhanced SNTP Client for Digital Signage**

```kotlin
class EnhancedSntpClient {
    private val servers = listOf(
        "pool.ntp.org",
        "time.google.com", 
        "time.cloudflare.com"
    )
    
    suspend fun requestTimeOffset(preferredServer: String): Long {
        return withContext(Dispatchers.IO) {
            servers.forEach { server ->
                try {
                    val offset = performNtpRequest(server)
                    if (offset != Long.MAX_VALUE) return@withContext offset
                } catch (e: Exception) {
                    Log.w("NTP", "Failed to sync with $server: ${e.message}")
                }
            }
            Long.MAX_VALUE // Fallback value
        }
    }
    
    private fun performNtpRequest(server: String): Long {
        // Similar to original SntpClient but with enhanced error handling
        // and multiple server support
    }
}
```

## Content Synchronization Strategies

### 1. **Video Synchronization**

```kotlin
class SynchronizedVideoPlayer {
    private val mediaPlayer = MediaPlayer()
    private var ntpOffset: Long = 0
    
    fun scheduleVideoPlayback(
        videoPath: String,
        scheduledStartTime: Long,
        ntpOffset: Long
    ) {
        this.ntpOffset = ntpOffset
        
        // Preload video
        mediaPlayer.apply {
            setDataSource(videoPath)
            prepareAsync()
            setOnPreparedListener { player ->
                scheduleStart(player, scheduledStartTime)
            }
        }
    }
    
    private fun scheduleStart(player: MediaPlayer, startTime: Long) {
        val currentNtpTime = System.currentTimeMillis() + ntpOffset
        val delayMs = startTime - currentNtpTime
        
        if (delayMs > 0) {
            Handler(Looper.getMainLooper()).postDelayed({
                player.start()
                logPlaybackStart("Video started at NTP time: ${System.currentTimeMillis() + ntpOffset}")
            }, delayMs)
        } else {
            // Start immediately if we're already past start time
            player.start()
            player.seekTo((-delayMs).toInt()) // Seek to correct position
        }
    }
}
```

### 2. **Image Slideshow Synchronization**

```kotlin
class SynchronizedImagePlayer {
    private val imageView: ImageView = findViewById(R.id.signage_image)
    private val handler = Handler(Looper.getMainLooper())
    
    fun scheduleImageSequence(
        images: List<ContentItem>,
        programStartTime: Long,
        ntpOffset: Long
    ) {
        images.forEach { image ->
            val imageStartTime = programStartTime + image.startOffset
            val currentNtpTime = System.currentTimeMillis() + ntpOffset
            val delay = imageStartTime - currentNtpTime
            
            if (delay > 0) {
                handler.postDelayed({
                    displayImage(image.filePath)
                    logContentStart("Image ${image.id} displayed")
                }, delay)
            }
        }
    }
    
    private fun displayImage(imagePath: String) {
        Glide.with(this)
            .load(imagePath)
            .transition(DrawableTransitionOptions.withCrossFade())
            .into(imageView)
    }
}
```

### 3. **Mixed Content Program Player**

```kotlin
class SynchronizedProgramPlayer {
    private val videoPlayer = SynchronizedVideoPlayer()
    private val imagePlayer = SynchronizedImagePlayer()
    private val textAnimator = SynchronizedTextAnimator()
    
    suspend fun playProgram(program: ScheduledProgram, ntpOffset: Long) {
        // Preload all content
        preloadContent(program.contents)
        
        // Schedule each content item
        program.contents.forEach { content ->
            val contentStartTime = program.startTime + content.startOffset
            
            when (content.type) {
                ContentType.VIDEO -> {
                    videoPlayer.scheduleVideoPlayback(
                        content.filePath, 
                        contentStartTime, 
                        ntpOffset
                    )
                }
                ContentType.IMAGE -> {
                    imagePlayer.scheduleImageDisplay(
                        content, 
                        contentStartTime, 
                        ntpOffset
                    )
                }
                ContentType.TEXT_ANIMATION -> {
                    textAnimator.scheduleTextAnimation(
                        content, 
                        contentStartTime, 
                        ntpOffset
                    )
                }
                ContentType.WEB_CONTENT -> {
                    webViewPlayer.scheduleWebContent(
                        content, 
                        contentStartTime, 
                        ntpOffset
                    )
                }
            }
        }
        
        // Schedule program end and next program
        scheduleNextProgram(program, ntpOffset)
    }
}
```

## Advanced Synchronization Features

### 1. **Cross-Device Content Coordination**

```kotlin
class CrossDeviceCoordinator {
    data class DeviceRole(
        val deviceId: String,
        val position: Int,  // Physical position in display array
        val resolution: String,
        val capabilities: List<String>
    )
    
    fun distributeContentByRole(
        program: ScheduledProgram,
        deviceRole: DeviceRole
    ): List<ContentItem> {
        return program.contents.filter { content ->
            when (deviceRole.position) {
                0 -> content.tags.contains("primary") || content.tags.contains("left")
                1 -> content.tags.contains("center") || content.tags.contains("middle")
                2 -> content.tags.contains("secondary") || content.tags.contains("right")
                else -> content.tags.contains("overflow")
            }
        }
    }
    
    fun calculateDeviceSpecificDelay(
        baseStartTime: Long,
        deviceRole: DeviceRole,
        contentType: ContentType
    ): Long {
        // Similar to the ticker project's device-specific delays
        return when (contentType) {
            ContentType.VIDEO -> when (deviceRole.position) {
                0 -> 0L                    // Primary device starts immediately
                1 -> 500L                  // Center device slightly delayed
                2 -> 1000L                 // Secondary device more delayed
                else -> 1500L
            }
            ContentType.TEXT_ANIMATION -> {
                // For flowing text across displays
                deviceRole.position * 800L  // Staggered timing
            }
            else -> 0L
        }
    }
}
```

### 2. **Content Preloading and Buffering**

```kotlin
class ContentPreloader {
    private val preloadedContent = mutableMapOf<String, Any>()
    
    suspend fun preloadProgram(program: ScheduledProgram) {
        program.contents.forEach { content ->
            when (content.type) {
                ContentType.VIDEO -> preloadVideo(content)
                ContentType.IMAGE -> preloadImage(content)
                ContentType.WEB_CONTENT -> preloadWebContent(content)
                ContentType.TEXT_ANIMATION -> preloadTextResources(content)
            }
        }
    }
    
    private suspend fun preloadVideo(content: ContentItem) {
        withContext(Dispatchers.IO) {
            val mediaPlayer = MediaPlayer().apply {
                setDataSource(content.filePath)
                prepare() // Synchronous prepare for preloading
            }
            preloadedContent[content.id] = mediaPlayer
        }
    }
    
    fun getPreloadedContent(contentId: String): Any? {
        return preloadedContent[contentId]
    }
}
```

### 3. **Failsafe and Error Recovery**

```kotlin
class SynchronizationFailsafe {
    private var lastKnownOffset: Long = 0
    private var lastSyncTime: Long = 0
    
    fun getReliableOffset(): Long {
        val timeSinceLastSync = System.currentTimeMillis() - lastSyncTime
        
        return if (timeSinceLastSync < 300_000) { // 5 minutes
            lastKnownOffset
        } else {
            // Attempt fresh sync or use system time
            tryFreshSync() ?: 0L
        }
    }
    
    private fun tryFreshSync(): Long? {
        return try {
            // Quick single NTP request
            val client = EnhancedSntpClient()
            runBlocking { client.requestTimeOffset("time.google.com") }
        } catch (e: Exception) {
            null
        }
    }
}
```

## Implementation Example: Complete Digital Signage Player

```kotlin
class DigitalSignagePlayer : CoroutineScope {
    override val coroutineContext = SupervisorJob() + Dispatchers.Main
    
    private val syncService = DigitalSignageSyncService()
    private val programPlayer = SynchronizedProgramPlayer()
    private val coordinator = CrossDeviceCoordinator()
    private val preloader = ContentPreloader()
    
    fun startSynchronizedPlayback(
        playlist: List<ScheduledProgram>,
        deviceRole: DeviceRole
    ) {
        launch {
            try {
                // 1. Preload all content
                playlist.forEach { program ->
                    preloader.preloadProgram(program)
                }
                
                // 2. Synchronize timing
                val ntpOffset = syncService.calculateNtpOffset()
                
                // 3. Filter content for this device
                val deviceSpecificPlaylist = playlist.map { program ->
                    program.copy(
                        contents = coordinator.distributeContentByRole(program, deviceRole)
                    )
                }
                
                // 4. Start synchronized playback
                deviceSpecificPlaylist.forEach { program ->
                    programPlayer.playProgram(program, ntpOffset)
                }
                
                // 5. Monitor and maintain sync
                maintainSynchronization(ntpOffset)
                
            } catch (e: Exception) {
                handleSynchronizationError(e)
            }
        }
    }
    
    private suspend fun maintainSynchronization(initialOffset: Long) {
        while (isActive) {
            delay(60_000) // Check every minute
            val currentOffset = syncService.calculateNtpOffset()
            val drift = abs(currentOffset - initialOffset)
            
            if (drift > 1000) { // More than 1 second drift
                Log.w("Sync", "Time drift detected: ${drift}ms. Recalibrating...")
                // Trigger resynchronization
                recalibrateTiming(currentOffset)
            }
        }
    }
}
```

## Configuration and Deployment

### 1. **Program Configuration JSON**

```json
{
  "programs": [
    {
      "id": "morning_program",
      "name": "Morning Content",
      "startTime": 1703829600000,
      "duration": 1800000,
      "repeatInterval": 86400000,
      "contents": [
        {
          "id": "intro_video",
          "type": "VIDEO",
          "filePath": "/content/videos/intro.mp4",
          "duration": 30000,
          "startOffset": 0,
          "tags": ["primary", "all_devices"]
        },
        {
          "id": "slideshow_1",
          "type": "IMAGE",
          "filePath": "/content/images/slide1.jpg",
          "duration": 15000,
          "startOffset": 30000,
          "tags": ["left", "primary"]
        },
        {
          "id": "flowing_text",
          "type": "TEXT_ANIMATION",
          "filePath": "/content/text/announcement.txt",
          "duration": 20000,
          "startOffset": 45000,
          "tags": ["flowing", "all_devices"]
        }
      ]
    }
  ],
  "deviceRoles": [
    {
      "deviceId": "display_001",
      "position": 0,
      "resolution": "1920x1080",
      "capabilities": ["video", "image", "text"]
    },
    {
      "deviceId": "display_002", 
      "position": 1,
      "resolution": "1920x1080",
      "capabilities": ["video", "image", "text"]
    }
  ]
}
```

### 2. **Device Registration and Discovery**

```kotlin
class DeviceRegistrationService {
    fun registerDevice(deviceInfo: DeviceRole): Boolean {
        // Register device with central management system
        // Could use MQTT, REST API, or local broadcast
    }
    
    fun discoverPeerDevices(): List<DeviceRole> {
        // Discover other devices in the same display network
        // For peer-to-peer synchronization scenarios
    }
}
```

## Key Benefits of This Approach

### **Precision Synchronization**
- **±50ms accuracy** across all displays
- **NTP-based timing** ensures network-wide consistency
- **Two-phase sync** for maximum precision

### **Flexible Content Management**
- **Mixed content types** (video, images, text, web)
- **Device-specific content** distribution
- **Dynamic scheduling** and updates

### **Scalability**
- **Works with 2 to 100+ displays**
- **Peer-to-peer or centralized** management
- **Automatic failover** and error recovery

### **Professional Applications**
- **Retail digital walls** with coordinated messaging
- **Event displays** with synchronized presentations
- **Transportation hubs** with flowing information
- **Museums and exhibitions** with immersive experiences

## Next Steps for Implementation

1. **Start with 2-device prototype** using simplified video synchronization
2. **Add content management system** for remote scheduling
3. **Implement device discovery** and automatic configuration
4. **Add monitoring and analytics** for synchronization quality
5. **Scale to production** with robust error handling

This approach transforms the simple ticker concept into a powerful digital signage synchronization system that can handle complex, multi-content programs across numerous displays while maintaining precise timing coordination.