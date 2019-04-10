# 미세제로정류장 코드 가이드라인

## 1. 첨부한 라이브러리를 아두이노에서 import 해준다.

![스크린샷 2019-04-10 오전 11.56.34](/Users/limdongjoon/Desktop/Screenshots/스크린샷 2019-04-10 오전 11.56.34.png)



## 2. 미세먼지 센서 연결

<https://blog.naver.com/roboholic84/220426416136>

위 블로그 참조

**회로도**

- 센서 : 아두이노
- 검은색 선 : GND
- 빨간색 선 : 5V
- 노란색 선 : Digital 8

## 2. 소스코드 작성

```c
#include <LiquidCrystal_I2C.h>/*LiquidCrystal_I2C*/ 

#define DUST_SENSER 8 // 미세먼지 센서 디지털 핀번호
#define O_PUMP_A 6
#define O_PUMP_B 5
#define O_RGB_R 10
#define O_RGB_G 9
#define O_RGB_B 11

#define On_Time 500

#define LCD_I2C_ADDR 0x27

LiquidCrystal_I2C lcd(LCD_I2C_ADDR, 16, 2);

/*미세먼지 센서 관련 변수들 선언*/
int dustDensity;
unsigned long duration;
unsigned long starttime;
unsigned long sampletime_ms = 300000;// 5초에 한번씩 측정
unsigned long lowpulseoccupancy = 0;
float ratio = 0;
float concentration = 0;
float concentration_ugm3 = 0;

/*디지털핀 초기화하기*/
void initPin() {
  pinMode(O_PUMP_A, OUTPUT); // 펌프
  pinMode(O_PUMP_B, OUTPUT); // 펌프
  pinMode(O_RGB_R, OUTPUT); // LED
  pinMode(O_RGB_G, OUTPUT); // LED
  pinMode(O_RGB_B, OUTPUT); // LED

  pinMode(DUST_SENSER, INPUT); // 미세먼지 센서

  digitalWrite(O_RGB_R, LOW); // LED
  digitalWrite(O_RGB_G, LOW); // LED
  digitalWrite(O_RGB_B, LOW); // LED
  analogWrite(O_PUMP_A, 0); // 펌프
  analogWrite(O_PUMP_B, 0); // 펌프
}

/*LCD INTRO출력하기*/
void introLcd() {
  lcd.print("Dust Zero Stop");
  lcd.setCursor(0, 1);
  lcd.print("Start");
}

/*LCD 습도 프린트하기*/
void printLcd() {
  //lcd.init();
  lcd.clear();
  lcd.backlight();
  lcd.setCursor(0, 0);
  lcd.print("미세먼지 농도 : ");
  lcd.print(dustDensity);
  lcd.print("%");
  lcd.setCursor(0, 1);
  // 미세먼지 값에 따라 다른 메세지 출력해주기
  if(dustDensity < 15) lcd.print("미세먼지 상태: 좋음");
  else if(dustDensity < 35) lcd.print("미세먼지 상태: 보통");
  else if(dustDensity < 75) lcd.print("미세먼지 상태: 나쁨");
  else lcd.print("미세먼지 상태: 매우나쁨");
}

/*LCD 초기화하기*/
void initLcd() {
  lcd.init();
  lcd.backlight();
  lcd.setCursor(0, 0);
  introLcd();
}

/*미세먼지농도 계산하기*/
void calcDustDensity() {
    duration = pulseIn(DUST_SENSER, LOW);
    lowpulseoccupancy = lowpulseoccupancy+duration;

    if ((millis()-starttime) > sampletime_ms){
        ratio = lowpulseoccupancy/(sampletime_ms*10.0);  // Integer percentage 0=>100
        concentration = 1.1*pow(ratio,3)-3.8*pow(ratio,2)+520*ratio+0.62;
        concentration_ugm3 = pm25pcs2ugm3 (concentration);
        /*
        lowpulseoccupancy: LPO시간, ~초간 측정된 총 입자 수 를 나타냅니다.
        ratio: ratio의 비율은 LPO에 따라 반영되며 concentration의 값에 변화를 줍니다. 초간 측정된 비(비율)를 나타냅니다
        concentration: 실질적인 미세먼지 값을 나타냅니다.
        concentration_ugm3: 작품설명표에 쓴 기준값 표 
        */ 
        lowpulseoccupancy = 0;
        dustDensity = int(concentration_ugm3);
        Serial.println(dustDensity);
        starttime = millis();
    }
}

float pm25pcs2ugm3 (float concentration_pcs){
  double pi = 3.14159;
  double density = 1.65 * pow (10, 12);
  double r25 = 0.44 * pow (10, -6);
  double vol25 = (4/3) * pi * pow (r25, 3);
  double mass25 = density * vol25;
  double K = 3531.5;

  return (concentration_pcs) * K * mass25;
}


void writeRGB(bool R, bool G, bool B) {
  digitalWrite(O_RGB_R, R);
  digitalWrite(O_RGB_G, G);
  digitalWrite(O_RGB_B, B);
}

void setup() {
  initPin();
  initLcd();
  //RGB LED를 보라색(빨강+파랑)으로 출력합니다.
  delay(2000);
  writeRGB(HIGH, LOW, HIGH);
}

void loop() {
  calcDustDensity();
  printLcd();
  //미세먼지 농도가 36(나쁨) 이상이면 펌프가 동작되고
  if (dustDensity > 36) {
    delay(2000);
    lcd.clear();
    lcd.noBacklight();
    for (int i = 0; i < 230; i++) {
      analogWrite(O_PUMP_A, i);
      delay(5);
    }
    delay(On_Time);
    analogWrite(O_PUMP_A, 0);
    analogWrite(O_PUMP_B, 0);
    delay(100);
   // 36보다 이하이면, 즉 보통 or 좋음이면 펌프를 멈춘다. 
  } else {
    analogWrite(O_PUMP_A, 0);
    analogWrite(O_PUMP_B, 0);
    delay(1000);
  }
}
```
