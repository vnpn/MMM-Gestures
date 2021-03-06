# MMM-Gestures 
MMM-Gestures is a head-less 3rd party module for [MagicMirror](https://github.com/MichMich/MagicMirror) that allows to control the Magic Mirror via gestures of two types of infrared sensors. Gestures can be up, down, left, right, far and close movements of the hand in front of a gesture sensor (APDS-9960) and present and away gestures in front of an IR distance sensor (GP2Y0A21YK).

## Functionality
* Control magic mirror modules via gestures, e.g.
** scroll through news via left / right gestures, show news details (description, full news article) via up gesture
** show compliment when person stands in front of mirror via present gesture
** reload mirror via far / close gesture
* Energy saving through turning off the monitor 5 minutes after use, and turning on the monitor if a person stands in front of the mirror

## Hardware Setup
In order to receive gesture events the following hardware is required:
* Distance sensor: GP2Y0A21YK, incl. connector cable
* Gesture sensor: APDS-9960 on breakout board with gesture sensor on one side, but no other electronic component on the same side for easier assembly (can be found on Aliexpress)
* Micro-controller: Arduino (Uno), incl. USB cable
* Jumper cable to connect APDS-9960 with Arduino
* Pin headers to solder to connector cable of GP2Y0A21YK
* Material to hold the sensors, e.g. laser cut sensor module from "Hardware Sensor Module\Mirror Gesture Module.ai"

### Arduino and Sensor Setup
The Arduino collects gesture events from gesture sensor APDS-9960 and distance from distance sensor GP2Y0A21YK (10-80cm), which are then forwarded on the serial port as text.

The circuit for an Arduino Uno:
* Input 1: APDS-9960 on digital pin 2 (interrupt) + I2C (SDA on pin A4, SCL on pin A5) + GND & VCC (3.3V)
* Input 2: GP2Y0A21YK on analog pin 0 (analog) + GND & VCC (5V)
* Output: serial out on USB

### Laser Cut Sensor Module
In order to place the sensors below the mirror you can use "Hardware Sensor Module\Mirror Gesture Module.ai" to cut out wholes of the right size for the APDS-9960 and GP2Y0A21YK sensor. The file contains three elements, in the order they need to be "printed":
* Back side: The first "sheet" of 4mm plywood needs to be processed from both sides. The backside cuts out of the whole module (need to adjust outer rectangle width and height to your mirror) and centered engravings for letting the sensors sink into the wood. The front of the gesture sensor APDS-9960 needs to be leveled with the wooden front of the sensor module. On a Zing 40 with 4mm plywood I used as engraving setting speed: 18% and power: 100% to engrave deep enough into the wood.
* Front side: The front side of the first "sheet" of 4mm plywood has cut-outs for the sensors' fronts and a decorative engraving.
* Frame: In order to hold the glass, panel and backlight, two or three more 4mm plywood sheets should be lasered using the third element from the file. This sheet has generous cut-outs for the sensors and wiring.

After wiring the sensors to the Arduino they can be glued to the first sheet from the back. Once dried, the other two to tree frames can be glued on the back of the first sheet.

### Arduino Software Setup
In order to compile you'll need to copy the following libraries to Arduino's library folder. The libraries are also included in this git repository.
* RunningMedian library, taken from https://github.com/RobTillaart/Arduino/tree/master/libraries/RunningMedian
* DistanceSensor library for GP2Y0A21YK, taken from https://github.com/jeroendoggen/Arduino-distance-sensor-library/tree/master/DistanceSensor
* SparkFun_APDS9960 library for APDS-9960 gesture sensor, taken from https://github.com/sparkfun/APDS-9960_RGB_and_Gesture_Sensor

### Arduino - Raspberry Pi Setup
Once the Arduino is connected to both sensors and the software is setup it will send gesture events to the Raspberry Pi via USB. On the Raspberry Pi a Node.js programm will listen to these events and make them available via a WebSocket server.

## Installation
1. Navigate into your MagicMirror's `modules` folder and execute `git clone https://github.com/thobach/MMM-Gestures.git`. A new folder will appear navigate into it.
2. Install dependencies via: `npm install serialport nodejs-websocket`
3. Follow below steps to install the Gestures Node.js App and add the module to your Magic Mirror configuration

### Gestures Node.js App on Raspberry Pi
Communication between Raspberry Pi and Arduino happens via the serial port (USB). The Node.js/gestures.js app from this project creates the connection between the two systems (Raspberry Pi and Arduino), forwards gesture and distance events to the web user interface via WebSocket communication and also controls the HDMI display to save power if nobody has interacted with or stood in front of the mirror for 5 minutes.

To start the Gestures Node.js App run: node "MMM-Gestures/Node.js/gestures.js". This node app cannot be run from Magic Mirror directly since Electron uses a non-compatible Node.js version for package serialport.

In order to launch the Node.js Gestures application on launch, you can adopt https://github.com/MichMich/MagicMirror/wiki/Auto-Starting-MagicMirror. You'll need to create a new shell script, e.g. gestures.sh and inside will need to call "node gestures.js" instead of "npm mm".

The following instructions assume you followed https://github.com/MichMich/MagicMirror/wiki/Auto-Starting-MagicMirror already.

Create a simple shell script to start the gesture application:
````
cd ~
nano gestures.sh
````

Add the following lines:
````
cd ~/MagicMirror/modules/MMM-Gestures/Node.js/
DISPLAY=:0 node gestures.js
````

Save and close, using the commands "CTRL-O" and "CTRL-X". Now make sure the shell script is executable by performing the following command:
````
chmod +x gestures.sh
````

Start the gesture application with the following command:
````
pm2 start gestures.sh
````

To make sure the gesture application restarts after rebooting, you need to save the current state of all scripts running via PM2. To do this, execute the following command
````
pm2 save
````

### Embedding MMM-Gestures
In order to load the MMM-Gestures module you will need to add the following configuration to your config/config.js file and ensure that the Node.js application gestures.js is loaded.
````javascript
modules: [
	{
		module: 'MMM-Gestures',
	},
]
````

## Reacting to Gestures
The MMM-Gestures.js Magic Mirror module listens to WebSocket events from the Gestures Node.js App and converts them to Magic Mirror-internal events that can be received via the built-in notificationReceived() function.

The received event has the following format:
* notification parameter: 'GESTURE'
* payload parameter: { gesture: 'UP/DOWN/LEFT/RIGHT/FAR/CLOSE/AWAY/PRESENT' }

Sample code for reacting to gesture events:
````javascript
notificationReceived: function(notification, payload, sender) {
	Log.info(this.name + " - received event");
	if(notification == 'GESTURE'){
		Log.info(this.name + " - received gesture");
		var gesture = payload.gesture;
		// actually RIGHT, because gesture sensor is built in upside down
		if(gesture.startsWith('LEFT')){
			Log.info(this.name + " - received right");
			
			// adjust some internal representation ...
			
			// update display
			this.updateDom(100);
		}
		// actually LEFT, because gesture sensor is built in upside down
		else if(gesture.startsWith('RIGHT')){
			...
		}
		// gesture event that was neither LEFT or RIGHT received
		else {
			Log.info(this.name + " - received other: " + gesture);
		}
	}
},
````

By default this module looks for the compliments module and only shows compliments when someone stands in front of the mirror. This is done by hiding the module by default, showing it when the "PRESENT" gesture is received and hiding the module again when the "AWAY" gesture is received. Further the user interface is reloaded when a FAR gesture is received, which can be useful user interface to testing purposes.

Available gestures:
* Distance sensor GP2Y0A21YK gestures:
** AWAY (fired when person is more than ~50cm away from the sensor)
** PRESENT (fired when person is less than ~50cm away from the sensor)
* Gesture sensor APDS-9960 gestures (only activated when person is in front of the mirror, reacts to movements e.g. of a hand, 3-10cm in front of the sensor):
** LEFT (left to right movement with an object)
** RIGHT (right to left movement with an object)
** UP (bottom to top movement with an object)
** DOWN (top to bottom movement with an object)
** FAR (close to far movement with an object)
** CLOSE (far to close movement with an object)

Note: If the sensor is installed upside-down the events from the gesture sensor APDS-9960 are reversed, e.g. a left gesture would be received as RIGHT. This does not apply to FAR and CLOSE gestures.

You can find a video demonstration of how the sensor works at https://www.youtube.com/watch?v=OS36IdgpEIo. Note, this break-out board is not optimal for use in Magic Mirror since all components are soldered on the front side. On Aliexpress you find nicer breakout boards that have only the sensor on the front side and the other components soldered to the back.

For debugging purposes you can configure the Magic Mirror's server IP in the MMM-Gestures config via:
````javascript
config: {
	serverIp: 'server-ip',
},
````

Afterwards you can open the Magic Mirror user interface in your normal browser: http://server-ip:8080 and the normal browser would connect to the WebSocket server on your Magic Mirror.

## Gesture-enabled Modules
The following modules react to gesture events from this module:
* Core application (reload user interface upon FAR gesture, useful for user interface testing)
* Compliments default module (no modification was needed since MMM-Gestures shows / hides the module upon PRESENT / AWAY events)
* MMM-NewsGestures (browse through news with left / right swipe, show news summary with up move and full news article with another up move, hides news summary or full article with down gesture)

If you build a Gesture-enabled Magic Mirror module, please let me know or create a pull request and I'll link it here.
