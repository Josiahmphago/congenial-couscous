# congenial-couscous
/*id like to use wifi manager to add more credentials without hard coding ,SSID,PASSWORD,BOTID ,TOKEN and Camera identity
im struggling to get ESP32-cam to communicate with telegram  ive also tried OTP to secure the credentials but nothing works , here is the code below:*/

#include <WiFiManager.h>
#include <Preferences.h>
#include <WiFiClientSecure.h>
#include <UniversalTelegramBot.h>
#include <ArduinoJson.h>

// Wi-Fi and Telegram
WiFiManager wm;
Preferences preferences;
WiFiClientSecure client;
UniversalTelegramBot bot("YOUR_BOT_TOKEN", client); // Replace with your bot token
String chat_id = "YOUR_CHAT_ID";  // Replace with your chat ID
String botToken, botId, camId;

// Replace with your SMS API or function
String phoneNumber = "+1234567890";  // Your phone number to receive OTP
String generatedOTP = "123456";  // Placeholder OTP, replace with real OTP generation
String password = wm.getWiFiPass();  // Correct function

// Function to send the OTP to the specified phone number
void sendOTP() {
  Serial.println("Sending OTP to phone: " + phoneNumber);
  // Example: Use an SMS API like Twilio to send OTP (for now it just simulates)
}

// Function to verify the OTP input
bool verifyOTP(String otpInput) {
  return otpInput == generatedOTP;  // Verify the input OTP
}

void setup() {
  Serial.begin(115200);
  client.setInsecure();  // Required for Telegram to skip SSL cert verification
  preferences.begin("credentials", false);

  // Load saved credentials
  String storedSSID = preferences.getString("ssid", "");
  String storedPassword = preferences.getString("password", "");
  botToken = preferences.getString("botToken", "");
  botId = preferences.getString("botId", "");
  camId = preferences.getString("camId", "");

  // WiFi Manager setup
  WiFiManagerParameter custom_botId("botId", "Enter BOT ID", botId.c_str(), 50);
  WiFiManagerParameter custom_botToken("botToken", "Enter Bot Token", botToken.c_str(), 50);
  WiFiManagerParameter custom_camId("camId", "Enter Camera Identity", camId.c_str(), 50);
  
  wm.addParameter(&custom_botId);
  wm.addParameter(&custom_botToken);
  wm.addParameter(&custom_camId);

  if (storedSSID == "") {
    // Open Wi-Fi Manager if no credentials
    if (!wm.autoConnect("ESP")) {
      Serial.println("Failed to connect to Wi-Fi Manager");
    }

    // Save new credentials
    preferences.putString("ssid", wm.getWiFiSSID());
    preferences.putString("password", wm.getWiFiPass());
    preferences.putString("botId", custom_botId.getValue());
    preferences.putString("botToken", custom_botToken.getValue());
    preferences.putString("camId", custom_camId.getValue());
    
  } else {
    // Connect to saved Wi-Fi credentials
    WiFi.begin(storedSSID.c_str(), storedPassword.c_str());
    if (WiFi.waitForConnectResult() != WL_CONNECTED) {
      Serial.println("Failed to connect to Wi-Fi");
    } else {
      Serial.println("Connected to Wi-Fi");
    }
  }

  // Telegram Connection Test
  bool botConnected = bot.getMe();  // returns true if connected
   if (botConnected){
    Serial.println("Telegram Bot Connected!");
    bot.sendMessage(chat_id, "Bot is now online!", "");
  } else {
    Serial.println("Failed to connect to Telegram Bot.");
  }
}

// Access WiFi Manager after OTP verification
void accessWiFiManager() {
  String otpInput;
  
  // Send OTP to phone
  sendOTP();
  
  Serial.println("Enter the OTP sent to your phone:");
  while (Serial.available() == 0) {
    // Wait for user to enter OTP via serial
  }
  
  otpInput = Serial.readStringUntil('\n');
  
  if (verifyOTP(otpInput)) {
    Serial.println("OTP verified. Accessing Wi-Fi Manager.");
    // Open Wi-Fi Manager for credential changes
    if (!wm.startConfigPortal("ESP")) {
      Serial.println("Failed to start Wi-Fi Manager");
    } else {
      Serial.println("Wi-Fi Manager opened.");
    }
  } else {
    Serial.println("Invalid OTP. Access denied.");
  }
}

// Function to handle Telegram messages
void handleTelegramMessages() {
  int numNewMessages = bot.getUpdates(bot.last_message_received + 1);
  while (numNewMessages) {
    for (int i = 0; i < numNewMessages; i++) {
      String text = bot.messages[i].text;

      // Start Command
      if (text == "/start") {
        bot.sendMessage(chat_id, "Welcome! Type /otp to get your OTP.", "");
      }

      // OTP Command
      else if (text == "/otp") {
        sendOTP();
        bot.sendMessage(chat_id, "OTP sent to your phone number.", "");
      }
    }
    numNewMessages = bot.getUpdates(bot.last_message_received + 1);
  }
}

void loop() {
  // Monitor Telegram messages
  handleTelegramMessages();

  // Monitor if user wants to access Wi-Fi Manager again via OTP
  if (Serial.available() > 0) {
    String input = Serial.readStringUntil('\n');
    if (input == "reset") {
      accessWiFiManager();
    }
  }
}
