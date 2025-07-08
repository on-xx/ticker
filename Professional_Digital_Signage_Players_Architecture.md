# Professional Digital Signage Players Architecture

## Overview

Professional digital signage systems require specialized players for different content types, each optimized for performance, synchronization, and reliability. This guide covers the complete player architecture used in commercial digital signage products.

## Content Type Classification and Player Requirements

### **Content Categories**
```kotlin
enum class ContentType(
    val playerType: String,
    val syncPrecision: SyncPrecision,
    val resourceRequirement: ResourceLevel
) {
    // Video Content
    VIDEO_FILE("VideoFilePlayer", SyncPrecision.FRAME_ACCURATE, ResourceLevel.HIGH),
    VIDEO_STREAM_LIVE("LiveStreamPlayer", SyncPrecision.FRAME_ACCURATE, ResourceLevel.VERY_HIGH),
    VIDEO_STREAM_VOD("VODStreamPlayer", SyncPrecision.FRAME_ACCURATE, ResourceLevel.HIGH),
    
    // Image Content  
    IMAGE_STATIC("StaticImagePlayer", SyncPrecision.MILLISECOND, ResourceLevel.LOW),
    IMAGE_SLIDESHOW("SlideshowPlayer", SyncPrecision.MILLISECOND, ResourceLevel.MEDIUM),
    IMAGE_ANIMATED_GIF("AnimatedImagePlayer", SyncPrecision.FRAME_ACCURATE, ResourceLevel.MEDIUM),
    
    // Web Content
    WEB_PAGE("WebContentPlayer", SyncPrecision.SECOND, ResourceLevel.HIGH),
    WEB_APPLICATION("WebAppPlayer", SyncPrecision.MILLISECOND, ResourceLevel.VERY_HIGH),
    
    // Interactive Content
    TOUCH_INTERACTIVE("InteractivePlayer", SyncPrecision.MILLISECOND, ResourceLevel.HIGH),
    SENSOR_RESPONSIVE("SensorPlayer", SyncPrecision.MILLISECOND, ResourceLevel.MEDIUM),
    
    // Real-time Content
    DATA_VISUALIZATION("DataVizPlayer", SyncPrecision.SECOND, ResourceLevel.MEDIUM),
    SOCIAL_FEED("SocialFeedPlayer", SyncPrecision.SECOND, ResourceLevel.MEDIUM),
    
    // Audio Content
    AUDIO_FILE("AudioFilePlayer", SyncPrecision.FRAME_ACCURATE, ResourceLevel.MEDIUM),
    AUDIO_STREAM("AudioStreamPlayer", SyncPrecision.FRAME_ACCURATE, ResourceLevel.HIGH),
    
    // 3D/AR Content
    THREE_D_MODEL("ThreeDPlayer", SyncPrecision.FRAME_ACCURATE, ResourceLevel.VERY_HIGH),
    AUGMENTED_REALITY("ARPlayer", SyncPrecision.FRAME_ACCURATE, ResourceLevel.VERY_HIGH)
}

enum class SyncPrecision(val toleranceMs: Long) {
    FRAME_ACCURATE(16),    // ±16ms (1 frame at 60fps)
    MILLISECOND(100),      // ±100ms
    SECOND(1000)           // ±1s
}
```

## Video Content Players

### **1. High-Performance Video File Player**

