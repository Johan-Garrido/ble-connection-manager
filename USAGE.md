# Typical Usage Flow - BLE Connection Manager

This document describes a typical usage flow for the BLE Connection Manager library, demonstrating how to implement a complete BLE communication workflow from device discovery to data exchange.

## Overview

A typical BLE workflow with this library follows these stages:
1. **Setup** - Initialize the connection manager and register callbacks
2. **Connect** - Establish connection to a BLE device
3. **Discover Services** - Automatically discover available services and characteristics
4. **Perform Operations** - Read/Write characteristics, enable notifications
5. **Handle Data** - Process incoming data from notifications or reads
6. **Disconnect** - Properly tear down the connection

## Installation

Before using the library, you need to add it to your project dependencies.

### Add JitPack Repository

Add the JitPack repository to your project's `settings.gradle.kts` (or `settings.gradle`):

```kotlin
dependencyResolutionManagement {
    repositoriesMode.set(RepositoriesMode.FAIL_ON_PROJECT_REPOS)
    repositories {
        google()
        mavenCentral()
        maven { url = uri("https://jitpack.io") }
    }
}
```

Or if using older Gradle versions, add to your root `build.gradle`:

```gradle
allprojects {
    repositories {
        google()
        mavenCentral()
        maven { url 'https://jitpack.io' }
    }
}
```

### Add Library Dependency

Add the BLE Connection Manager dependency to your app module's `build.gradle.kts` (or `build.gradle`):

```kotlin
dependencies {
    implementation("com.github.JohanGarridoPSL:ble-connection-manager:1.1.1")
}
```

Or using Groovy syntax:

```gradle
dependencies {
    implementation 'com.github.JohanGarridoPSL:ble-connection-manager:1.1.1'
}
```

Sync your project with Gradle files after adding these dependencies.

## Typical Flow Example

### Step 1: Setup and Initialization

First, set up your BLE permissions in `AndroidManifest.xml`:

```xml
<uses-permission android:name="android.permission.BLUETOOTH" />
<uses-permission android:name="android.permission.BLUETOOTH_ADMIN" />
<uses-permission android:name="android.permission.ACCESS_FINE_LOCATION" />

<!-- For Android 12+ -->
<uses-permission android:name="android.permission.BLUETOOTH_CONNECT" />
<uses-permission android:name="android.permission.BLUETOOTH_SCAN" />
```

Initialize the ConnectionManager (typically in your Activity or ViewModel):

```kotlin
import com.prettysmartlabs.ble_connection_manager.connection_manager.ConnectionManager
import com.prettysmartlabs.ble_connection_manager.connection_manager.ConnectionEventListener

class BleDeviceManager(private val context: Context) {
    
    private var connectedDevice: BluetoothDevice? = null
    private lateinit var connectionEventListener: ConnectionEventListener
    
    init {
        // Optional: Listen to bond state changes for devices that require pairing
        ConnectionManager.listenToBondStateChanges(context)
        
        setupConnectionListener()
        ConnectionManager.registerListener(connectionEventListener)
    }
}
```

### Step 2: Create Connection Event Listener

Define callbacks to handle the connection lifecycle and data exchange:

```kotlin
private fun setupConnectionListener() {
    connectionEventListener = ConnectionEventListener().apply {
        
        // Connection lifecycle callbacks
        onConnectToDeviceEnqueued = { device ->
            Log.d(TAG, "Connection request enqueued for ${device.address}")
            // Update UI: Show "Connecting..." state
        }
        
        onConnectingToGatt = { device ->
            Log.d(TAG, "Connecting to GATT server of ${device.address}")
        }
        
        onGattConnected = { device ->
            Log.d(TAG, "GATT connected, discovering services...")
        }
        
        onGattDiscoveringServices = { device ->
            Log.d(TAG, "Discovering services on ${device.address}")
        }
        
        onConnectionSetupComplete = { gatt ->
            Log.d(TAG, "Connection setup complete for ${gatt.device.address}")
            connectedDevice = gatt.device
            
            // Connection is ready - start your application flow
            onDeviceReady(gatt)
        }
        
        onDisconnect = { device ->
            Log.d(TAG, "Disconnected from ${device.address}")
            connectedDevice = null
            // Update UI: Show disconnected state
        }
        
        alreadyConnected = { device ->
            Log.w(TAG, "Device ${device.address} is already connected")
        }
        
        // Error handling
        onDisconnectDueToFailure = { device, error ->
            Log.e(TAG, "Connection failed: $error")
            handleConnectionError(device, error)
        }
        
        onFailedToDisconnect = { device, error ->
            Log.e(TAG, "Failed to disconnect: $error")
        }
    }
}
```

