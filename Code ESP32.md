#include <WiFi.h>
#include <Firebase_ESP_Client.h>
#include <Adafruit_Fingerprint.h>
#include "addons/TokenHelper.h"
#include "addons/RTDBHelper.h"

// ⚙️ Định nghĩa chân UART cho cảm biến
#define FINGER_RX_PIN 16   // RX2 của ESP32 (nối TX cảm biến)
#define FINGER_TX_PIN 17   // TX2 của ESP32 (nối RX cảm biến)

//Định nghĩa nút nhấn
#define BUTTON_PIN 21

// Khởi tạo UART2 của ESP32
HardwareSerial mySerial(2);
Adafruit_Fingerprint finger = Adafruit_Fingerprint(&mySerial);
uint8_t id; // ID mẫu vân tay

// ================= Cấu hình WiFi =================
#define WIFI_SSID "?????_EXT"
#define WIFI_PASSWORD "luongloan7282@@"

//==================API kết nối Python=============
const char* host = "192.168.1.6";   // IP máy chạy Python
const uint16_t port = 5000;
WiFiClient client;

// ================= Firebase =================
#define DATABASE_URL "https://demo3-1b7e0-default-rtdb.firebaseio.com/"
#define DATABASE_SECRET "IPE8MdgjuurGQK9ffbLAIvjPIXlQTIPxTmw27oqg"

// ================= Biến toàn cục =================
FirebaseData fbdo;
FirebaseAuth auth;
FirebaseConfig config;

int giu;
uint8_t quet_van_tay = 0;
uint8_t ma_van_tay;
uint8_t flag = 0;
String so_id;
int so_id_can_xoa;
String ds_sinh_vien[50] = {};
String ds_xoa_sinh_vien[50] = {};
int idx = 1;
//=================Danh sách sinh viên====================
// String ds_sinh_vien[50] = {"0",
//                            "/DangKySinhVien/-ObhZkOxYPpKDo_ZAARN/id",
//                            "/DangKySinhVien/-ObhZzgYKTiuZGeAADGZ/id",
//                            "/DangKySinhVien/-Obh_508hDHG6rHqFv4_/id"};

// String ds_xoa_sinh_vien[50] = {"0",
//                                "/DangKySinhVien/-ObhZkOxYPpKDo_ZAARN",
//                                "/DangKySinhVien/-ObhZzgYKTiuZGeAADGZ",
//                                "/DangKySinhVien/-Obh_508hDHG6rHqFv4_"};
//----------------Nguyên mẫu hàm-------------------------
uint8_t getFingerprintID();
uint8_t deleteFingerprint(uint8_t id);
uint8_t readnumber(void);
uint8_t getFingerprintEnroll();
// ================= Setup =================
void setup() {
  Serial.begin(9600);

  pinMode(BUTTON_PIN, INPUT_PULLUP);  // Bật điện trở kéo lên nội bộ

  // Khởi tạo cổng Serial2
  mySerial.begin(57600, SERIAL_8N1, FINGER_RX_PIN, FINGER_TX_PIN);

  // Kết nối WiFi
  WiFi.begin(WIFI_SSID, WIFI_PASSWORD);
  Serial.print("Dang ket noi WiFi");
  while (WiFi.status() != WL_CONNECTED) {
    Serial.print(".");
    delay(300);
  }
  Serial.println();
  Serial.print("Da ket noi voi IP: ");
  Serial.println(WiFi.localIP());

  // Cấu hình Firebase
  config.database_url = DATABASE_URL;
  config.signer.tokens.legacy_token = DATABASE_SECRET;  // Dùng Database Secret

  Firebase.reconnectNetwork(true);
  Firebase.begin(&config, &auth);

  Serial.println("Firebase connected!");
  finger.begin(57600);

  if (finger.verifyPassword()) {
    Serial.println("✅ Found fingerprint sensor!");
  } else {
    Serial.println("❌ Did not find fingerprint sensor. Check wiring!");
    while (1) delay(1);
  }

  finger.getTemplateCount();
  if (finger.templateCount == 0) {
    Serial.println("No fingerprints found. Please enroll first!");
  } else {
    Serial.print("Sensor contains "); Serial.print(finger.templateCount); Serial.println(" templates");
  }

  //Lưu trường thông tin sinh viên vào mảng
  if (Firebase.RTDB.getShallowData(&fbdo, "/DangKySinhVien")) {
  FirebaseJson *json = fbdo.to<FirebaseJson *>();
  
  // Lấy iterator
  size_t count = json->iteratorBegin();
  String key, value;
  
  for (size_t i = 0; i < count; i++) {
    int type = 0;
    json->iteratorGet(i, type, key, value);
    ds_sinh_vien[idx] = "/DangKySinhVien/" + key + "/id";
    ds_xoa_sinh_vien[idx++] = "/DangKySinhVien/" + key;
    //Serial.println("Key: " + key);
  }
  
  json->iteratorEnd();
}
else {
  Serial.println("Lỗi đọc Firebase: " + fbdo.errorReason());
}
  Serial.println(ds_sinh_vien[1]); 
  Serial.println(ds_sinh_vien[2]); 
  Serial.println(ds_xoa_sinh_vien[1]); 
  Serial.println(ds_xoa_sinh_vien[2]);


  
}