```kotlin
class ProfessionalVideoPlayer : SynchronizedContentPlayer {
    private var exoPlayer: ExoPlayer? = null
    private var mediaPlayer: MediaPlayer? = null
    private var vlcPlayer: VLCVideoPlayer? = null
    
    // Multiple player backends for different scenarios
    private val playerBackends = mapOf(
        VideoBackend.EXOPLAYER to ::createExoPlayer,
        VideoBackend.MEDIAPLAYER to ::createMediaPlayer,
        VideoBackend.VLC to ::createVLCPlayer,
        VideoBackend.FFMPEG to ::createFFmpegPlayer
    )
    
    override suspend fun initialize(content: VideoContent) {
        val backend = selectOptimalBackend(content)
        val player = playerBackends[backend]?.invoke(content)
        
        when (backend) {
            VideoBackend.EXOPLAYER -> {
                exoPlayer = player as ExoPlayer
                configureExoPlayerForSync()
            }
            VideoBackend.VLC -> {
                vlcPlayer = player as VLCVideoPlayer
                configureVLCForSync()
            }
            // ... other backends
        }
        
        preloadContent(content)
    }
    
    private fun selectOptimalBackend(content: VideoContent): VideoBackend {
        return when {
            // 4K+ content with hardware acceleration
            content.resolution.width >= 3840 -> VideoBackend.EXOPLAYER
            
            // Legacy codecs or problematic files
            content.hasLegacyCodec() -> VideoBackend.VLC
            
            // Low-latency requirements
            content.requiresLowLatency -> VideoBackend.FFMPEG
            
            // Standard content
            else -> VideoBackend.EXOPLAYER
        }
    }
    
    private fun configureExoPlayerForSync() {
        val loadControl = DefaultLoadControl.Builder()
            .setBufferDurationsMs(
                50,     // Min buffer: 50ms for low latency
                2000,   // Max buffer: 2s
                50,     // Buffer for playback: 50ms
                50      // Buffer after rebuffer: 50ms
            )
            .build()
            
        exoPlayer = ExoPlayer.Builder(context)
            .setLoadControl(loadControl)
            .setRenderersFactory(CustomRenderersFactory()) // Hardware acceleration
            .build()
            
        // Frame-accurate seeking
        exoPlayer?.seekParameters = SeekParameters.EXACT
        
        // Video frame callback for synchronization
        exoPlayer?.setVideoFrameMetadataListener { _, metadata ->
            handleFrameRendered(metadata.presentationTimeUs)
        }
    }
}

enum class VideoBackend {
    EXOPLAYER,    // Google's media player (recommended)
    MEDIAPLAYER,  // Android native (basic)
    VLC,          // VLC for Android (compatibility)
    FFMPEG        // FFmpeg-based (custom)
}
```

### **2. Live Streaming Player**

```kotlin
class LiveStreamPlayer : SynchronizedContentPlayer {
    private var hlsPlayer: ExoPlayer? = null
    private var webRtcPlayer: WebRTCPlayer? = null
    private var mpeg2tsPlayer: MPEG2TSPlayer? = null
    
    override suspend fun initialize(content: StreamContent) {
        when (content.protocol) {
            StreamProtocol.HLS -> initializeHLSPlayer(content)
            StreamProtocol.DASH -> initializeDASHPlayer(content)
            StreamProtocol.WEBRTC -> initializeWebRTCPlayer(content)
            StreamProtocol.RTMP -> initializeRTMPPlayer(content)
            StreamProtocol.SRT -> initializeSRTPlayer(content)
        }
    }
    
    private fun initializeHLSPlayer(content: StreamContent) {
        val dataSourceFactory = DefaultDataSourceFactory(context, "DigitalSignagePlayer")
        val hlsMediaSourceFactory = HlsMediaSource.Factory(dataSourceFactory)
        
        // Configure for low-latency streaming
        hlsMediaSourceFactory.setAllowChunklessPreparation(true)
        
        val mediaSource = hlsMediaSourceFactory.createMediaSource(
            MediaItem.fromUri(content.streamUrl)
        )
        
        hlsPlayer = ExoPlayer.Builder(context)
            .setLoadControl(createLowLatencyLoadControl())
            .build()
            
        hlsPlayer?.setMediaSource(mediaSource)
        hlsPlayer?.prepare()
    }
    
    private fun initializeWebRTCPlayer(content: StreamContent) {
        webRtcPlayer = WebRTCPlayer(context).apply {
            configure(WebRTCConfig(
                lowLatency = true,
                bufferSize = 100, // 100ms buffer
                audioSync = true,
                videoSync = true
            ))
            
            setStreamUrl(content.streamUrl)
            setOnFrameAvailableListener { frameTimeUs ->
                notifyFrameRendered(frameTimeUs)
            }
        }
    }
    
    override fun startSynchronizedPlayback(ntpStartTime: Long) {
        val currentNtpTime = System.currentTimeMillis() + ntpOffset
        val delay = ntpStartTime - currentNtpTime
        
        when {
            delay > 0 -> {
                // Schedule start
                Handler(Looper.getMainLooper()).postDelayed({
                    startPlayback()
                }, delay)
            }
            delay > -5000 -> {
                // Start with seek compensation (within 5 seconds)
                startPlayback()
                seekToLiveEdge() // Catch up to live stream
            }
            else -> {
                // Too far behind, restart stream
                restartStream()
            }
        }
    }
}
```

