# Firebase_Web_GPIO_shtc3
## Code Esp8266
```
#include <Arduino.h>
#if defined(ESP32)
  #include <WiFi.h>
#elif defined(ESP8266)
  #include <ESP8266WiFi.h>
#endif
#include <Firebase_ESP_Client.h>
#include "Adafruit_SHTC3.h"

// Provide the token generation process info.
#include "addons/TokenHelper.h"
// Provide the RTDB payload printing info and other helper functions.
#include "addons/RTDBHelper.h"

// Insert your network credentials
#define WIFI_SSID "IoT_1Mbps"
#define WIFI_PASSWORD "1234567890"

// Insert Firebase project API Key
#define API_KEY "AIzaSyCylVpE18RtTy-KVtzUOGQu4UpOWB-cHyM"

// Insert Authorized Email and Corresponding Password
#define USER_EMAIL "kititach.k@en.rmutt.ac.th"
#define USER_PASSWORD "030413426Turk"

// Insert RTDB URLefine the RTDB URL
#define DATABASE_URL "https://esp8266-shtc3-efabd-default-rtdb.firebaseio.com/"

// Define Firebase objects
FirebaseData fbdo;
FirebaseData stream;
FirebaseAuth auth;
FirebaseConfig config;

// Variable to save USER UID
String uid;

// Variables to save database paths
String listenerPath;
String databasePath;
String tempPath;
String humPath;

// shtc3 sensor
Adafruit_SHTC3 shtc3 = Adafruit_SHTC3(); // I2C
float temperature;
float humidity;

// Timer variables (send new readings every three minutes)
unsigned long sendDataPrevMillis = 0;
unsigned long timerDelay = 180000;

// Declare outputs
const int output1 = 12;
const int output2 = 13;
const int output3 = 14;

// Initialize WiFi
void initWiFi() {
  WiFi.begin(WIFI_SSID, WIFI_PASSWORD);
  Serial.print("Connecting to WiFi ..");
  while (WiFi.status() != WL_CONNECTED) {
    Serial.print('.');
    delay(1000);
  }
  Serial.println(WiFi.localIP());
  Serial.println();
}

// Write float values to the database
void sendFloat(String path, float value){
  if (Firebase.RTDB.setFloat(&stream, path.c_str(), value)){
    Serial.print("Writing value: ");
    Serial.print (value);
    Serial.print(" on the following path: ");
    Serial.println(path);
    Serial.println("PASSED");
    Serial.println("PATH: " + stream.dataPath());
    Serial.println("TYPE: " + stream.dataType());
  }
  else {
    Serial.println("FAILED");
    Serial.println("REASON: " + stream.errorReason());
  }
}

// Callback function that runs on database changes
void streamCallback(FirebaseStream data){
  Serial.printf("stream path, %s\nevent path, %s\ndata type, %s\nevent type, %s\n\n",
                data.streamPath().c_str(),
                data.dataPath().c_str(),
                data.dataType().c_str(),
                data.eventType().c_str());
  printResult(data); //see addons/RTDBHelper.h
  Serial.println();

  // Get the path that triggered the function
  String streamPath = String(data.dataPath());

  // if the data returned is an integer, there was a change on the GPIO state on the following path /{gpio_number}
  if (data.dataTypeEnum() == fb_esp_rtdb_data_type_integer){
    String gpio = streamPath.substring(1);
    int state = data.intData();
    Serial.print("GPIO: ");
    Serial.println(gpio);
    Serial.print("STATE: ");
    Serial.println(state);
    digitalWrite(gpio.toInt(), state);
  }

  /* When it first runs, it is triggered on the root (/) path and returns a JSON with all keys
  and values of that path. So, we can get all values from the database and updated the GPIO states*/
  if (data.dataTypeEnum() == fb_esp_rtdb_data_type_json){
    FirebaseJson json = data.to<FirebaseJson>();

    // To iterate all values in Json object
    size_t count = json.iteratorBegin();
    Serial.println("\n---------");
    for (size_t i = 0; i < count; i++){
        FirebaseJson::IteratorValue value = json.valueAt(i);
        int gpio = value.key.toInt();
        int state = value.value.toInt();
        Serial.print("STATE: ");
        Serial.println(state);
        Serial.print("GPIO:");
        Serial.println(gpio);
        digitalWrite(gpio, state);
        Serial.printf("Name: %s, Value: %s, Type: %s\n", value.key.c_str(), value.value.c_str(), value.type == FirebaseJson::JSON_OBJECT ? "object" : "array");
    }
    Serial.println();
    json.iteratorEnd(); // required for free the used memory in iteration (node data collection)
  }
  
  //This is the size of stream payload received (current and max value)
  //Max payload size is the payload size under the stream path since the stream connected
  //and read once and will not update until stream reconnection takes place.
  //This max value will be zero as no payload received in case of ESP8266 which
  //BearSSL reserved Rx buffer size is less than the actual stream payload.
  Serial.printf("Received stream payload size: %d (Max. %d)\n\n", data.payloadLength(), data.maxPayloadLength());
}

void streamTimeoutCallback(bool timeout){
  if (timeout)
    Serial.println("stream timeout, resuming...\n");
  if (!stream.httpConnected())
    Serial.printf("error code: %d, reason: %s\n\n", stream.httpCode(), stream.errorReason().c_str());
}

void setup(){
  Serial.begin(115200);
  initWiFi();

  Serial.println("SHTC3 test");
  if (! shtc3.begin()) {
    Serial.println("Couldn't find SHTC3");
    while (1) delay(1);
  }
  Serial.println("Found SHTC3 sensor");

  // Initialize Outputs
  pinMode(output1, OUTPUT);
  pinMode(output2, OUTPUT);
  pinMode(output3, OUTPUT);
  
  // Assign the api key (required)
  config.api_key = API_KEY;

  // Assign the user sign in credentials
  auth.user.email = USER_EMAIL;
  auth.user.password = USER_PASSWORD;

  // Assign the RTDB URL (required)
  config.database_url = DATABASE_URL;

  Firebase.reconnectWiFi(true);
  stream.setResponseSize(4096);

  // Assign the callback function for the long running token generation task */
  config.token_status_callback = tokenStatusCallback; //see addons/TokenHelper.h

  // Assign the maximum retry of token generation
  config.max_token_generation_retry = 5;

  // Initialize the library with the Firebase authen and config
  Firebase.begin(&config, &auth);

  // Getting the user UID might take a few seconds
  Serial.println("Getting User UID");
  while ((auth.token.uid) == "") {
    Serial.print('.');
    delay(1000);
  }
  
  // Print user UID
  uid = auth.token.uid.c_str();
  Serial.print("User UID: ");
  Serial.println(uid);

  // Update database path
  databasePath = "/UsersData/" + uid;

  // Update database path for sensor readings
  tempPath = databasePath + "/temperature"; // --> UsersData/<user_uid>/temperature
  humPath = databasePath + "/humidity"; // --> UsersData/<user_uid>/humidity
  listenerPath = databasePath + "/GPIO/";

  // Streaming (whenever data changes on a path)
  // Begin stream on a database path --> board1/outputs/digital
  if (!Firebase.RTDB.beginStream(&stream, listenerPath.c_str()))
    Serial.printf("stream begin error, %s\n\n", stream.errorReason().c_str());

  // Assign a calback function to run when it detects changes on the database
  Firebase.RTDB.setStreamCallback(&stream, streamCallback, streamTimeoutCallback);

  delay(2000);
}

void loop(){
  sensors_event_t humiditys, temp;
  shtc3.getEvent(&humiditys, &temp);

  // Send new readings to database
  if (Firebase.ready() && (millis() - sendDataPrevMillis > timerDelay || sendDataPrevMillis == 0)){
    sendDataPrevMillis = millis();

    // Get latest sensor readings
    temperature = temp.temperature;
    humidity = humiditys.relative_humidity;

    // Send readings to database:
    sendFloat(tempPath, temperature);
    sendFloat(humPath, humidity);
    }
}
```