### Step 3: Connect to Device

After scanning and finding your target device (using standard Android BLE scanning APIs):

```kotlin
fun connectToDevice(device: BluetoothDevice) {
    Log.d(TAG, "Initiating connection to ${device.address}")
    ConnectionManager.connect(device, context)
}
```

### Step 4: Device Ready - Discover and Access Services

Once connected, discover services and prepare for operations:

```kotlin
private fun onDeviceReady(gatt: BluetoothGatt) {
    // Print the GATT table for debugging (optional)
    gatt.printGattTable()
    
    // Get available services
    val services = ConnectionManager.getServicesOnDevice(gatt.device)
    services?.forEach { service ->
        Log.d(TAG, "Service found: ${service.uuid}")
        service.characteristics.forEach { characteristic ->
            Log.d(TAG, "  Characteristic: ${characteristic.uuid}")
        }
    }
    
    // Start your application-specific flow
    startDataExchange(gatt.device)
}

private fun startDataExchange(device: BluetoothDevice) {
    // Example: Request larger MTU for better throughput
    ConnectionManager.requestMtu(device, 512)
    
    // Example: Enable notifications on a specific characteristic
    enableNotificationsForData(device)
    
    // Example: Read initial device state
    readDeviceState(device)
}
```

### Step 5: Enable Notifications

Set up notification handlers and enable notifications on characteristics:

```kotlin
private fun setupNotificationHandlers() {
    connectionEventListener.apply {
        onNotificationsEnabled = { device, characteristic ->
            Log.d(TAG, "Notifications enabled for ${characteristic.uuid}")
            // Notifications are ready - device will send data updates
        }
        
        onCharacteristicChanged = { device, characteristic, value ->
            Log.d(TAG, "Notification received from ${characteristic.uuid}: ${value.toHexString()}")
            // Process incoming data
            handleIncomingData(characteristic.uuid, value)
        }
        
        onNotificationsDisabled = { device, characteristic ->
            Log.d(TAG, "Notifications disabled for ${characteristic.uuid}")
        }
    }
}

private fun enableNotificationsForData(device: BluetoothDevice) {
    val serviceUuid = UUID.fromString("0000180d-0000-1000-8000-00805f9b34fb") // Example: Heart Rate Service
    val characteristicUuid = UUID.fromString("00002a37-0000-1000-8000-00805f9b34fb") // Example: Heart Rate Measurement
    
    ConnectionManager.getServicesOnDevice(device)?.let { services ->
        services.find { it.uuid == serviceUuid }?.let { service ->
            ConnectionManager.findCharacteristicOnServiceByUuid(service, characteristicUuid)?.let { characteristic ->
                ConnectionManager.enableNotifications(device, characteristic)
            }
        }
    }
}
```

### Step 6: Read Characteristics

Read data from the device:

```kotlin
private fun setupReadHandlers() {
    connectionEventListener.onCharacteristicRead = { device, characteristic, value ->
        Log.d(TAG, "Read from ${characteristic.uuid}: ${value.toHexString()}")
        // Process read data
        processReadData(characteristic.uuid, value)
    }
}

private fun readDeviceState(device: BluetoothDevice) {
    val serviceUuid = UUID.fromString("0000180f-0000-1000-8000-00805f9b34fb") // Example: Battery Service
    val characteristicUuid = UUID.fromString("00002a19-0000-1000-8000-00805f9b34fb") // Example: Battery Level
    
    ConnectionManager.getServicesOnDevice(device)?.let { services ->
        services.find { it.uuid == serviceUuid }?.let { service ->
            ConnectionManager.findCharacteristicOnServiceByUuid(service, characteristicUuid)?.let { characteristic ->
                ConnectionManager.readCharacteristic(device, characteristic)
            }
        }
    }
}
```

### Step 7: Write to Characteristics

Send commands or data to the device:

```kotlin
private fun setupWriteHandlers() {
    connectionEventListener.onCharacteristicWrite = { device, characteristic ->
        Log.d(TAG, "Write completed for ${characteristic.uuid}")
        // Write operation confirmed
    }
}

private fun sendCommand(device: BluetoothDevice, command: ByteArray) {
    val serviceUuid = UUID.fromString("your-service-uuid")
    val characteristicUuid = UUID.fromString("your-characteristic-uuid")
    
    ConnectionManager.getServicesOnDevice(device)?.let { services ->
        services.find { it.uuid == serviceUuid }?.let { service ->
            ConnectionManager.findCharacteristicOnServiceByUuid(service, characteristicUuid)?.let { characteristic ->
                ConnectionManager.writeCharacteristic(device, characteristic, command)
            }
        }
    }
}

// Example: Send a command to the device
fun turnOnDevice() {
    connectedDevice?.let { device ->
        val command = byteArrayOf(0x01) // Example command
        sendCommand(device, command)
    }
}
```

### Step 8: Monitor Connection Quality

Read RSSI and handle MTU changes:

```kotlin
private fun setupConnectionMonitoring() {
    connectionEventListener.apply {
        onReadRemoteRssi = { device, rssi ->
            Log.d(TAG, "RSSI: $rssi dBm")
            // Update UI with signal strength
            updateSignalStrength(rssi)
        }
        
        onMtuChanged = { device, newMtu ->
            Log.d(TAG, "MTU changed to $newMtu bytes")
            // Adjust data packet size if needed
        }
        
        onBondStateChanged = { device, previousState, newState ->
            Log.d(TAG, "Bond state changed from $previousState to $newState")
            // Handle pairing state changes
        }
    }
}

// Periodically check signal strength
private fun startRssiMonitoring(device: BluetoothDevice) {
    // Read RSSI every 5 seconds
    handler.postDelayed(object : Runnable {
        override fun run() {
            ConnectionManager.readRemoteRssi(device)
            handler.postDelayed(this, 5000)
        }
    }, 5000)
}
```

### Step 9: Handle Incoming Data

Process data received from the device:

```kotlin
private fun handleIncomingData(characteristicUuid: UUID, data: ByteArray) {
    when (characteristicUuid.toString()) {
        "00002a37-0000-1000-8000-00805f9b34fb" -> {
            // Heart Rate Measurement
            val heartRate = parseHeartRate(data)
            Log.d(TAG, "Heart Rate: $heartRate bpm")
            // Update UI
        }
        "your-custom-characteristic-uuid" -> {
            // Custom data
            val value = parseCustomData(data)
            // Process custom data
        }
    }
}

private fun processReadData(characteristicUuid: UUID, data: ByteArray) {
    when (characteristicUuid.toString()) {
        "00002a19-0000-1000-8000-00805f9b34fb" -> {
            // Battery Level
            val batteryLevel = data[0].toInt()
            Log.d(TAG, "Battery Level: $batteryLevel%")
            // Update UI
        }
    }
}
```

### Step 10: Disconnect

Properly disconnect from the device when done:

```kotlin
fun disconnect() {
    connectedDevice?.let { device ->
        Log.d(TAG, "Disconnecting from ${device.address}")
        
        // Optional: Disable notifications before disconnecting
        disableAllNotifications(device)
        
        // Disconnect
        ConnectionManager.teardownConnection(device = device)
    }
}

private fun disableAllNotifications(device: BluetoothDevice) {
    // Disable notifications on all subscribed characteristics
    val serviceUuid = UUID.fromString("your-service-uuid")
    val characteristicUuid = UUID.fromString("your-characteristic-uuid")
    
    ConnectionManager.getServicesOnDevice(device)?.let { services ->
        services.find { it.uuid == serviceUuid }?.let { service ->
            ConnectionManager.findCharacteristicOnServiceByUuid(service, characteristicUuid)?.let { characteristic ->
                ConnectionManager.disableNotifications(device, characteristic)
            }
        }
    }
}
```

### Step 11: Cleanup

Unregister the listener when your component is destroyed:

```kotlin
fun cleanup() {
    // Disconnect if still connected
    disconnect()
    
    // Unregister listener
    ConnectionManager.unregisterListener(connectionEventListener)
}
```

## Complete Workflow Example

Here's a complete example showing all steps together:

```kotlin
class BleDeviceController(private val context: Context) {
    
    private var connectedDevice: BluetoothDevice? = null
    private lateinit var connectionEventListener: ConnectionEventListener
    private val handler = Handler(Looper.getMainLooper())
    
    companion object {
        private const val TAG = "BleDeviceController"
        
        // Example UUIDs - replace with your device's UUIDs
        val SERVICE_UUID = UUID.fromString("0000180d-0000-1000-8000-00805f9b34fb")
        val DATA_CHARACTERISTIC_UUID = UUID.fromString("00002a37-0000-1000-8000-00805f9b34fb")
        val COMMAND_CHARACTERISTIC_UUID = UUID.fromString("00002a39-0000-1000-8000-00805f9b34fb")
    }
    
    init {
        ConnectionManager.listenToBondStateChanges(context)
        setupConnectionListener()
        ConnectionManager.registerListener(connectionEventListener)
    }
    
    private fun setupConnectionListener() {
        connectionEventListener = ConnectionEventListener().apply {
            // Connection flow
            onConnectToDeviceEnqueued = { device ->
                Log.d(TAG, "⏳ Connection enqueued: ${device.address}")
            }
            
            onConnectionSetupComplete = { gatt ->
                Log.d(TAG, "✅ Connected to ${gatt.device.address}")
                connectedDevice = gatt.device
                onDeviceConnected(gatt)
            }
            
            onDisconnect = { device ->
                Log.d(TAG, "❌ Disconnected from ${device.address}")
                connectedDevice = null
            }
            
            // Data operations
            onCharacteristicRead = { device, characteristic, value ->
                Log.d(TAG, "📖 Read: ${characteristic.uuid} = ${value.toHexString()}")
            }
            
            onCharacteristicWrite = { device, characteristic ->
                Log.d(TAG, "📝 Write complete: ${characteristic.uuid}")
            }
            
            onCharacteristicChanged = { device, characteristic, value ->
                Log.d(TAG, "🔔 Notification: ${characteristic.uuid} = ${value.toHexString()}")
                handleNotification(characteristic.uuid, value)
            }
            
            onNotificationsEnabled = { device, characteristic ->
                Log.d(TAG, "🔔 Notifications enabled: ${characteristic.uuid}")
            }
            
            // Connection quality
            onMtuChanged = { device, mtu ->
                Log.d(TAG, "📊 MTU: $mtu bytes")
            }
            
            onReadRemoteRssi = { device, rssi ->
                Log.d(TAG, "📶 RSSI: $rssi dBm")
            }
            
            // Error handling
            onDisconnectDueToFailure = { device, error ->
                Log.e(TAG, "⚠️ Connection failed: $error")
                handleError(error)
            }
        }
    }
    
    // Connect to device
    fun connect(device: BluetoothDevice) {
        ConnectionManager.connect(device, context)
    }
    
    // Called when device is ready
    private fun onDeviceConnected(gatt: BluetoothGatt) {
        gatt.printGattTable() // Debug: print services
        
        val device = gatt.device
        
        // 1. Request larger MTU
        ConnectionManager.requestMtu(device, 512)
        
        // 2. Enable notifications
        enableNotifications(device)
        
        // 3. Read initial state
        readInitialData(device)
        
        // 4. Start monitoring
        startRssiMonitoring(device)
    }
    
    private fun enableNotifications(device: BluetoothDevice) {
        findCharacteristic(device, SERVICE_UUID, DATA_CHARACTERISTIC_UUID)?.let { char ->
            ConnectionManager.enableNotifications(device, char)
        }
    }
    
    private fun readInitialData(device: BluetoothDevice) {
        findCharacteristic(device, SERVICE_UUID, DATA_CHARACTERISTIC_UUID)?.let { char ->
            ConnectionManager.readCharacteristic(device, char)
        }
    }
    
    // Send command to device
    fun sendCommand(command: ByteArray) {
        connectedDevice?.let { device ->
            findCharacteristic(device, SERVICE_UUID, COMMAND_CHARACTERISTIC_UUID)?.let { char ->
                ConnectionManager.writeCharacteristic(device, char, command)
            }
        }
    }
    
    // Handle incoming notifications
    private fun handleNotification(uuid: UUID, data: ByteArray) {
        when (uuid) {
            DATA_CHARACTERISTIC_UUID -> {
                // Process your data
                val value = parseData(data)
                // Update UI or state
            }
        }
    }
    
    private fun parseData(data: ByteArray): String {
        // Implement your parsing logic
        return data.toHexString()
    }
    
    private fun startRssiMonitoring(device: BluetoothDevice) {
        handler.postDelayed(object : Runnable {
            override fun run() {
                if (connectedDevice != null) {
                    ConnectionManager.readRemoteRssi(device)
                    handler.postDelayed(this, 5000)
                }
            }
        }, 5000)
    }
    
    private fun handleError(error: DisconnectionError) {
        when (error) {
            is DisconnectionError.GeneralError -> {
                Log.e(TAG, "General error: ${error.details}")
            }
            is DisconnectionError.DeviceNotConnected -> {
                Log.e(TAG, "Device not connected")
            }
            is DisconnectionError.DfuUploadFailure -> {
                Log.e(TAG, "DFU upload failed")
            }
            is DisconnectionError.ExpectedDisconnect -> {
                Log.d(TAG, "Expected disconnect")
            }
        }
    }
    
    // Helper to find characteristic
    private fun findCharacteristic(
        device: BluetoothDevice,
        serviceUuid: UUID,
        characteristicUuid: UUID
    ): BluetoothGattCharacteristic? {
        return ConnectionManager.getServicesOnDevice(device)
            ?.find { it.uuid == serviceUuid }
            ?.let { service ->
                ConnectionManager.findCharacteristicOnServiceByUuid(service, characteristicUuid)
            }
    }
    
    // Disconnect from device
    fun disconnect() {
        connectedDevice?.let { device ->
            ConnectionManager.teardownConnection(device = device)
        }
    }
    
    // Clean up resources
    fun cleanup() {
        handler.removeCallbacksAndMessages(null)
        disconnect()
        ConnectionManager.unregisterListener(connectionEventListener)
    }
}
```