## Image and Static Content Players

### **3. Advanced Image Player with Transitions**

```kotlin
class ProfessionalImagePlayer : SynchronizedContentPlayer {
    private val imageViews = arrayOf<ImageView>(
        findViewById(R.id.image_layer_1),
        findViewById(R.id.image_layer_2)
    )
    private var currentLayerIndex = 0
    private val transitionEngine = ImageTransitionEngine()
    
    override suspend fun initialize(content: ImageContent) {
        when (content.type) {
            ImageType.STATIC -> preloadStaticImage(content)
            ImageType.SLIDESHOW -> preloadSlideshow(content)
            ImageType.ANIMATED_GIF -> preloadAnimatedImage(content)
            ImageType.SVG_ANIMATED -> preloadSVGAnimation(content)
        }
    }
    
    private suspend fun preloadSlideshow(content: SlideshowContent) {
        content.images.forEach { imageItem ->
            withContext(Dispatchers.IO) {
                // Preload with Glide for memory efficiency
                Glide.with(context)
                    .asBitmap()
                    .load(imageItem.filePath)
                    .diskCacheStrategy(DiskCacheStrategy.ALL)
                    .preload()
            }
        }
    }
    
    override fun scheduleContentTransition(
        fromContent: ImageContent,
        toContent: ImageContent,
        transitionTime: Long,
        ntpOffset: Long
    ) {
        val currentNtpTime = System.currentTimeMillis() + ntpOffset
        val delay = transitionTime - currentNtpTime
        
        Handler(Looper.getMainLooper()).postDelayed({
            executeImageTransition(fromContent, toContent)
        }, maxOf(0, delay))
    }
    
    private fun executeImageTransition(
        fromContent: ImageContent,
        toContent: ImageContent
    ) {
        val nextLayerIndex = (currentLayerIndex + 1) % imageViews.size
        val nextImageView = imageViews[nextLayerIndex]
        val currentImageView = imageViews[currentLayerIndex]
        
        // Load next image in background layer
        Glide.with(context)
            .load(toContent.filePath)
            .listener(object : RequestListener<Drawable> {
                override fun onResourceReady(
                    resource: Drawable?,
                    model: Any?,
                    target: Target<Drawable>?,
                    dataSource: DataSource?,
                    isFirstResource: Boolean
                ): Boolean {
                    // Execute transition when image is loaded
                    transitionEngine.executeTransition(
                        from = currentImageView,
                        to = nextImageView,
                        transition = toContent.transitionEffect,
                        duration = toContent.transitionDuration
                    )
                    currentLayerIndex = nextLayerIndex
                    return false
                }
                
                override fun onLoadFailed(
                    e: GlideException?,
                    model: Any?,
                    target: Target<Drawable>?,
                    isFirstResource: Boolean
                ): Boolean {
                    handleImageLoadError(e, toContent)
                    return false
                }
            })
            .into(nextImageView)
    }
}

class ImageTransitionEngine {
    fun executeTransition(
        from: ImageView,
        to: ImageView,
        transition: TransitionEffect,
        duration: Long
    ) {
        when (transition) {
            TransitionEffect.FADE -> executeFadeTransition(from, to, duration)
            TransitionEffect.SLIDE_LEFT -> executeSlideTransition(from, to, duration, -1f)
            TransitionEffect.SLIDE_RIGHT -> executeSlideTransition(from, to, duration, 1f)
            TransitionEffect.ZOOM_IN -> executeZoomTransition(from, to, duration, true)
            TransitionEffect.DISSOLVE -> executeDissolveTransition(from, to, duration)
        }
    }
    
    private fun executeFadeTransition(from: ImageView, to: ImageView, duration: Long) {
        to.alpha = 0f
        to.visibility = View.VISIBLE
        
        val animator = ValueAnimator.ofFloat(0f, 1f).apply {
            this.duration = duration
            interpolator = LinearInterpolator()
            
            addUpdateListener { animation ->
                val progress = animation.animatedValue as Float
                to.alpha = progress
                from.alpha = 1f - progress
            }
            
            addListener(object : AnimatorListenerAdapter() {
                override fun onAnimationEnd(animation: Animator) {
                    from.visibility = View.GONE
                    from.alpha = 1f
                }
            })
        }
        
        animator.start()
    }
}
```