## index.html
```
<!DOCTYPE html>
<!-- Complete Project Details at: https://RandomNerdTutorials.com/ -->
<html>
  <head>
    <meta charset="utf-8">
    <meta name="viewport" content="width=device-width, initial-scale=1">
    <title>ESP IoT Firebase App</title>

    <!-- update the version number as needed -->
    <script src="https://www.gstatic.com/firebasejs/8.8.1/firebase-app.js"></script>

    <!-- include only the Firebase features as you need -->
    <script src="https://www.gstatic.com/firebasejs/8.8.1/firebase-auth.js"></script>
    <script src="https://www.gstatic.com/firebasejs/8.8.1/firebase-database.js"></script>

    <script>
      // REPLACE WITH YOUR web app's Firebase configuration
       const firebaseConfig = {
         apiKey: "AIzaSyCylVpE18RtTy-KVtzUOGQu4UpOWB-cHyM",
         authDomain: "esp8266-shtc3-efabd.firebaseapp.com",
         databaseURL: "https://esp8266-shtc3-efabd-default-rtdb.firebaseio.com",
         projectId: "esp8266-shtc3-efabd",
         storageBucket: "esp8266-shtc3-efabd.appspot.com",
         messagingSenderId: "680482548907",
         appId: "1:680482548907:web:1779768a73d1f375161bee"
       };

      // Initialize firebase
      firebase.initializeApp(firebaseConfig);

      // Make auth and database references
      const auth = firebase.auth();
      const db = firebase.database();

    </script>
    <link rel="stylesheet" href="https://use.fontawesome.com/releases/v5.7.2/css/all.css" integrity="sha384-fnmOCqbTlWIlj8LyTjo7mOUStjsKC4pOpQbqyi7RrhN7udi9RwhKkMHpvLbHG9Sr" crossorigin="anonymous">
    <link rel="icon" type="image/png" href="favicon.png">
    <link rel="stylesheet" type="text/css" href="style.css">
</head>

<body>
  <!--TOP BAR-->
  <div class="topnav">
    <h1>Sensor Readings App <i class="fas fa-lightbulb"></i></h1>
  </div>

  <!--AUTHENTICATION BAR (USER DETAILS/LOGOUT BUTTON)-->
  <div id="authentication-bar" style="display: none;">
    <p><span id="authentication-status">User logged in</span>
       <span id="user-details">USEREMAIL</span>
       <a href="/" id="logout-link">(logout)</a>
    </p>
  </div>

  <!--LOGIN FORM-->
  <form id="login-form" style="display: none;">
    <div class="form-elements-container">
      <label for="input-email"><b>Email</b></label>
      <input type="text" placeholder="Enter Username" id="input-email" required>

      <label for="input-password"><b>Password</b></label>
      <input type="password" placeholder="Enter Password" id="input-password" required>

      <button type="submit" id="login-button">Login</button>
      <p id="error-message" style="color:red;"></p>
    </div>
  </form>

  <!--CONTENT (SENSOR READINGS)-->
  <div class="content-sign-in" id="content-sign-in" style="display: none;">
    <div class="card-grid">
      <!--CARD FOR GPIO 12-->
      <div class="card">
        <p class="card-title"><i class="fas fa-lightbulb"></i> GPIO 12</p>
        <p>
          <button class="button-on" id="btn1On">ON</button>
          <button class="button-off" id="btn1Off">OFF</button>
        </p>
        <p class="state">State:<span id="state1"></span></p>
      </div>

      <!--CARD FOR GPIO 13-->
      <div class="card">
        <p class="card-title"><i class="fas fa-lightbulb"></i> GPIO 13</p>
        <p>
          <button class="button-on" id="btn2On">ON</button>
          <button class="button-off" id="btn2Off">OFF</button>
        </p>
        <p class="state">State:<span id="state2"></span></p>
      </div>

      <!--CARD FOR GPIO 14-->
      <div class="card">
        <p class="card-title"><i class="fas fa-lightbulb"></i> GPIO 14</p>
        <p>
          <button class="button-on" id="btn3On">ON</button>
          <button class="button-off" id="btn3Off">OFF</button>
        </p>
        <p class="state">State:<span id="state3"></span></p>
      </div>
  
      <!--TEMPERATURE-->
      <div class="card">
        <p><i class="fas fa-thermometer-half" style="color:#059e8a;"></i> TEMPERATURE</p>
        <p><span class="reading"><span id="temp"></span> &deg;C</span></p>
      </div>
      <!--HUMIDITY-->
      <div class="card">
        <p><i class="fas fa-tint" style="color:#00add6;"></i> HUMIDITY</p>
        <p><span class="reading"><span id="hum"></span> &percnt;</span></p>
      </div>
    </div>
  </div>
    <script src="scripts/auth.js"></script>
    <script src="scripts/index.js"></script>
  </body>
</html>
```