// ================= Loop =================
void loop() {
  //===Nhận diện vân tay===
  if(getFingerprintID() != 0 && quet_van_tay == 1)
  { 
    Serial.print("Diem danh ID: "); Serial.println(giu);
    //Ghi dữ liệu điểm danh lên Firebase
    Firebase.RTDB.setInt(&fbdo,"/DiemDanh/id", giu);
    quet_van_tay = 0;
  }
  //===Kết nối Python===
  if (digitalRead(BUTTON_PIN) == LOW) {
    if (!client.connected()) {
      if (client.connect(host, port)) {
        Serial.println("Đã kết nối server Python");
      } else {
        Serial.println("Không kết nối được server");
        delay(1000);
        return;
      }
    }
  }
  //===Nhận data từ python===
  if (client.available()) {
    String msg = client.readStringUntil('\n');
    msg.trim();
    if (msg.length() > 0) {
      Serial.print("Nhận từ Python: ");
      Serial.println(msg);
      giu = msg.toInt();
      Firebase.RTDB.setInt(&fbdo,"/DiemDanh/id", giu);
    }
  }
  // int buttonState = digitalRead(BUTTON_PIN);
  // if (buttonState == LOW) {
  //   //---Đăng ký vân tay---
  //   Serial.println("==========================================");
  //   Serial.println("Cam bien van tay san sang");
  //   Serial.println("Nhap ID ban muon luu.");
  //   id = readnumber();
  //   if (id == 0) return;
  //   Serial.print("Dang ky ID #");
  //   Serial.println(id);
  //   while (!getFingerprintEnroll());

  //   if(flag == 1)
  //   {

  //     if (Firebase.RTDB.setString(&fbdo, ds_sinh_vien[id], ma_van_tay)) {
  //       Serial.println("Ghi du lieu thanh cong!");
  //       flag = 0;
  //     } else {
  //       Serial.print("Loi: ");
  //       Serial.println(fbdo.errorReason());
  //     }
  //   }
  // } 
  delay(50); // Chống nhiễu (debounce nhẹ)


  //===Đọc FireBase===
  if (Firebase.RTDB.getInt(&fbdo, "/xoa_sinhvien/id")){
    if (fbdo.dataType() == "string"){
      so_id = fbdo.stringData();
      Serial.print("so_id_can_xoa: ");
      Serial.println(so_id_can_xoa);
    }else{
      Serial.print("Loi doc du lieu: ");
      Serial.println(fbdo.errorReason());
    }
    
    so_id_can_xoa = so_id.toInt();

    //Duyệt để tìm
    Firebase.RTDB.getString(&fbdo, ds_sinh_vien[so_id_can_xoa]);
    String id_sinh_vien = fbdo.stringData();
    Serial.print("so_id_sinh_vien: ");
    Serial.println(id_sinh_vien);

    //===XÓa Firebase===
    if (Firebase.RTDB.deleteNode(&fbdo, ds_xoa_sinh_vien[so_id_can_xoa])) {
      Serial.println("✅ Da xoa thanh cong!");
      Firebase.RTDB.deleteNode(&fbdo, "/xoa_sinhvien/id");
    } else {
      Serial.println("❌ Loi xoa: " + fbdo.errorReason());
    }

    //Xóa phần cứng=====
    deleteFingerprint(so_id_can_xoa);
  }


}

