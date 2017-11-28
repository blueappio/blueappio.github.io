# BluaApp Web Bluetooth

## Basic Overview
Blueapp-wb is fast, easy to use and feature-rich Web Bluetooth library. It is based on Web Bluetooth spec, and enables developers to create node apps for discovering, connecting, and listening for advertisement of bluetooth devices using Blueapp platform.

## Getting Started

There are few prerequisites for using and testing Blueapp-wb. Since it's based on BlueApp platform, we need to get some information about gateway that we want to use for communication between our app and particular device.

For that we want to use [BlueApp](http://blueapp.io) portal for managing gateways and devices.
For using and testing this app there are few prerequisites that you must fulfill.

### Prerequisites

For communication between our application and BLE devices via gateway we are using node BlueApp Web Bluetooth library based on GATT-IP protocol. Blueapp-wb is based on official Web Bluetooth specs.

### Installing

Download blueapp-wb npm package by running:

```
npm install blueapp-wb
```

or

```
npm install
```

because it's already in package.json.

### Blueapp account setup

In order to get our app working, we need to provide it with some information about gateway. In order to get some information first we need to open account on BlueApp.io webpage:

<img src="https://github.com/rajicdalibor/webBluetoothTest/blob/master/images/mainpage.png" width="80%" alt="Main page"/>

<!-- ![](https://github.com/rajicdalibor/webBluetoothTest/blob/master/images/mainpage.JPG | width=100 "Blueapp main page") -->

After getting new account, we are able to open just our new organization, but at this point we are unable to see any of the gateways assigned to that organization. For testing purposes we can switch to some existing organization with already attached gateway with nearby bluetooth devices.
In order to join particular organization, we need owner's invitation. To get that please send email with invitation request to Blueapp team and you will get one in short term. From organizations where we are marked as admin, we can invite other users to join by sending them invitation.

Now, with gateway available, we can test some of the applications listed in main application menu. Eventually we can add our new application to our organization and use it with our gateways.

For development and testing our new app on local machine, we need gateway's token, which tells application to which gateway it should connect for scanning for BLE devices.
Selecting My Devices tab you can check all the gateways that are connected to particular organization. We can click on particular organization to open gateway's details. On gateway's details page we can find Client Token that we need for our app.

<img src="https://github.com/rajicdalibor/webBluetoothTest/blob/master/images/gatewaydetails.png" width="80%" alt="Gateway details"/>

## Starting the app

Now we have our app setup and our gateway's token, and we can start the app and test it.
Since we are building node application, we are starting it by with command node ourApp.js. There are two ways we can pass gateway's token that we get from Blueapp. One is storing gateway's token into environment variable as "token", or passing token argument when starting the app (node ourApp.js gatewaytoken).


### Requesting device
The main part of the application starts with navigator.bluetooth.requestDevice() promise function. According to Web Bluetooth we are using this function to search for devices that matches option parameters passed. In this case we passed manufacturerData that we want to match.

```javascript
var options = {
    filters: [{manufacturerData: {0x1019:{}}}],
    optionalServices: [CURRENT_SERVICE_UUID],
    acceptAllDevices: false
};
```

We can also pass the name, namePrefix, serviceData or services in filters object. Because we are listening for data advertisement, all this data that we want to match must be advertised by the device without connecting to it. We can also set acceptAllDevices to true, if we want to get first device that we get advertisement from.

### Connect

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

### Watch for advertisement

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



### Supported features

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



### Adding application to Blueapp

When we get our application ready we can add it on Blueapp portal in our organization.

Let's open our organization in organizations tab. There we can see all the organization that we are subscribed to, and we can list organization's apps.

<img src="https://github.com/rajicdalibor/webBluetoothTest/blob/master/images/organization.png" width="80%" alt="Organization"/>

Before we can add app to our organization, we have to post our app on some domain service, and set application's url to applications page. (You can upload it on your github account)

<img src="https://github.com/rajicdalibor/webBluetoothTest/blob/master/images/applicationsetup.png" width="80%" alt="Application setup"/>

It's also required to add some device filter (uuid or name).

Now we can see our application listed on main page and use from there.

## More information

For more GATT protocol information check [GATT](https://www.bluetooth.com/specifications/gatt/generic-attributes-overview).

If you need more information about Web Bluetooth visit official [page](https://webbluetoothcg.github.io/web-bluetooth).



