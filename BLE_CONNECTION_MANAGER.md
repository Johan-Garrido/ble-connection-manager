# BLE Connection Manager

A comprehensive Android library for managing Bluetooth Low Energy (BLE) connections with enhanced callbacks and robust error handling.

## Overview

The BLE Connection Manager is a Kotlin library for Android that simplifies BLE device communication. It provides a high-level interface for connecting to BLE devices, performing operations like reading/writing characteristics and descriptors, managing notifications, and handling connection states.

This library is based on the original work by Punch Through Design LLC and has been enhanced by Pretty Smart Labs with additional features and improved error handling.

## Features

- **Simple Connection Management**: Easy-to-use API for connecting and disconnecting BLE devices
- **Comprehensive Callbacks**: Detailed event listeners for all connection states and operations
- **Operation Queue**: Thread-safe queuing system for BLE operations
- **Error Handling**: Advanced error analysis and recovery mechanisms
- **RSSI Reading**: Support for reading remote device signal strength
- **MTU Management**: Automatic and manual MTU size configuration
- **Notification Support**: Enable/disable notifications and indications
- **Bond State Monitoring**: Track device pairing status changes
- **Android Version Compatibility**: Support for Android 29+ with compatibility layers

## Installation

### Gradle (via JitPack)

Add JitPack repository to your project's `build.gradle`:

```gradle
allprojects {
    repositories {
        ...
        maven { url 'https://jitpack.io' }
    }
}
```

Add the dependency to your app's `build.gradle`:

```gradle
dependencies {
    implementation 'com.github.JohanGarridoPSL:ble-connection-manager:1.2.0'
}
```

## Quick Start

### 1. Initialize the Connection Manager

```kotlin
import com.prettysmartlabs.ble_connection_manager.connection_manager.ConnectionManager
import com.prettysmartlabs.ble_connection_manager.connection_manager.ConnectionEventListener

// Register for bond state changes (optional)
ConnectionManager.listenToBondStateChanges(applicationContext)
```

### 2. Create a Connection Event Listener

```kotlin
val connectionEventListener = ConnectionEventListener().apply {
    onConnectionSetupComplete = { gatt ->
        Log.d("BLE", "Connected to ${gatt.device.address}")
        // Connection is ready - you can now perform operations
    }
    
    onDisconnect = { device ->
        Log.d("BLE", "Disconnected from ${device.address}")
    }
    
    onCharacteristicRead = { device, characteristic, value ->
        Log.d("BLE", "Read ${characteristic.uuid}: ${value.toHexString()}")
    }
    
    onCharacteristicWrite = { device, characteristic ->
        Log.d("BLE", "Write to ${characteristic.uuid} completed")
    }
    
    onCharacteristicChanged = { device, characteristic, value ->
        Log.d("BLE", "Notification from ${characteristic.uuid}: ${value.toHexString()}")
    }
}
```

### 3. Register the Listener and Connect

```kotlin
// Register the listener
ConnectionManager.registerListener(connectionEventListener)

// Connect to a device
ConnectionManager.connect(bluetoothDevice, applicationContext)
```

### 4. Perform Operations

```kotlin
// Read a characteristic
ConnectionManager.readCharacteristic(device, characteristic)

// Write to a characteristic
val data = byteArrayOf(0x01, 0x02, 0x03)
ConnectionManager.writeCharacteristic(device, characteristic, data)

// Enable notifications
ConnectionManager.enableNotifications(device, characteristic)

// Read RSSI
ConnectionManager.readRemoteRssi(device)

// Request MTU change
ConnectionManager.requestMtu(device, 512)
```

## API Reference

### ConnectionManager

The main singleton class for managing BLE connections.

#### Connection Methods

```kotlin
// Connect to a device
fun connect(device: BluetoothDevice, context: Context)

// Disconnect from a device
fun teardownConnection(reason: Int = GATT_DISCONNECT_NO_ERROR, device: BluetoothDevice)
```

#### Characteristic Operations