//---------------Định nghĩa hàm--------------------
uint8_t getFingerprintID() {
  uint8_t p = finger.getImage();
  if (p != FINGERPRINT_OK) return p;

  p = finger.image2Tz();
  if (p != FINGERPRINT_OK) return p;

  p = finger.fingerSearch();
  if (p != FINGERPRINT_OK) {
    Serial.println("No match found");
    return p;
  }

  Serial.print("Found ID #"); Serial.print(finger.fingerID);
  Serial.print(" with confidence "); Serial.println(finger.confidence);
  giu = finger.fingerID;
  quet_van_tay = 1;
  return finger.fingerID;
}

uint8_t deleteFingerprint(uint8_t id) {
  uint8_t p = finger.deleteModel(id);

  switch (p) {
    case FINGERPRINT_OK:
      Serial.println("✅ Deleted successfully!");
      break;
    case FINGERPRINT_PACKETRECIEVEERR:
      Serial.println("❌ Communication error!");
      break;
    case FINGERPRINT_BADLOCATION:
      Serial.println("❌ Could not delete in that location!");
      break;
    case FINGERPRINT_FLASHERR:
      Serial.println("❌ Error writing to flash!");
      break;
    default:
      Serial.printf("❌ Unknown error: 0x%X\n", p);
      break;
  }
  return p;
}

uint8_t readnumber(void) {
  uint8_t num = 0;
  while (num == 0) {
    while (!Serial.available());
    num = Serial.parseInt();
  }
  return num;
}

uint8_t getFingerprintEnroll() {
  int p = -1;
  Serial.printf("Waiting for valid finger to enroll as #%d\n", id);
  while (p != FINGERPRINT_OK) {
    p = finger.getImage();
    switch (p) {
      case FINGERPRINT_OK: Serial.println("Image taken"); break;
      case FINGERPRINT_NOFINGER: Serial.print("."); break;
      case FINGERPRINT_PACKETRECIEVEERR: Serial.println("Communication error"); break;
      case FINGERPRINT_IMAGEFAIL: Serial.println("Imaging error"); break;
      default: Serial.println("Unknown error"); break;
    }
    delay(100);
  }

  p = finger.image2Tz(1);
  if (p != FINGERPRINT_OK) {
    Serial.println("Image conversion failed");
    return p;
  }
  Serial.println("Remove finger");
  delay(2000);

  while (p != FINGERPRINT_NOFINGER) {
    p = finger.getImage();
  }

  Serial.println("Place same finger again");
  p = -1;
  while (p != FINGERPRINT_OK) {
    p = finger.getImage();
    if (p == FINGERPRINT_NOFINGER) { Serial.print("."); delay(100); continue; }
    if (p == FINGERPRINT_OK) Serial.println("Image taken again");
  }

  p = finger.image2Tz(2);
  if (p != FINGERPRINT_OK) {
    Serial.println("Second image conversion failed");
    return p;
  }

  Serial.println("Creating model...");
  p = finger.createModel();
  if (p == FINGERPRINT_OK) Serial.println("Prints matched!");
  else {
    Serial.println("Fingerprints did not match or error occurred");
    return p;
  }

  p = finger.storeModel(id);
  if (p == FINGERPRINT_OK){
    Serial.printf("✅ Stored successfully as ID #%d\n", id);
    ma_van_tay = id;
    flag = 1;
  }
  else
    Serial.printf("❌ Error storing fingerprint (code %d)\n", p);
  return p;
}
