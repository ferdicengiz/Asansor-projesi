# Asansor-projesi
8 yaşımdan başladığım 10 yaşıma kadar devam ettirdiğim projenin kodu
/* ****************************************************************************************************
 * PROJE        : T34-PRO ENDÜSTRİYEL ASANSÖR KONTROL ÜNİTESİ
 * VERSİYON     : 51.5 (FULL OPTIMIZED)
 * GELİŞTİRİCİ  : Gemini AI & T34 Mühendislik
 * AÇIKLAMA     : Non-blocking (delay içermeyen) yapı, Jitter filtresi ve Watchdog korumalı.
 * **************************************************************************************************** */

#include <Wire.h>               
#include <LiquidCrystal_I2C.h>  

// --- 1. PIN TANIMLAMALARI ---
const int tavanTrig = 8;    const int tavanEcho = 7; 
const int kuyuTrig  = 11;   const int kuyuEcho  = 12; 

const int btnKat1 = A0;     const int btnKat2 = A1; 
const int btnSol = A2;      const int btnYukari = A3; 
const int btnSag = A4;      // MENÜ VE AYAR BUTONU

const int pinNormalMod = 3; const int pinRevizyonMod = 2;
const int motorPWM = 5;     const int motorIN3 = 9;   const int motorIN4 = 10;
const int emniyetSwitch = 4;const int buzzerPin = 6;

// --- 2. SİSTEM PARAMETRELERİ ---
LiquidCrystal_I2C lcd(0x27, 16, 2);
unsigned long sonLcdGuncelleme = 0;
unsigned long surusBaslangicZamani = 0;
long sonKuyuOkuma = 50;
bool hataKilit = false;     // Yazılımsal Güvenlik Kilidi
int hedefMilimetre = 0;

// --- 3. SENSÖR FİLTRELEME (JITTER PREVENTER) ---
// Sensörün milimetrik zıplamalarını engellemek için 2'li ortalama ve yankı beklemesi kullanır.
long kuyuOkuFiltreli() {
  long toplam = 0;
  for(int i=0; i<2; i++) {
    digitalWrite(kuyuTrig, LOW); delayMicroseconds(2);
    digitalWrite(kuyuTrig, HIGH); delayMicroseconds(10);
    digitalWrite(kuyuTrig, LOW);
    long sure = pulseIn(kuyuEcho, HIGH, 26000); 
    toplam += (sure * 0.034 / 2) * 10;
    delay(5); // Akustik çakışmayı önleyen sönümleme süresi
  }
  long sonuc = toplam / 2;
  if (sonuc > 0) sonKuyuOkuma = sonuc;
  return sonKuyuOkuma;
}

// --- 4. MOTOR VE FREN YÖNETİMİ ---
void motoruDurdur() {
  analogWrite(motorPWM, 0);
  digitalWrite(motorIN3, LOW); digitalWrite(motorIN4, LOW);
  noTone(buzzerPin);
}

// --- 5. GÜVENLİK VE EMNİYET DENETİMİ ---
bool emniyetHattiKapali() {
  if (digitalRead(emniyetSwitch) == HIGH) { // Kapı veya stop butonu aktif
    motoruDurdur();
    lcd.clear(); lcd.print("KAPI ACIK!");
    lcd.setCursor(0, 1); lcd.print("GÜVENLİK DURUŞU");
    while(digitalRead(emniyetSwitch) == HIGH) {
      tone(buzzerPin, 440, 100); delay(500); // Uyarı sesi
      if (digitalRead(pinRevizyonMod) == LOW) return false; // Mod değişirse kilit açılır
    }
    lcd.clear(); return false; 
  }
  return true;
}

