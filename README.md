# Arduino_Ile_Hafizali_Robot_Kol_Projesi
#include <Servo.h> 

Servo servo_0;
Servo servo_1;
Servo servo_2;
Servo servo_3;

int sensorPin0 = A4;    // göbek
int sensorPin1 = A5;    // gripper
int sensorPin2 = A6;    // yukarı aşağı
int sensorPin3 = A7;    // ileri geri

long previousMillis1 = 0;
long previousMillis2 = 0;
long previousMillis3 = 0;
long previousMillis4 = 0;
long previousMicros = 0;
unsigned long currentMillis = millis();
unsigned long currentMicros = micros();


int Delay[7] = {0,0,1,1,1,1,1}; // saniye olarak ne kadar gecikme olacağı
int SensVal[4]; // pot değerleri
float dif[4], ist[4], sol[4],  dir[4]; // anlık pozisyon ile hafızadaki pozisyon arası fark
int joint0[180];// servolar için diziler
int joint1[180];
int joint2[180];
int joint3[180];
int top = 179; 

boolean playmode = false, Step = false;

void setup()
{
  
  pinMode(4, INPUT);  
  pinMode(6, INPUT);
  pinMode(13, OUTPUT); 
  digitalWrite(13, HIGH);   
  servo_0.attach(3); 
  servo_1.attach(5);
  servo_2.attach(9);
  servo_3.attach(10);
  Serial.begin(9600); 
  Serial.println("mini robot kol hazır");     

  digitalWrite(13, LOW);
}

void loop() 
{
  currentMillis = millis(); // millis fonk zamanlama hesapları için kullanılacak
  currentMicros = micros();
  
  // Butonun değerini oku  
  Button();
  
  if(!playmode) // oynatma modunda değilsek
  {        
    if(currentMillis - previousMillis1 > 25) // 25ms de bir modu güncelle
    {
      if (arrayStep < top) 
      {
        previousMillis1 = currentMillis; //eski zamanı güncelle
        readPot(); // potlardan değerleri al
        mapping(); // servolar için ms ye map et
        move_servo(); // servoları yeni pozisyonlarına hareket ettir
          
      } 
    } 
  } 
   
  else if(playmode) 
  {
    if (Step) 
    {
      digitalWrite(13, HIGH); 
      if (arrayStep < arrayMax) 
      {
        arrayStep += 1;
        Read();
        calculate(); 
        Step = 0;
        digitalWrite(13, LOW);  
      }
      else
      {
        arrayStep = 0;
        calc_pause(); 
        countverz = 0; 
        while(countverz < verz)
        { 
          countverz += 1;
          calc_pause();
          digitalWrite(13, HIGH); delay(25);   
          digitalWrite(13, LOW); delay(975); 
        }
      }
      //Serial.println(arrayStep);
    }
    else // servoları hareket ettiriyoruz
    {
      if (currentMicros - previousMicros > time) // tek bir mikrostep adımı yaptık
      { // 
        previousMicros = currentMicros;
        play_servo(); 
      }
    }
  }

// Donanımsal durdurma pini PIN 4
    while (digitalRead(4) == false) //buton değeri açık devre ise çalışmayı beklemeye alıyor. 
      { 
        digitalWrite(13, HIGH); delay(500);   
        digitalWrite(13, LOW); delay(500);
      }
//  Kontrol için serial monitöre yazdırma
  
    /* if(currentMillis - previousMillis2 > 5000)
    { 
      previousMillis2 = currentMillis;
      /*count0 = 0;
      while(count0 < 4)
      {
        int val = SensVal[count0];
      // val = map(val, 142, 888, 0, 180);
        Serial.println(val);
        //Serial.println("test");
        count0 += 1;
      }
      Serial.println(playmode); 
      Serial.println(arrayStep);    
      Serial.println(arrayMax);    
      Serial.println(" ");    
    } */
}

//  alt fonksiyonlar
void calc_pause() 
{
    readPot();
    temp = SensVal[3];
    if (temp < 0) temp = 0;
    temp = map(temp, 0, 680, 0 ,5); 
    verz = Delay[temp]; 
    Serial.print(temp);
          Serial.print(" ");
          Serial.print(verz);
          Serial.print(" ");
          Serial.println(countverz);
}

void readPot() 
{
   SensVal[0] = analogRead(sensorPin0); //SensVal[0] += -10; //
   SensVal[1] = analogRead(sensorPin1); //SensVal[1] += 280; // 
   SensVal[2] = analogRead(sensorPin2); //SensVal[2] += -50; // 
   SensVal[3] = analogRead(sensorPin3); // SensVal[3] += 0;//
  Serial.print(SensVal[2]);Serial.print(" "); 
}
void mapping() 
{
  ist[0] = map(SensVal[0], 150, 900, 600, 2400);//  
  ist[1] = map(SensVal[1], 1000, 100, 550, 2400);// 
  ist[2] = map(SensVal[2], 120, 860, 400, 2500);// 
  ist[3] = map(SensVal[3], 1023, 0, 500, 2500);// 
 Serial.println(ist[2]); 
}
void record()
{
    joint0[arrayStep] = ist[0]; 
    joint1[arrayStep] = ist[1];
    joint2[arrayStep] = ist[2];
    joint3[arrayStep] = ist[3];
}
void Read()
{
    sol[0] = joint0[arrayStep];
    sol[1] = joint1[arrayStep];
    sol[2] = joint2[arrayStep];
    sol[3] = joint3[arrayStep];
}
void move_servo()
{       
  servo_0.writeMicroseconds(ist[3]); 
  servo_1.writeMicroseconds(ist[2]); 
  servo_2.writeMicroseconds(ist[0]); 
  servo_3.writeMicroseconds(ist[1]); 
}

