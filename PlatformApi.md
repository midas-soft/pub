# <a name="interactionDiagram"></a>Interaction diagram
![Untitled Diagram (4)](https://user-images.githubusercontent.com/4074799/129939753-acc9ce7d-5b82-4d46-b974-21e2dc1d7f3a.png)

# เกมที่เปิดให้บริการ
- ```ID 0``` = Rising fortune

# <a name="platformApi"></a>Platform Integration Manual
## Registration
- ก่อนที่ผู้เล่นจะเริ่มเล่นเกมได้ platform จะต้องลงทะเบียนผู้เล่นรายนั้นกับ GSlot ก่อน โดยใช้ endpoint ```/registerUser``` ดังจะกล่าวต่อไป

## Authentication
หลังจากที่ผู้เล่นเลือกเกมแล้ว platform ต้อง redirect มาที่ Url ด้านล่างนี้
```
http://13.212.150.168/build/web-mobile/
?game=<GAME_ID>
&platform=<PLATFORM_ID>
&token=<AUTH_TOKEN>
```
- ```GAME_ID``` = เกมที่ผู้เล่นต้องการจะเข้า
- ```PLATFORM_ID``` = platform id ที่ผู้เล่นสังกัดอยู่ (platform id จะ assign โดย GSlot)
- ```AUTH_TOKEN``` = authentication token ของผู้เล่นนั้น

### Auth Token
platform จะต้อง generate token ซึ่งมี payload ตาม structure ด้านล่างนี้
```
{
    username: string,
    credit: string,
}
```
- ```username``` = username ของผู้ใช้ที่ลงทะเบียนกับทาง platform และได้ลงทะเบียนกับทาง GSlot แล้ว
- ```credit``` = credit ของผู้เล่นในขณะที่ generate token

platform จะต้อง generate token โดยใช้ secret ตามที่ได้รับ assign จาก GSlot

## Endpoints
platform สามารถติดต่อกับ GSlot ได้ผ่านทาง HTTP protocol ทาง URL ```http://13.212.150.168:8082/```
### Basics
ทุกๆ request ต้องระบุข้อมูลดังนี้
```
{
    metadata: {
        timestamp: number,
    },
    apiKey: string,
}
```
- ```metadata.timestamp``` = Unix timestamp in millisec เมื่อเวลาที่ส่ง request นี้ออกมา
- ```apiKey``` = API key ของ platform ที่ได้รับ assign จาก GSlot

**Status code** จะเป็น ```200 OK``` ถ้า request success และจะเป็น ```500 Internal Server Error``` ถ้าเกิด error ขึ้น

### registerUser
```POST /registerUser```
- ใช้สำหรับการลงทะเบียนผู้เล่นจาก platform เข้ามาในสารบบของ GSlot ใช้เมื่อเข้าเกมของ GSlot ครั้งแรก
- request นี้เป็น idempotent สามารถ request หลายๆ ครั้งได้ โดยมีผลเสมือนกับว่า request ครั้งเดียว

**Request:**
```
{
    metadata: {
        timestamp: number,
    },
    apiKey: string,
    username: string,
    jackpotChannel: number
}
```
- ```username``` = username ของ user ที่ลงทะเบียนไว้กับทาง platform -> username นี้จะถูกใช้เมื่อ GSlot ติดต่อกับ platform
  - ```username``` จำกัดความยาวที่ 5-20 ตัวอักษร และมีได้แค่สัญลักษณ์ ```A-Z```, ```a-z```, ```0-9```, ```_```
- ```jackpotChannel``` = channel ที่ platform ต้องการให้ user คนนั้นสังกัดอยู่
  - มูลค่าสะสมของ jackpot และอัตราจ่าย จะแยกกันในแต่ละ channel

**Response:**
```
{
    success?: boolean,
    error?: string,
}
```
- ```success``` = เป็น ```true``` เมื่อลงทะเบียนสำเร็จ, field นี้จะไม่ปรากฎ ถ้ามี error
- ```error``` = ระบุ error code (ถ้ามี)

**Errors:**
- ```INVALID_PLATFORM_ID``` = ระบุ API key ของ platform ไม่ถูกต้อง
- ```INVALID_USERNAME_PATTERN``` = pattern ของ username ไม่ถูกต้อง
- ```INVALID_CHANNEL``` = ระบุ jackpot channel ไม่ถูกต้อง

### getUnfinishedTransactions
```POST /getUnfinishedTransactions```
- ใช้สำหรับถาม GSlot ว่า มี transaction ใดที่ยังติดต่อกับ platform ไม่ได้บ้าง

**Request:**
```
{
    metadata: {
        timestamp: number,
    },
    apiKey: string,
}
```

**Response:**
```
{
    error?: string,
    [betToken: string]: { //object ย่อย สามารถมีได้หลายรายการ
        type: 'settle' | 'cancel'
    	amount: string,
    	username: string,
    	gameId: number,
    	startTime: number,
    	actionType: number,
    }
}
```
- ```error``` = error (ถ้ามี)
- ```betToken``` = unique identifier ของ transaction
- ```type``` = เป็น ```settle``` ถ้าเป็นการสรุปผลการเดิมพันแพ้หรือชนะ, เป็น ```cancel``` ถ้าเป็นการยกเลิกการเดิมพัน
- ```amount``` = จำนวนเงินที่ชนะ (ถ้า ```type = cancel``` หรือการเล่นครั้งนั้นแพ้ แล้ว ```amount``` จะเป็น ```0``` เสมอ)
- ```username``` = username ของผู้เล่น
- ```gameId``` = ID ของเกมที่ผู้เล่นกำลังเล่น
- ```startTime``` = Unix timestamp in millisec ที่ผู้เล่นทำ action นี้
- ```actionType``` = ชนิดของการเล่น
  - ```1``` = Spin
  - ```2``` = Gamble

**Errors:**
- ```INVALID_PLATFORM_ID```

### finishTransaction
```POST /finishTransaction```
- ใช้สำหรับแจ้งให้ GSlot ทราบว่า transaction ที่ระบุ platform รับทราบแล้ว
- เป็น idempotent endpoint -> สามารถส่ง request นี้ซ้ำๆ โดยระบุ betToken เดิมได้ โดยจะมีผลเสมือนกับ request ครั้งเดียว

**Request:**
```
{
    metadata: {
        timestamp: number,
    },
    apiKey: string,
    betToken: string,
}
```
- ```betToken``` = unique identifier ของ transaction

**Response:**
```
{
    success?: boolean,
    error?: string,
}
```
- ```success``` = เป็น ```true``` เมื่อทำสำเร็จ, field นี้จะไม่ปรากฎ ถ้ามี error
  - ```success = true``` แม้ว่าจะเคยส่ง request ที่ระบุ ```betToken``` เดิมที่เคยทำสำเร็จไปแล้ว
- ```error``` = ระบุ error code (ถ้ามี)

**Error:**
- ```INVALID_PLATFORM_ID```
- ```ACTION_NOT_FOUND``` = ไม่พบ transaction ที่ระบุ

## Platform endpoints
Platform ที่ต้องการจะเชื่อมต่อกับ GSlot จะต้องเปิด endpoints ดังต่อไปนี้ ไว้สำหรับให้ GSlot หักเงิน รายงานผลการเล่น และสอบถามยอดเครดิตคงเหลือ และต้องมี structure ตามที่กำหนด และต้องรายงาน error ตามที่กำหนด ดังต่อไปนี้
- ก่อนที่จะเชื่อมต่อกับ GSlot ทั้ง platform และ GSlot จะต้องตกลง ```agentKey``` กันก่อน เพื่อให้ platform สามารถยืนยันตัวตนผู้ส่ง request ได้ว่าส่งมาจาก GSlot จริงๆ
### POST /getUserBalance
- ใช้สำหรับสอบถามยอดเครดิตปัจจุบันของผู้เล่น

**Request:**
```
{
    timestamp: number,
    username: string,
    agentKey: string,
    gameType: 'SLOT',
}
```
- ```timestamp``` = timestamp in milliseconds ณ เวลาที่ส่ง request
- ```username``` = username ของผู้เล่นที่ลงทะเบียนไว้กับ platform
- ```agentKey``` = token สำหรับใช้ยืนยันตัวตนว่าส่งมาจาก GSlot จริงๆ ซึ่งต้องตกลงกันกับ platform ก่อน
- ```gameType``` = ชนิดของเกม เป็น ```"SLOT"``` เสมอ

**Response:**
```
{
    currentCredit?: string,
    error?: string,
}
```
- ```currentCredit``` = จำนวนเครดิตปัจจุบันของผู้เล่น
- ```error``` = error code

**Errors:**
- ```INVALID_AGENT_KEY``` = agentKey ไม่ถูกต้อง ไม่ตรงตามที่ได้ตกลงกันไว้
- ```INVALID_USERNAME``` = username ไม่ถูกต้อง ไม่มีอยู่ในสารบบของ platform


*สำหรับ endpoint อื่นๆ จะมีบาง field และ errors ที่มีความหมายเช่นเดียวกันกับ endpoint นี้ ดังนั้นจะไม่กล่าวซ้ำอีก*

### PATCH /placeBet
- ใช้สำหรับตัดเงินของผู้เล่นมาใช่ในการเล่นเกม; เมื่อ platform ได้รับ request นี้ จะต้องตัดเครดิตออกจากบัญชีของผู้เล่น

**Request:**
```
{
    timestamp: number,
    username: string,
    agentKey: string,
    gameType: 'SLOT',
    gameId: number,
    actionType: number,
    amount: string,
    token: string
}
```
- ```gameId``` = ID ของเกมที่ผู้เล่นกำลังเล่น (เช่น ```0``` = Rising Fortune)
- ```actionType``` = ชนิดของการกระทำในเกมของผู้เล่นรายนี้ ได้แก่ ```1``` = Spin, ```2``` = Gamble และบางเกมอาจมี action เพิ่มเติมจากนี้
- ```amount``` = จำนวน credit ที่ต้องการถอนออกมาใช้ในการเล่น
- ```token``` = unique identifier ของ action นี้ ซึ่งจะใช้ในการอ้างอิงในการส่ง ```PATCH /playerWin```, ```PATCH /playerLost``` และ ```PATCH /cancelBet``` ต่อไป

**Response:**
```
{
    currentCredit?: string,
    error?: string,
}
```
- ```currentCredit``` = จำนวนเครดิตปัจจุบันของผู้เล่น หลังจากที่ตัดเงิน
- ```error``` = error code

**Errors:**
- ```INVALID_AGENT_KEY```
- ```INVALID_USERNAME```
- ```INSUFFICIENT_FUND``` = เครดิตในบัญชีของ platform ไม่เพียงพอ

### PATCH /playerWin
- ใช้สำหรับรายงานผลการเล่นชนะของผู้เล่น และฝากเงินเข้าบัญชีของผู้เล่นบน platform
- โดยปกติจะ request ตามหลัง ```/placeBet``` แต่ในกรณีที่ผู้เล่นได้ free spin (ไม่มีการตัดเงินของผู้เล่น) จะไม่มีการ request ```/placeBet``` นำมาก่อน

**Request:**
```
{
    timestamp: number,
    username: string,
    agentKey: string,
    gameType: 'SLOT',
    gameId: number,
    actionType: number,
    amount: string,
    token: string
}
```
- ```amount``` = จำนวนเครดิตที่ผู้เล่นเล่นชนะ
- ```token``` = unique identifier ของ action ที่ผู้เล่นเลือกเล่น ถ้าก่อนหน้านี้มีการส่ง ```/placeBet``` แล้ว ```token``` จะตรงกัน

**Response:**
```
{
    currentCredit?: string,
    error?: string,
}
```
- ```currentCredit``` = จำนวนเครดิตปัจจุบันของผู้เล่น หลังจากที่ได้รับเงิน

**Errors:**
- ```INVALID_AGENT_KEY```
- ```INVALID_USERNAME```

### PATCH /playerLost
- ใช้สำหรับรายงานผลการ<ins>เล่นแพ้</ins>ของผู้เล่น และฝากเงินเข้าบัญชีของผู้เล่นบน platform
- โดยปกติจะ request ตามหลัง ```/placeBet``` แต่ในกรณีที่ผู้เล่นได้ free spin (ไม่มีการตัดเงินของผู้เล่น) จะไม่มีการ request ```/placeBet``` นำมาก่อน

**Request:**
```
{
    timestamp: number,
    username: string,
    agentKey: string,
    gameType: 'SLOT',
    gameId: number,
    actionType: number,
    amount: '0',
    token: string
}
```
- ```token``` = unique identifier ของ action ที่ผู้เล่นเลือกเล่น ถ้าก่อนหน้านี้มีการส่ง ```/placeBet``` แล้ว ```token``` จะตรงกัน

**Response:**
```
{
    currentCredit?: string,
    error?: string,
}
```

**Errors:**
- ```INVALID_AGENT_KEY```
- ```INVALID_USERNAME```

### PATCH /cancelBet
- ใช้สำหรับการยกเลิกการ ```/placeBet``` ที่ได้กระทำไปแล้ว; เมื่อ platform ได้รับ request นี้ ให้คืนเครดิตที่ตัดออกไปจากการ ```/placeBet``` ครั้งก่อนหน้า
- จะส่งไปยัง platform เมื่อเกิด error ในระหว่างการติดต่อกับ platform หรือ error ภายในของตัว GSlot เอง
**Request:**
```
{
    timestamp: number,
    username: string,
    agentKey: string,
    gameType: 'SLOT',
    gameId: number,
    actionType: number,
    amount: '0',
    token: string
}
```
- ```token``` = unique identifier ของ action ซึ่งจะตรงกันกับ ```token``` ที่ได้ส่งไปยัง endpoint ```/placeBet``` ก่อนหน้านี้

**Response:**
```
{
    currentCredit?: string,
    error?: string,
}
```
- ```currentCredit``` = จำนวนเครดิต หลังจากที่ผู้เล่นได้รับเงินคืน

**Errors:**
- ```INVALID_AGENT_KEY```
- ```INVALID_USERNAME```

## Transaction error handling
- หาก GSlot ไม่สามารถ request ```/placeBet``` ไปยัง platform ได้สำเร็จ ภายใน 10 วินาที หลังจากที่ผู้เล่นเริ่มเล่น ทาง GSlot จะยกเลิกการเล่นตรั้งนี้ และส่ง ```/cancelBet``` ไปยัง platform จนกว่า platform จะได้รับ request เป็นเวลา 3 ชั่วโมง หลังจากนั้นถ้ายังติดต่อ platform ไม่ได้ GSlot จะหยุดส่ง request
- หาก GSlot ไม่สามารถ request ```/playerWin``` หรือ ```/playerLost``` ไปยัง platform ได้สำเร็จ ภายในเวลา 3 ชั่วโมง หลังจากที่ผู้เล่นเล่นเสร็จแล้ว GSlot จะหยุดส่ง request
- **Transaction ใดที่ไม่สามารถติดต่อได้ภายใน 3 ชั่วโมง จะไม่มีการติดต่อไปที่ platform อีก; platform จะต้องเป็นผู้ query transaction ที่ยังตกค้าง โดยใช้ endpoint ```POST /getUnfinishedTransactions``` พร้อมทั้งยืนยันความสมบูรณ์ของ transaction ที่ตกค้างด้วย endpoint ```POST /finishTransaction``` ตามที่ได้กล่าวไปแล้วข้างต้น**
- ในขณะที่ยังมี transaction ที่ยังไม่ได้ยืนยันความสมบูรณ์ ผู้เล่นจะไม่สามารถเล่นเกมนั้นต่อได้ จนกว่าจะได้รับการยืนยันจาก platform