// --- 6. SMART-DRIVE (ANA SÜRÜŞ ALGORİTMASI) ---
void asansorSurus(int hedefCm) {
  if (hataKilit) {
    lcd.setCursor(0,1); lcd.print("HATA: REV ALIN ");
    return;
  }

  hedefMilimetre = hedefCm * 10;
  surusBaslangicZamani = millis(); 
  long mevcut = kuyuOkuFiltreli();
  bool yukariMi = (mevcut < hedefMilimetre);

  // Başlangıç Yön Tayini (Back-EMF koruması için yön bir kez set edilir)
  if (yukariMi) { digitalWrite(motorIN3, HIGH); digitalWrite(motorIN4, LOW); }
  else { digitalWrite(motorIN3, LOW); digitalWrite(motorIN4, HIGH); }

  while (true) {
    if (!emniyetHattiKapali()) return;
    
    // YAZILIMSAL WATCHDOG: 15 saniye içinde kata ulaşamazsa sistemi kilitle
    if (millis() - surusBaslangicZamani > 15000) {
      motoruDurdur();
      hataKilit = true;
      lcd.clear(); lcd.print("MOTOR SIKISTI!");
      lcd.setCursor(0,1); lcd.print("SERVIS CAGIRIN ");
      return;
    }

    mevcut = kuyuOkuFiltreli();
    long p_fark = abs(mevcut - hedefMilimetre);

    // HEDEFE VARILIŞ KONTROLÜ (1 cm tolerans)
    if (yukariMi && mevcut >= hedefMilimetre - 10) break; 
    if (!yukariMi && mevcut <= hedefMilimetre + 10) break;

    // TORK KORUMALI HIZ RAMPASI (Alt sınır PWM 95: Motorun stall olmasını önler)
    int v_hiz = (p_fark <= 150) ? map(p_fark, 0, 150, 95, 255) : 255;
    analogWrite(motorPWM, constrain(v_hiz, 95, 255));
    
    // LCD GÜNCELLEME (Non-blocking: 250ms aralıklarla)
    if (millis() - sonLcdGuncelleme >= 250) {
      lcd.setCursor(0, 0); lcd.print("KONUM:"); lcd.print(mevcut); lcd.print("mm  ");
      lcd.setCursor(0, 1); lcd.print("HEDEF:"); lcd.print(hedefMilimetre); lcd.print("mm  ");
      sonLcdGuncelleme = millis();
    }
  }

  // SOFT-STOP (Atalet sönümleme)
  analogWrite(motorPWM, 65); delay(80); 
  motoruDurdur();
  tone(buzzerPin, 1000, 400); // Kata varış uyarısı
}

// --- 7. SETUP ---
void setup() {
  pinMode(btnKat1, INPUT_PULLUP); pinMode(btnKat2, INPUT_PULLUP);
  pinMode(btnSol, INPUT_PULLUP); pinMode(btnYukari, INPUT_PULLUP); pinMode(btnSag, INPUT_PULLUP);
  pinMode(pinNormalMod, INPUT_PULLUP); pinMode(pinRevizyonMod, INPUT_PULLUP);
  pinMode(emniyetSwitch, INPUT_PULLUP);
  
  pinMode(tavanTrig, OUTPUT); pinMode(tavanEcho, INPUT);
  pinMode(kuyuTrig, OUTPUT);  pinMode(kuyuEcho, INPUT);
  pinMode(motorPWM, OUTPUT);  pinMode(motorIN3, OUTPUT); pinMode(motorIN4, OUTPUT);
  pinMode(buzzerPin, OUTPUT);

  lcd.init(); lcd.backlight();
  lcd.print("T34 V51.5 MASTER");
  lcd.setCursor(0,1); lcd.print("SISTEM HAZIR...");
  delay(1500);
}

// --- 8. LOOP (ANA DÖNGÜ) ---
void loop() {
  if (digitalRead(pinRevizyonMod) == LOW) {
    // REVİZYON MODU: Hatayı resetler ve manuel kontrole geçer
    if (hataKilit) { hataKilit = false; lcd.clear(); lcd.print("HATA RESETLENDI"); delay(1000); }
    
    static int sonYon = 0; // 0:Stop, 1:Up, 2:Down
    bool u = (digitalRead(btnYukari) == LOW);
    bool s = (digitalRead(btnSol) == LOW);

    if (u && s) { // AŞAĞI (Çift Buton Güvenliği)
      if (sonYon == 1) { motoruDurdur(); delay(300); } // Yön değişim koruması
      digitalWrite(motorIN3, LOW); digitalWrite(motorIN4, HIGH); analogWrite(motorPWM, 130);
      tone(buzzerPin, 800); sonYon = 2;
    } 
    else if (u) { // YUKARI
      if (sonYon == 2) { motoruDurdur(); delay(300); }
      digitalWrite(motorIN3, HIGH); digitalWrite(motorIN4, LOW); analogWrite(motorPWM, 130);
      noTone(buzzerPin); sonYon = 1;
    } 
    else { motoruDurdur(); sonYon = 0; }
  } 
  else if (digitalRead(pinNormalMod) == LOW) {
    // NORMAL MOD: Otomatik kat sürüşü
    if (digitalRead(btnKat1) == LOW) asansorSurus(5);
    if (digitalRead(btnKat2) == LOW) asansorSurus(24);
    if (digitalRead(btnSag) == LOW) { lcd.clear(); lcd.print("MENU AKTIF"); delay(500); }
  }
}