## Usage in an Activity

```kotlin
class MainActivity : AppCompatActivity() {
    
    private lateinit var bleController: BleDeviceController
    private var targetDevice: BluetoothDevice? = null
    
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)
        
        bleController = BleDeviceController(this)
        
        // After scanning and finding your device
        // bleController.connect(targetDevice)
    }
    
    private fun onDeviceFound(device: BluetoothDevice) {
        targetDevice = device
        bleController.connect(device)
    }
    
    private fun sendCommandToDevice() {
        val command = byteArrayOf(0x01, 0x02, 0x03)
        bleController.sendCommand(command)
    }
    
    override fun onDestroy() {
        super.onDestroy()
        bleController.cleanup()
    }
}
```

## Key Points

1. **Always register the listener before connecting** to ensure you don't miss any callbacks
2. **Use the operation queue** - the library automatically queues and serializes all BLE operations
3. **Wait for `onConnectionSetupComplete`** before performing any operations
4. **Handle errors gracefully** using the `onDisconnectDueToFailure` callback
5. **Unregister listeners** when done to prevent memory leaks
6. **Use extension functions** like `printGattTable()`, `isReadable()`, `toHexString()` for easier debugging
7. **Request appropriate MTU** for your data throughput needs (default is 23 bytes)
8. **Monitor RSSI** to track connection quality
9. **Always disconnect properly** before your Activity/Fragment is destroyed

## Common Patterns

### Pattern 1: Request-Response
```kotlin
// Send a command and wait for response via notification
fun queryDeviceStatus() {
    val queryCommand = byteArrayOf(0xAA, 0x01)
    sendCommand(queryCommand)
    // Response will arrive via onCharacteristicChanged callback
}
```

### Pattern 2: Periodic Updates
```kotlin
// Enable notifications and receive periodic data
fun startMonitoring() {
    enableNotifications(device)
    // Data will arrive periodically via onCharacteristicChanged
}
```

### Pattern 3: One-time Read
```kotlin
// Read a characteristic once
fun getBatteryLevel() {
    readCharacteristic(device, batteryCharacteristic)
    // Result arrives in onCharacteristicRead callback
}
```

## Troubleshooting

- **Device not connecting**: Check Bluetooth permissions and ensure Bluetooth is enabled
- **Operations failing**: Ensure `onConnectionSetupComplete` has been called before performing operations
- **Missing notifications**: Verify the characteristic supports notifications and that you've enabled them
- **Data corruption**: Check MTU size - increase if sending large data packets
- **Memory leaks**: Always unregister listeners in `onDestroy()`

## Next Steps

For more detailed API documentation, see [BLE_CONNECTION_MANAGER.md](BLE_CONNECTION_MANAGER.md).