## Web Content Players

### **4. Synchronized Web Content Player**

```kotlin
class SynchronizedWebPlayer : SynchronizedContentPlayer {
    private var webView: WebView? = null
    private var chromiumWebView: ChromiumWebView? = null
    private val webViewJSInterface = WebViewSyncInterface()
    
    override suspend fun initialize(content: WebContent) {
        when (content.renderingEngine) {
            WebEngine.SYSTEM_WEBVIEW -> initializeSystemWebView(content)
            WebEngine.CHROMIUM -> initializeChromiumWebView(content)
            WebEngine.GECKO -> initializeGeckoWebView(content)
        }
    }
    
    private fun initializeSystemWebView(content: WebContent) {
        webView = WebView(context).apply {
            settings.apply {
                javaScriptEnabled = true
                domStorageEnabled = true
                allowFileAccess = true
                allowContentAccess = true
                mediaPlaybackRequiresUserGesture = false
                mixedContentMode = WebSettings.MIXED_CONTENT_ALWAYS_ALLOW
                
                // Performance optimizations
                cacheMode = WebSettings.LOAD_DEFAULT
                setAppCacheEnabled(true)
                databaseEnabled = true
                
                // Rendering optimizations
                hardwareAccelerated = true
                setLayerType(View.LAYER_TYPE_HARDWARE, null)
            }
            
            // Add JavaScript interface for synchronization
            addJavascriptInterface(webViewJSInterface, "DigitalSignageSync")
            
            webViewClient = SynchronizedWebViewClient()
            webChromeClient = SynchronizedWebChromeClient()
        }
    }
    
    class WebViewSyncInterface {
        @JavascriptInterface
        fun notifyContentReady() {
            // Called when web content is ready for synchronization
            contentReadyCallback?.invoke()
        }
        
        @JavascriptInterface
        fun getCurrentTime(): Long {
            return System.currentTimeMillis() + ntpOffset
        }
        
        @JavascriptInterface
        fun scheduleAction(actionName: String, delayMs: Long) {
            // Schedule synchronized actions from web content
            Handler(Looper.getMainLooper()).postDelayed({
                executeWebAction(actionName)
            }, delayMs)
        }
    }
    
    override fun loadContent(content: WebContent) {
        val html = when (content.type) {
            WebContentType.URL -> {
                webView?.loadUrl(content.url)
                return
            }
            WebContentType.HTML_FILE -> loadHTMLFile(content.filePath)
            WebContentType.HTML_STRING -> content.htmlContent
            WebContentType.WIDGET -> generateWidgetHTML(content)
        }
        
        webView?.loadDataWithBaseURL(
            "file:///android_asset/",
            html,
            "text/html",
            "UTF-8",
            null
        )
    }
    
    private fun generateWidgetHTML(content: WebContent): String {
        return """
        <!DOCTYPE html>
        <html>
        <head>
            <meta charset="UTF-8">
            <meta name="viewport" content="width=device-width, initial-scale=1.0">
            <style>
                body { 
                    margin: 0; 
                    padding: 0; 
                    font-family: Arial, sans-serif;
                    background: ${content.backgroundColor ?: "#000000"};
                    color: ${content.textColor ?: "#FFFFFF"};
                }
                .sync-container {
                    width: 100vw;
                    height: 100vh;
                    display: flex;
                    align-items: center;
                    justify-content: center;
                }
            </style>
        </head>
        <body>
            <div class="sync-container" id="content-container">
                ${content.widgetContent}
            </div>
            
            <script>
                // Synchronization JavaScript
                let syncInterface = window.DigitalSignageSync;
                let startTime = null;
                
                function initializeSync() {
                    if (syncInterface) {
                        syncInterface.notifyContentReady();
                    }
                }
                
                function startSynchronizedContent(ntpStartTime) {
                    startTime = ntpStartTime;
                    let currentTime = syncInterface.getCurrentTime();
                    let delay = ntpStartTime - currentTime;
                    
                    if (delay > 0) {
                        setTimeout(startContent, delay);
                    } else {
                        startContent();
                    }
                }
                
                function startContent() {
                    // Content-specific initialization
                    ${content.initializationScript ?: ""}
                }
                
                // Initialize when page loads
                window.addEventListener('load', initializeSync);
            </script>
        </body>
        </html>
        """.trimIndent()
    }
}
```