//  her bir adımın hesaplanması
void calculate()
{
      // hareket mesafesi
      dif[0] = abs(ist[0]-sol[0]);
      dif[1] = abs(ist[1]-sol[1]);
      dif[2] = abs(ist[2]-sol[2]);
      dif[3] = abs(ist[3]-sol[3]);

      //4 servo içindeki en büyük hareket edenin bulunması
      stepsMax = max(dif[0],dif[1]);
      stepsMax = max(stepsMax,dif[2]);
      stepsMax = max(stepsMax,dif[3]);
     
      
      //Serial.println(stepsMax); 
      
      if (stepsMax < 500) // eğer en uzun hareket <500 ise delay 1200 diğer durumlarda delay 600ms
        del = 1200;
      else
        del = 600;
      
        if (sol[0] < ist[0]) dir[0] = 0-dif[0]/stepsMax; else dir[0] = dif[0]/stepsMax;
      if (sol[1] < ist[1]) dir[1] = 0-dif[1]/stepsMax; else dir[1] = dif[1]/stepsMax;
      if (sol[2] < ist[2]) dir[2] = 0-dif[2]/stepsMax; else dir[2] = dif[2]/stepsMax;
      if (sol[3] < ist[3]) dir[3] = 0-dif[3]/stepsMax; else dir[3] = dif[3]/stepsMax;
        //Serial.println(dir4); 

}
void play_servo()
{
    steps += 1;
    if (steps < stepsMax) 
    {
      //time = del*5;
      if(steps == 20) time = del*4;         
      else if(steps == 40) time = del*3;   
      else if(steps == 80) time = del*2;    
      else if(steps == 100) time = del-1;   
      
      if(steps == stepsMax-200) time = del*2;       
      else if(steps == stepsMax-80) time = del*3;
      else if(steps == stepsMax-40) time = del*4;
      else if(steps == stepsMax-20) time = del*5;
      
      ist[0] += dir[0]; 
      ist[1] += dir[1];
      ist[2] += dir[2];
      ist[3] += dir[3];

      servo_0.writeMicroseconds(ist[3]); 
      servo_1.writeMicroseconds(ist[2]); 
      servo_2.writeMicroseconds(ist[0]); 
      servo_3.writeMicroseconds(ist[1]); 
    }
    else
    {
      Step = 1; 
      steps = 0; 
    }
}

void data_out() 
{
  int i = 0;
  while(i < arrayMax)
  {
    digitalWrite(13, HIGH);
    i += 1;
    Serial.print(joint0[i]); Serial.print(", ");
  }
  Serial.println("Joint0");
  i = 0;
  while(i < arrayMax)
  {
    digitalWrite(13, HIGH);
    i += 1;
    Serial.print(joint1[i]); Serial.print(", ");
  }
  Serial.println("Joint1");
  i = 0;
  while(i < arrayMax)
  {
    digitalWrite(13, HIGH);
    i += 1;
    Serial.print(joint2[i]); Serial.print(", ");
  }
  Serial.println("Joint2");
  i = 0;
  while(i < arrayMax)
  {
    digitalWrite(13, HIGH);
    i += 1;
    Serial.print(joint3[i]); Serial.print(", ");
  }
  Serial.println("Joint3");
}

void Button() 
{
  if (digitalRead(6) == false)
  {
    delay(1);
    if (digitalRead(6) == true) 
    {
      if (Taster == 0)
      {
        Taster = 1; 
        previousMillis3 = currentMillis; 
        //Serial.print("Status Record "); Serial.println(Taster); 
      }
      else if ((Taster == 1) && (currentMillis - previousMillis3 < 250))
      {
        Taster = 2;
        //Serial.println(Taster); 
      }
      /*else if ((Taster == 2) && (currentMillis - previousMillis3 < 500))
      {
        Taster = 3;
        Serial.println(Taster); 
      }*/
    }
  }
    
    if ((Taster == 1) && (currentMillis - previousMillis3 > 1000)) 
    {
      arrayStep += 1;
      arrayMax = arrayStep;
      record();
      Taster = 0;
      playmode = false;
      Serial.print("Record Step: "); Serial.println(arrayStep);
      digitalWrite(13, HIGH);
      delay(100);
      digitalWrite(13, LOW);
    }
    else if (Taster == 2)
    {
      arrayStep = 0;
      playmode = true;
      Taster = 0;
      Step = 1;
      Serial.println("playmode ");
      data_out();
      delay(250);   
      digitalWrite(13, LOW);    
    }
    /*if (Taster == 3)
    {
      Taster = 0;
      Serial.println("Clear ");
    }*/
    if (currentMillis - previousMillis3 > 2000) 
    {
      Taster = 0;
     
    }
}