## style.css
```
html {
    font-family: Verdana, Geneva, Tahoma, sans-serif;
    display: inline-block;
    text-align: center;
}
h1 {
    font-size: 1.8rem;
    color: white;
}
.topnav {
    overflow: hidden;
    background-color: #049faa;
    color: white;
    font-size: 1rem;
    padding: 10px;
}
#authentication-bar{
    background-color:mintcream;
    padding-top: 10px;
    padding-bottom: 10px;
}

#user-details{
    color: cadetblue;
}

.content {
    padding: 20px;
}

body {
    margin: 0;
}

.card-grid {
    max-width: 800px;
    margin: 0 auto;
    display: grid;
    grid-gap: 2rem;
    grid-template-columns: repeat(auto-fit, minmax(200px, 1fr));
}
.card {
    background-color: white;
    box-shadow: 2px 2px 12px 1px rgba(140,140,140,.5);
}

.card-title {
    font-size: 1.2rem;
    color: #034078
}
.state {
    color:#1282A2;
}
button {
    background-color: #049faa;
    color: white;
    padding: 14px 20px;
    margin: 8px 0;
    border: none;
    cursor: pointer;
    border-radius: 4px;
}
.button-on {
    background-color:#034078;
}
.button-on:hover {
    background-color: #1282A2;
}
.button-off {
    background-color:#858585;
}
.button-off:hover {
    background-color: #252524;
}

.form-elements-container{
    padding: 16px;
    width: 250px;
    margin: 0 auto;
}

input[type=text], input[type=password] {
    width: 100%;
    padding: 12px 20px;
    margin: 8px 0;
    display: inline-block;
    border: 1px solid #ccc;
    box-sizing: border-box;
}
```