## Interactive Content Players

### **5. Touch and Sensor Interactive Player**

```kotlin
class InteractiveContentPlayer : SynchronizedContentPlayer {
    private val gestureDetector = GestureDetectorCompat(context, InteractiveGestureListener())
    private val sensorManager = context.getSystemService(Context.SENSOR_SERVICE) as SensorManager
    private val interactionLogger = InteractionLogger()
    
    override suspend fun initialize(content: InteractiveContent) {
        when (content.interactionType) {
            InteractionType.TOUCH -> initializeTouchInteraction(content)
            InteractionType.GESTURE -> initializeGestureInteraction(content)
            InteractionType.PROXIMITY -> initializeProximityInteraction(content)
            InteractionType.MOTION -> initializeMotionInteraction(content)
            InteractionType.VOICE -> initializeVoiceInteraction(content)
        }
    }
    
    private fun initializeTouchInteraction(content: InteractiveContent) {
        val touchOverlay = TouchInteractionOverlay(context).apply {
            setOnInteractionListener { interaction ->
                handleInteraction(interaction, content)
            }
        }
        
        // Add touch zones based on content configuration
        content.touchZones.forEach { zone ->
            touchOverlay.addTouchZone(TouchZone(
                id = zone.id,
                bounds = zone.bounds,
                action = zone.action,
                feedback = zone.feedbackType
            ))
        }
    }
    
    private fun handleInteraction(
        interaction: UserInteraction,
        content: InteractiveContent
    ) {
        // Log interaction with precise timing
        interactionLogger.logInteraction(
            interaction = interaction,
            ntpTime = System.currentTimeMillis() + ntpOffset,
            contentId = content.id
        )
        
        // Execute interaction response
        when (interaction.type) {
            InteractionType.TOUCH -> handleTouchInteraction(interaction, content)
            InteractionType.GESTURE -> handleGestureInteraction(interaction, content)
            // ... other interaction types
        }
        
        // Synchronize interaction effects across displays if needed
        if (content.syncAcrossDisplays) {
            synchronizeInteractionEffect(interaction)
        }
    }
    
    private fun synchronizeInteractionEffect(interaction: UserInteraction) {
        val syncMessage = InteractionSyncMessage(
            type = interaction.type,
            data = interaction.data,
            timestamp = System.currentTimeMillis() + ntpOffset,
            originDisplayId = deviceId
        )
        
        // Broadcast to other displays
        interactionSyncBroadcaster.broadcast(syncMessage)
    }
}

class InteractionLogger {
    fun logInteraction(
        interaction: UserInteraction,
        ntpTime: Long,
        contentId: String
    ) {
        val logEntry = InteractionLogEntry(
            interactionId = UUID.randomUUID().toString(),
            contentId = contentId,
            displayId = deviceId,
            interactionType = interaction.type,
            coordinates = interaction.coordinates,
            timestamp = ntpTime,
            sessionId = currentSessionId
        )
        
        // Store locally and sync to analytics server
        localInteractionDb.insert(logEntry)
        analyticsUploader.queueForUpload(logEntry)
    }
}
```

