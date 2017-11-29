## Web Bluetooth example

First thing, before starting to write our app, we need to integrate Blueapp Web Bluetooth library. If we want to write web app, we can use blueapp.io library. There are three ways we can pick it up:

* Add source to index.html:
```
<script src="https://blueappio.github.io/blueapp.io/blueapp.io.min.js"></script>
```

* Install via bower:
```
bower install blueapp.io --save
```

* Install via npm:
```
npm install blueapp.io --save
```

On the other side, if we want to write node app, we can use blueapp-wb library. We can find it on npm.

```
npm install blueapp-wb
```

### Application code

Getting required package we can start writing our app. Featured code is common regardless which app we want to write (web or node).

#### Requesting device
The main part of the application starts with navigator.bluetooth.requestDevice() promise function. According to Web Bluetooth we are using this function to search for devices that matches option parameters passed. In this case we passed manufacturerData that we want to match.

```javascript
var options = {
    filters: [{manufacturerData: {0x1019:{}}}],
    optionalServices: [CURRENT_SERVICE_UUID],
    acceptAllDevices: false
};
```

We can also pass the name, namePrefix, serviceData or services in filters object. Because we are listening for data advertisement, all this data that we want to match must be advertised by the device without connecting to it. We can also set acceptAllDevices to true, if we want to get first device that we get advertisement from.

#### Connect

After getting device as a response from requestDevice(), most common thing we want to do is to connect to it. We can do that by calling promise function device.gatt.connect(). When connected to a device we are able to check for services. We can use only services that we passed in filters and optionalServices. Otherwise it will return typeError.

So, after getting server as a response from connect() function, we are calling server.getPrimaryService() with service uuid as a parameter. In return we get service.

From this point we can check for service characteristic with getCharacteristic() promise function. Again we are passing characteristic uuid as a parameter. Now that we have characteristic, we can either read value, write value, start notifications and listen for event or get characteristic descriptor. For read/write we use readValue() and writeValue() promise functions (byteArray response/argument). For notifications we use startNotifications() promise function, and then we should listen for 'characteristicvaluechanged' event to get readouts from device on every value change. Response is in byteArray format, so we need to parse it.

For checking for characteristic descriptor we are using getDescriptor() promise function with passed descriptor uuid. Getting that we can read and write value on it using readValue() and writeValue() promise functions. Response or argument should be in byteArray format.

```javascript
...
navigator.bluetooth.requestDevice(options)
        .then(function(device) {
            console.log('> Found ' + device.name);
            console.log('Connecting to GATT Server...');
            wbdevice = device;
            // Connecting on device
            return wbdevice.gatt.connect()
                .then(function (server) {
                    // Getting primary service from device with passed uuid
                    return server.getPrimaryService(CURRENT_SERVICE_UUID)
                        .then(function (service) {
                            // Getting characteristic from service with passed uuid
                            return service.getCharacteristic(CURRENT_UUID)
                                .then(function (characteristic) {
                                    // Storing in global variable
                                    deviceChar = characteristic;
                                    // Starting notifications on characteristic
                                    return characteristic.startNotifications()
                                        .then(function () {
                                            // Listening for event
                                            characteristic.addEventListener('characteristicvaluechanged', function (event) {
                                                var readoutBuffer = event.target.value.buffer;
                                                // Parsing readout data
                                                var readoutValue = parseCharacteristicValue(readoutBuffer);
                                                console.log(readoutValue);
                                            });
                                        });
                                })
                        })
                 })
            // Error handling
            }).catch(onError);
...
```

#### Watch for advertisement

Second thing we can do is just to listen for advertisement from device without connecting to it. As a return from requestDevice(), if gateway finds device that is matching with passed options we get device object. Getting that, we can connect to that device (if that is available) with device.gatt.connect() function, or we can continuously listen for advertisement from that particular device with device.watchAdvertisements() function. Following that, we have to subscribe to "advertisementreceived" event, and as a return, every time device emits new advertisement we are catching it.

From the device as a return we get manufacturerData, and it's in ByteArray format. So we need to add some parsing function to get some meaningful data. In this case we have our helper function for that (getDataFromMfr()).

In case we want to stop watching for advertisement we can call unwatchAdvertisements() function.

```javascript
    navigator.bluetooth.requestDevice(options)
        .then(function (device) {
            console.log('> Found ' + device.name + ' matched to', device);
            console.log('Connecting to GATT Server...');
            wbdevice = device;
            // Starting to watch for advertisement
            wbdevice.watchAdvertisements();
            // Listening for event from requested device
            wbdevice.addEventListener('advertisementreceived', function (event) {
                // Getting data from manufacturerData map
                var data = event.manufacturerData.get(0x1019);
                // Converting byte array to hex string
                var result = arrayBufferToHexString(data);
                // Parsing hex string into meaningful data (using sensor's instructions)
                getDataFromMfr(result);
            });
        }).catch(onError);
```



#### Other supported features

Besides requesting for device, Web Bluetooth gives us ability to continuously scan for nearby devices, with passed parameters by calling requestLEScan() promise function. This feature is not yet supported in official Google Web Bluetooth, and in Blueapp.io it is supported based on official Web Bluetooth specs. Therefore, it is subject to change.

Again we have to prepare options object with three possible parameters: filters, acceptAllAdvertisement and keepRepeatedDevices. Like in requestDevice, filter can contain name, namePrefix, services, manufacturerData and serviceData. Setting acceptAllDevices to true we should get advertisement from all devices. If we set keepRepeatedDevices to false we should get advertisement from same device only once.

After calling requestLEScan() promise function, if it finds any device that matches with passed filter, we can attach eventListener to navigator.bluetooth object in order to get advertised device data. We can parse it to get some meaningful data.

```javascript
navigator.bluetooth.requestLEScan({
  filters: [{manufacturerData: {0x1019: {}}}],
  options: {
    keepRepeatedDevices: true,
  }
}).then(function() {
  navigator.bluetooth.addEventListener('advertisementreceived', function(event) {
    var data = event.manufacturerData.get(0x1019);
    var result = arrayBufferToHexString(data);
    getDataFromMfr(result);
  });
})
```