## auth.js
```
document.addEventListener("DOMContentLoaded", function(){

    // listen for auth status changes
    auth.onAuthStateChanged(user => {
        if (user) {
            console.log("user logged in");
            console.log(user);
            setupUI(user);
            var uid = user.uid;
            console.log(uid);
        } else {
            console.log("user logged out");
            setupUI();
        }
    });

    // login
    const loginForm = document.querySelector('#login-form');
    loginForm.addEventListener('submit', (e) => {
        e.preventDefault();
        // get user info
        const email = loginForm['input-email'].value;
        const password = loginForm['input-password'].value;
        // log the user in
        auth.signInWithEmailAndPassword(email, password).then((cred) => {
            // close the login modal & reset form
            loginForm.reset();
            console.log(email);
        })
        .catch((error) =>{
            const errorCode = error.code;
            const errorMessage = error.message;
            document.getElementById("error-message").innerHTML = errorMessage;
            console.log(errorMessage);
        });
    });

    // logout
    const logout = document.querySelector('#logout-link');
    logout.addEventListener('click', (e) => {
    e.preventDefault();
    auth.signOut();
    });
    
});
```

## index.js
```
const loginElement = document.querySelector('#login-form');
const contentElement = document.querySelector("#content-sign-in");
const userDetailsElement = document.querySelector('#user-details');
const authBarElement = document.querySelector("#authentication-bar");

// Elements for GPIO states
const stateElement1 = document.getElementById("state1");
const stateElement2 = document.getElementById("state2");
const stateElement3 = document.getElementById("state3");

// Elements for sensor readings
const tempElement = document.getElementById("temp");
const humElement = document.getElementById("hum");
const presElement = document.getElementById("pres");

// Button Elements
const btn1On = document.getElementById('btn1On');
const btn1Off = document.getElementById('btn1Off');
const btn2On = document.getElementById('btn2On');
const btn2Off = document.getElementById('btn2Off');
const btn3On = document.getElementById('btn3On');
const btn3Off = document.getElementById('btn3Off');

// MANAGE LOGIN/LOGOUT UI
const setupUI = (user) => {
  if (user) {
    //toggle UI elements
    loginElement.style.display = 'none';
    contentElement.style.display = 'block';
    authBarElement.style.display ='block';
    userDetailsElement.style.display ='block';
    userDetailsElement.innerHTML = user.email;

    var uid = user.uid;
    console.log(uid);

    // Database path for GPIO states
    var dbPathOutput1 = 'UsersData/' + uid.toString() + '/GPIO/12';
    var dbPathOutput2 = 'UsersData/' + uid.toString() + '/GPIO/13';
    var dbPathOutput3 = 'UsersData/' + uid.toString() + '/GPIO/14';

    // Database references
    var dbRefOutput1 = firebase.database().ref().child(dbPathOutput1);
    var dbRefOutput2 = firebase.database().ref().child(dbPathOutput2);
    var dbRefOutput3 = firebase.database().ref().child(dbPathOutput3);

    // Database paths (with user UID)
    var dbPathTemp = 'UsersData/' + uid.toString() + '/temperature';
    var dbPathHum = 'UsersData/' + uid.toString() + '/humidity';
    var dbPathPres = 'UsersData/' + uid.toString() + '/pressure';

    // Database references
    var dbRefTemp = firebase.database().ref().child(dbPathTemp);
    var dbRefHum = firebase.database().ref().child(dbPathHum);
    var dbRefPres = firebase.database().ref().child(dbPathPres);
        
    //Update states depending on the database value
    dbRefOutput1.on('value', snap => {
        if(snap.val()==1) {
            stateElement1.innerText="ON";
        }
        else{
            stateElement1.innerText="OFF";
        }
    });
    dbRefOutput2.on('value', snap => {
        if(snap.val()==1) {
            stateElement2.innerText="ON";
        }
        else{
            stateElement2.innerText="OFF";
        }
    });
    dbRefOutput3.on('value', snap => {
        if(snap.val()==1) {
            stateElement3.innerText="ON";
        }
        else{
            stateElement3.innerText="OFF";
        }
    });

    // Update database uppon button click
    btn1On.onclick = () =>{
        dbRefOutput1.set(1);
    }
    btn1Off.onclick = () =>{
        dbRefOutput1.set(0);
    }

    btn2On.onclick = () =>{
        dbRefOutput2.set(1);
    }
    btn2Off.onclick = () =>{
        dbRefOutput2.set(0);
    }

    btn3On.onclick = () =>{
        dbRefOutput3.set(1);
    }
    btn3Off.onclick = () =>{
        dbRefOutput3.set(0);
    }

    // Update page with new readings
    dbRefTemp.on('value', snap => {
        tempElement.innerText = snap.val().toFixed(2);
      });
  
      dbRefHum.on('value', snap => {
        humElement.innerText = snap.val().toFixed(2);
      });

  // if user is logged out
  } else{
    // toggle UI elements
    loginElement.style.display = 'block';
    authBarElement.style.display ='none';
    userDetailsElement.style.display ='none';
    contentElement.style.display = 'none';
  }
}
```