## Real-Time Data Players

### **6. Live Data Visualization Player**

```kotlin
class DataVisualizationPlayer : SynchronizedContentPlayer {
    private val dataSourceManager = DataSourceManager()
    private val chartEngine = ChartEngine()
    private val realtimeUpdater = RealtimeDataUpdater()
    
    override suspend fun initialize(content: DataVisualizationContent) {
        // Initialize data sources
        content.dataSources.forEach { dataSource ->
            dataSourceManager.registerDataSource(dataSource)
        }
        
        // Setup visualization components
        when (content.visualizationType) {
            VisualizationType.CHART -> initializeChartVisualization(content)
            VisualizationType.MAP -> initializeMapVisualization(content)
            VisualizationType.DASHBOARD -> initializeDashboard(content)
            VisualizationType.TICKER -> initializeTicker(content)
        }
        
        // Start real-time data updates
        realtimeUpdater.startUpdates(content.updateInterval)
    }
    
    private fun initializeChartVisualization(content: DataVisualizationContent) {
        val chartView = LineChart(context).apply {
            description.isEnabled = false
            setTouchEnabled(false)
            setDragEnabled(false)
            setScaleEnabled(false)
            setPinchZoom(false)
            
            // Configure for real-time updates
            setMaxVisibleValueCount(content.maxDataPoints)
            setDrawGridBackground(false)
            
            // Styling
            legend.isEnabled = content.showLegend
            setBackgroundColor(Color.parseColor(content.backgroundColor))
        }
        
        // Setup data update listener
        realtimeUpdater.setDataUpdateListener { newData ->
            updateChartData(chartView, newData)
        }
    }
    
    private fun updateChartData(chart: LineChart, newData: List<DataPoint>) {
        val entries = newData.map { dataPoint ->
            Entry(dataPoint.timestamp.toFloat(), dataPoint.value.toFloat())
        }
        
        val dataSet = LineDataSet(entries, "Real-time Data").apply {
            color = Color.parseColor(content.primaryColor)
            setDrawCircles(false)
            lineWidth = 2f
            mode = LineDataSet.Mode.CUBIC_BEZIER
        }
        
        chart.data = LineData(dataSet)
        chart.notifyDataSetChanged()
        chart.invalidate() // Refresh chart
    }
    
    override fun synchronizeDataUpdate(updateTime: Long, ntpOffset: Long) {
        val currentNtpTime = System.currentTimeMillis() + ntpOffset
        val delay = updateTime - currentNtpTime
        
        Handler(Looper.getMainLooper()).postDelayed({
            // Fetch and display latest data simultaneously across all displays
            realtimeUpdater.triggerSynchronizedUpdate()
        }, maxOf(0, delay))
    }
}

class DataSourceManager {
    private val dataSources = mutableMapOf<String, DataSource>()
    
    fun registerDataSource(config: DataSourceConfig) {
        val dataSource = when (config.type) {
            DataSourceType.REST_API -> RestApiDataSource(config)
            DataSourceType.WEBSOCKET -> WebSocketDataSource(config)
            DataSourceType.MQTT -> MQTTDataSource(config)
            DataSourceType.DATABASE -> DatabaseDataSource(config)
            DataSourceType.FILE -> FileDataSource(config)
        }
        
        dataSources[config.id] = dataSource
    }
    
    suspend fun fetchData(sourceId: String): List<DataPoint> {
        return dataSources[sourceId]?.fetchLatestData() ?: emptyList()
    }
}
```