```kotlin
// Read characteristic value
fun readCharacteristic(device: BluetoothDevice, characteristic: BluetoothGattCharacteristic)

// Write to characteristic
fun writeCharacteristic(device: BluetoothDevice, characteristic: BluetoothGattCharacteristic, payload: ByteArray)

// Enable notifications/indications
fun enableNotifications(device: BluetoothDevice, characteristic: BluetoothGattCharacteristic)

// Disable notifications/indications
fun disableNotifications(device: BluetoothDevice, characteristic: BluetoothGattCharacteristic)
```

#### Descriptor Operations

```kotlin
// Read descriptor value
fun readDescriptor(device: BluetoothDevice, descriptor: BluetoothGattDescriptor)

// Write to descriptor
fun writeDescriptor(device: BluetoothDevice, descriptor: BluetoothGattDescriptor, payload: ByteArray)
```

#### Utility Methods

```kotlin
// Get services on connected device
fun getServicesOnDevice(device: BluetoothDevice): List<BluetoothGattService>?

// Find characteristic by UUID
fun findCharacteristicOnServiceByUuid(service: BluetoothGattService, uuid: UUID): BluetoothGattCharacteristic?

// Read remote RSSI
fun readRemoteRssi(device: BluetoothDevice)

// Request MTU change
fun requestMtu(device: BluetoothDevice, mtu: Int)
```

#### Listener Management

```kotlin
// Register event listener
fun registerListener(listener: ConnectionEventListener)

// Unregister event listener
fun unregisterListener(listener: ConnectionEventListener)
```

### ConnectionEventListener

Event listener class with callback properties for various BLE events.

#### Connection Events

```kotlin
var onConnectionSetupComplete: ((gatt: BluetoothGatt) -> Unit)? = null
var onDisconnect: ((device: BluetoothDevice) -> Unit)? = null
var onConnectToDeviceEnqueued: ((device: BluetoothDevice) -> Unit)? = null
var onConnectingToGatt: ((device: BluetoothDevice) -> Unit)? = null
var onGattConnected: ((device: BluetoothDevice) -> Unit)? = null
var onGattDisconnected: ((device: BluetoothDevice) -> Unit)? = null
var alreadyConnected: ((BluetoothDevice) -> Unit)? = null
```

#### Operation Events

```kotlin
var onCharacteristicRead: ((device: BluetoothDevice, characteristic: BluetoothGattCharacteristic, value: ByteArray) -> Unit)? = null
var onCharacteristicWrite: ((device: BluetoothDevice, characteristic: BluetoothGattCharacteristic) -> Unit)? = null
var onCharacteristicChanged: ((device: BluetoothDevice, characteristic: BluetoothGattCharacteristic, value: ByteArray) -> Unit)? = null
var onDescriptorRead: ((device: BluetoothDevice, descriptor: BluetoothGattDescriptor, value: ByteArray) -> Unit)? = null
var onDescriptorWrite: ((device: BluetoothDevice, descriptor: BluetoothGattDescriptor) -> Unit)? = null
var onNotificationsEnabled: ((device: BluetoothDevice, characteristic: BluetoothGattCharacteristic) -> Unit)? = null
var onNotificationsDisabled: ((device: BluetoothDevice, characteristic: BluetoothGattCharacteristic) -> Unit)? = null
```

#### Other Events

```kotlin
var onMtuChanged: ((device: BluetoothDevice, newMtu: Int) -> Unit)? = null
var onReadRemoteRssi: ((device: BluetoothDevice, rssi: Int) -> Unit)? = null
var onBondStateChanged: ((device: BluetoothDevice?, previousBondState: Int, bondState: Int) -> Unit)? = null
```

#### Error Events

```kotlin
var onDisconnectDueToFailure: ((device: BluetoothDevice, error: DisconnectionError) -> Unit)? = null
var onFailedToDisconnect: ((device: BluetoothDevice, error: DisconnectionError) -> Unit)? = null
```

### Error Handling

The library includes comprehensive error handling through the `DisconnectionError` sealed class:

```kotlin
sealed class DisconnectionError {
    data class DfuUploadFailure(val status: Int, val device: BluetoothDevice) : DisconnectionError()
    data class ExpectedDisconnect(val status: Int, val device: BluetoothDevice) : DisconnectionError()
    data class GeneralError(val status: Int, val device: BluetoothDevice, val details: String?) : DisconnectionError()
    data class DeviceNotConnected(val status: Int, val device: BluetoothDevice) : DisconnectionError()
}
```

## Extension Functions

The library provides useful extension functions for BLE operations:

### BluetoothGatt Extensions

```kotlin
// Print GATT table for debugging
fun BluetoothGatt.printGattTable()

// Find characteristic by UUID
fun BluetoothGatt.findCharacteristic(characteristicUuid: UUID, serviceUuid: UUID? = null): BluetoothGattCharacteristic?

// Find descriptor by UUID
fun BluetoothGatt.findDescriptor(descriptorUuid: UUID, characteristicUuid: UUID? = null, serviceUuid: UUID? = null): BluetoothGattDescriptor?
```

### BluetoothGattCharacteristic Extensions

```kotlin
// Check properties
fun BluetoothGattCharacteristic.isReadable(): Boolean
fun BluetoothGattCharacteristic.isWritable(): Boolean
fun BluetoothGattCharacteristic.isWritableWithoutResponse(): Boolean
fun BluetoothGattCharacteristic.isNotifiable(): Boolean
fun BluetoothGattCharacteristic.isIndicatable(): Boolean

// Execute write operation
fun BluetoothGattCharacteristic.executeWrite(gatt: BluetoothGatt, payload: ByteArray, writeType: Int)
```

### BluetoothGattDescriptor Extensions

```kotlin
// Check permissions
fun BluetoothGattDescriptor.isReadable(): Boolean
fun BluetoothGattDescriptor.isWritable(): Boolean

// Check if it's a CCCD
fun BluetoothGattDescriptor.isCccd(): Boolean

// Execute write operation
fun BluetoothGattDescriptor.executeWrite(gatt: BluetoothGatt, payload: ByteArray)
```

### ByteArray Extensions

```kotlin
// Convert to hex string
fun ByteArray.toHexString(): String
```

## Complete Example

Here's a complete example of using the BLE Connection Manager:

```kotlin
class MainActivity : AppCompatActivity() {
    private lateinit var connectionEventListener: ConnectionEventListener
    private var targetDevice: BluetoothDevice? = null
    
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)
        
        setupBleConnectionManager()
    }
    
    private fun setupBleConnectionManager() {
        // Initialize bond state listener
        ConnectionManager.listenToBondStateChanges(this)
        
        // Create connection event listener
        connectionEventListener = ConnectionEventListener().apply {
            onConnectionSetupComplete = { gatt ->
                Log.d("BLE", "Connected to ${gatt.device.address}")
                runOnUiThread {
                    // Update UI - connection established
                }
            }
            
            onDisconnect = { device ->
                Log.d("BLE", "Disconnected from ${device.address}")
                runOnUiThread {
                    // Update UI - disconnected
                }
            }
            
            onCharacteristicRead = { device, characteristic, value ->
                Log.d("BLE", "Read from ${characteristic.uuid}: ${value.toHexString()}")
                // Process read data
            }
            
            onCharacteristicWrite = { device, characteristic ->
                Log.d("BLE", "Write to ${characteristic.uuid} completed")
            }
            
            onCharacteristicChanged = { device, characteristic, value ->
                Log.d("BLE", "Notification from ${characteristic.uuid}: ${value.toHexString()}")
                // Process notification data
            }
            
            onNotificationsEnabled = { device, characteristic ->
                Log.d("BLE", "Notifications enabled for ${characteristic.uuid}")
            }
            
            onReadRemoteRssi = { device, rssi ->
                Log.d("BLE", "RSSI: $rssi dBm")
            }
            
            onMtuChanged = { device, mtu ->
                Log.d("BLE", "MTU changed to $mtu")
            }
            
            onDisconnectDueToFailure = { device, error ->
                Log.e("BLE", "Connection failed: $error")
                when (error) {
                    is DisconnectionError.DfuUploadFailure -> {
                        // Handle DFU failure
                    }
                    is DisconnectionError.GeneralError -> {
                        // Handle general connection error
                    }
                    is DisconnectionError.DeviceNotConnected -> {
                        // Handle device not connected error
                    }
                    is DisconnectionError.ExpectedDisconnect -> {
                        // Handle expected disconnection
                    }
                }
            }
            
            onBondStateChanged = { device, previousState, newState ->
                Log.d("BLE", "Bond state changed from $previousState to $newState")
            }
        }
        
        // Register the listener
        ConnectionManager.registerListener(connectionEventListener)
    }
    
    private fun connectToDevice(device: BluetoothDevice) {
        targetDevice = device
        ConnectionManager.connect(device, this)
    }
    
    private fun disconnectFromDevice() {
        targetDevice?.let { device ->
            ConnectionManager.teardownConnection(device = device)
        }
    }
    
    private fun readCharacteristic(serviceUuid: UUID, characteristicUuid: UUID) {
        targetDevice?.let { device ->
            ConnectionManager.getServicesOnDevice(device)?.let { services ->
                services.find { it.uuid == serviceUuid }?.let { service ->
                    ConnectionManager.findCharacteristicOnServiceByUuid(service, characteristicUuid)?.let { characteristic ->
                        ConnectionManager.readCharacteristic(device, characteristic)
                    }
                }
            }
        }
    }
    
    private fun writeCharacteristic(serviceUuid: UUID, characteristicUuid: UUID, data: ByteArray) {
        targetDevice?.let { device ->
            ConnectionManager.getServicesOnDevice(device)?.let { services ->
                services.find { it.uuid == serviceUuid }?.let { service ->
                    ConnectionManager.findCharacteristicOnServiceByUuid(service, characteristicUuid)?.let { characteristic ->
                        ConnectionManager.writeCharacteristic(device, characteristic, data)
                    }
                }
            }
        }
    }
    
    private fun enableNotifications(serviceUuid: UUID, characteristicUuid: UUID) {
        targetDevice?.let { device ->
            ConnectionManager.getServicesOnDevice(device)?.let { services ->
                services.find { it.uuid == serviceUuid }?.let { service ->
                    ConnectionManager.findCharacteristicOnServiceByUuid(service, characteristicUuid)?.let { characteristic ->
                        ConnectionManager.enableNotifications(device, characteristic)
                    }
                }
            }
        }
    }
    
    override fun onDestroy() {
        super.onDestroy()
        ConnectionManager.unregisterListener(connectionEventListener)
    }
}
```

## Requirements

- **Android SDK**: Minimum API level 29 (Android 10)
- **Compile SDK**: API level 35
- **Kotlin**: Version 1.9+
- **Java**: Version 17

## Permissions

Add the following permissions to your `AndroidManifest.xml`:

```xml
<uses-permission android:name="android.permission.BLUETOOTH" />
<uses-permission android:name="android.permission.BLUETOOTH_ADMIN" />
<uses-permission android:name="android.permission.ACCESS_FINE_LOCATION" />

<!-- For Android 12+ -->
<uses-permission android:name="android.permission.BLUETOOTH_CONNECT" />
<uses-permission android:name="android.permission.BLUETOOTH_SCAN" />
```

## License

This library is licensed under the Apache License, Version 2.0. It incorporates code originally developed by Punch Through Design LLC, with modifications by Pretty Smart Labs.

## Contributing

This library is maintained by Pretty Smart Labs. For issues, feature requests, or contributions, please contact the development team.

## Changelog

### Version 1.1.1
- Added remote RSSI reading capability
- Enhanced connection state callbacks
- Improved error handling with detailed error types
- Added bond state monitoring
- Enhanced documentation and code comments
- Added utility functions for characteristic and descriptor operations
- Improved Android version compatibility