## Audio Content Players

### **7. Synchronized Audio Player**

```kotlin
class SynchronizedAudioPlayer : SynchronizedContentPlayer {
    private var mediaPlayer: MediaPlayer? = null
    private var audioTrack: AudioTrack? = null
    private val audioFocusManager = AudioFocusManager(context)
    
    override suspend fun initialize(content: AudioContent) {
        when (content.type) {
            AudioType.BACKGROUND_MUSIC -> initializeBackgroundAudio(content)
            AudioType.SOUND_EFFECT -> initializeSoundEffect(content)
            AudioType.VOICE_NARRATION -> initializeNarration(content)
            AudioType.LIVE_STREAM -> initializeLiveAudioStream(content)
        }
    }
    
    private fun initializeBackgroundAudio(content: AudioContent) {
        mediaPlayer = MediaPlayer().apply {
            setDataSource(content.filePath)
            setAudioStreamType(AudioManager.STREAM_MUSIC)
            prepareAsync()
            
            setOnPreparedListener { player ->
                // Configure for synchronized playback
                player.seekTo(0)
                player.isLooping = content.loop
                
                // Notify ready for synchronization
                notifyPlayerReady()
            }
            
            setOnErrorListener { _, what, extra ->
                Log.e("AudioPlayer", "MediaPlayer error: $what, $extra")
                handleAudioError(what, extra)
                true
            }
        }
    }
    
    override fun startSynchronizedPlayback(ntpStartTime: Long) {
        val currentNtpTime = System.currentTimeMillis() + ntpOffset
        val delay = ntpStartTime - currentNtpTime
        
        if (delay > 0) {
            Handler(Looper.getMainLooper()).postDelayed({
                executeAudioStart()
            }, delay)
        } else {
            // Compensate for late start
            val compensationMs = (-delay).toInt()
            executeAudioStart(compensationMs)
        }
    }
    
    private fun executeAudioStart(seekOffsetMs: Int = 0) {
        audioFocusManager.requestAudioFocus { focusResult ->
            if (focusResult == AudioManager.AUDIOFOCUS_REQUEST_GRANTED) {
                mediaPlayer?.apply {
                    if (seekOffsetMs > 0) {
                        seekTo(seekOffsetMs)
                    }
                    start()
                }
                
                startAudioSyncMonitoring()
            }
        }
    }
    
    private fun startAudioSyncMonitoring() {
        val monitoringHandler = Handler(Looper.getMainLooper())
        val monitoringRunnable = object : Runnable {
            override fun run() {
                mediaPlayer?.let { player ->
                    val currentPosition = player.currentPosition
                    val expectedPosition = calculateExpectedPosition()
                    val drift = currentPosition - expectedPosition
                    
                    if (Math.abs(drift) > 50) { // More than 50ms drift
                        correctAudioDrift(drift)
                    }
                }
                
                monitoringHandler.postDelayed(this, 100) // Check every 100ms
            }
        }
        
        monitoringHandler.post(monitoringRunnable)
    }
}
```

## Player Architecture Integration

### **8. Content Player Factory and Manager**

```kotlin
class ContentPlayerFactory {
    fun createPlayer(contentType: ContentType): SynchronizedContentPlayer {
        return when (contentType) {
            ContentType.VIDEO_FILE -> ProfessionalVideoPlayer()
            ContentType.VIDEO_STREAM_LIVE -> LiveStreamPlayer()
            ContentType.IMAGE_STATIC -> ProfessionalImagePlayer()
            ContentType.WEB_PAGE -> SynchronizedWebPlayer()
            ContentType.TOUCH_INTERACTIVE -> InteractiveContentPlayer()
            ContentType.DATA_VISUALIZATION -> DataVisualizationPlayer()
            ContentType.AUDIO_FILE -> SynchronizedAudioPlayer()
            ContentType.THREE_D_MODEL -> ThreeDModelPlayer()
            // ... other content types
        }
    }
}

class ContentPlayerManager {
    private val activePlayers = mutableMapOf<String, SynchronizedContentPlayer>()
    private val playerFactory = ContentPlayerFactory()
    private val syncCoordinator = SynchronizationCoordinator()
    
    suspend fun scheduleContent(
        content: Content,
        startTime: Long,
        endTime: Long,
        ntpOffset: Long
    ) {
        val player = playerFactory.createPlayer(content.type)
        
        // Initialize player
        player.initialize(content)
        
        // Register with sync coordinator
        syncCoordinator.registerPlayer(content.id, player)
        
        // Schedule playback
        val currentNtpTime = System.currentTimeMillis() + ntpOffset
        val delay = startTime - currentNtpTime
        
        Handler(Looper.getMainLooper()).postDelayed({
            player.startSynchronizedPlayback(startTime)
            activePlayers[content.id] = player
            
            // Schedule content end
            val duration = endTime - startTime
            Handler(Looper.getMainLooper()).postDelayed({
                stopContent(content.id)
            }, duration)
            
        }, maxOf(0, delay))
    }
    
    fun stopContent(contentId: String) {
        activePlayers[contentId]?.let { player ->
            player.stop()
            syncCoordinator.unregisterPlayer(contentId)
            activePlayers.remove(contentId)
        }
    }
}
```

## Production Performance Optimizations

### **Hardware Acceleration Configuration**
```kotlin
class PlayerOptimizationManager {
    fun optimizeForDevice(): PlayerOptimizations {
        val deviceSpecs = DeviceSpecAnalyzer.analyze()
        
        return PlayerOptimizations(
            // Video optimizations
            useHardwareDecoding = deviceSpecs.hasHardwareDecoder,
            videoBufferSize = calculateOptimalBufferSize(deviceSpecs),
            enableGPURendering = deviceSpecs.hasGPU,
            
            // Audio optimizations
            audioBufferSize = deviceSpecs.audioBufferSize,
            sampleRate = deviceSpecs.optimalSampleRate,
            
            // Memory optimizations
            enableMemoryMapping = deviceSpecs.availableRAM > 4096,
            imageCache = calculateImageCacheSize(deviceSpecs),
            
            // Network optimizations
            tcpWindowSize = deviceSpecs.networkCapacity,
            enableZeroLatency = deviceSpecs.supportLowLatency
        )
    }
}
```

## Key Architectural Benefits

### **Specialized Performance**
- **Video players** optimized for 4K+ content with frame-accurate synchronization
- **Web players** with JavaScript integration for interactive content
- **Data players** with real-time update capabilities
- **Audio players** with sub-frame audio synchronization

### **Production Reliability**
- **Multiple codec support** through different player backends
- **Automatic fallback** when hardware acceleration fails
- **Memory management** optimized for 24/7 operation
- **Error recovery** with seamless content switching

### **Synchronization Precision**
- **Frame-level accuracy** for video content (±16ms)
- **Millisecond precision** for interactive content
- **Real-time monitoring** and drift correction
- **Cross-player coordination** for mixed content programs

This professional architecture enables digital signage systems to handle any content type while maintaining precise synchronization across multiple displays, rivaling expensive commercial digital signage solutions.