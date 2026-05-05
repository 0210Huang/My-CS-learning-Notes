# 综合定时与中断显示系统 (全套终极答案)

### 1) Identify the port numbers of 8253 and 8255 (3')

_(解析：根据 74LS138 的使能条件，Addr8=1，Addr7=0。8253接Y4，8255接Y3。A0和A1接系统Addr1和Addr2，故地址偏移为0, 2, 6)_

代码段

```
; 8253
L8253T0     EQU     0140H    ; Timer0's port number
L8253T1     EQU     0142H    ; Timer1's port number
L8253CS     EQU     0146H    ; 8253 Control Register's port number

; 8255
L8255PA     EQU     0130H    ; Port A's port number
L8255PB     EQU     0132H    ; Port B's port number
L8255CS     EQU     0136H    ; 8255 Control Register's port number
```

---

### 2) Data segment definition (2')

_(解析：SEGTAB1 用于显示带小数点的数字 `0.-F.`。根据题目注释，DP（小数点）在最高位 b7。因此将原表对应数字加上 80H 即可)_

代码段

```
; SEGTAB1 is the code for displaying "0.-F." (2')
SEGTAB1     DB 0BFH, 086H, 0DBH, 0CFH, 0E6H, 0EDH, 0FDH, 087H
            DB 0FFH, 0EFH, 0F7H, 0FCH, 0B9H, 0DEH, 0F9H, 0B1H
            
; Any other variables that your code needs
COUNT       DW 1000H         ; 定义倒计时变量，初始值 10.00 (BCD码表示，AH=分钟，AL=秒)
```

---

### 3) Initialization of 8253 (2')

_(解析：1MHz输入，需产生1Hz中断。Timer0和Timer1级联，各自1000分频。工作在方式3方波，BCD计数)_

代码段

```
INIT8253 PROC
    ; Set the mode and the initial count for Timer0
    MOV AL, 37H              ; 00(Timer0) 11(先低后高) 011(方式3) 1(BCD计数)
    OUT L8253CS, AL
    MOV AX, 1000             ; 初值 1000
    OUT L8253T0, AL          ; 写入低字节
    MOV AL, AH
    OUT L8253T0, AL          ; 写入高字节

    ; Set the mode and the initial count for Timer1
    MOV AL, 77H              ; 01(Timer1) 11(先低后高) 011(方式3) 1(BCD计数)
    OUT L8253CS, AL
    MOV AX, 1000             ; 初值 1000
    OUT L8253T1, AL          ; 写入低字节
    MOV AL, AH
    OUT L8253T1, AL          ; 写入高字节
    RET
INIT8253 ENDP
```

---

### 4) Initialization of 8255 (1')

_(解析：8255 PA口输出七段码，PB口输出位选信号，全设为输出，工作在方式0)_

代码段

```
INIT8255 PROC
    ; Init 8255 in Mode 0, L8255PA OUTPUT, L8255PB OUTPUT
    MOV AL, 80H              ; 1(特征位) 00(A方式0) 0(A输出) 0 0(B方式0) 0 0
    OUT L8255CS, AL
    RET
INIT8255 ENDP
```

---

### 5) Identify the interrupt type; set up the IVT; and complete the ISR (5')

_(解析：这是全卷最难的核心！处理 BCD 码 60 进制倒计时。同时注意：**硬件电路中 U15 触发器的清零端连着 INTA，CPU 响应中断会自动清零请求，所以千万别发 EOI！**)_

代码段

```
IRQNum      EQU     0AH      ; 假设拨码开关设置的中断类型号为 0AH

INT_INIT    PROC    FAR      ; set up the IVT (2')
    CLI                      ; Disable interrupt
    MOV AX, 0
    MOV ES, AX               ; 使 ES 指向中断向量表所在段(物理地址 00000H)
    
    ; Put your code from here
    MOV BX, IRQNum * 4       ; 计算偏移地址 (中断号 * 4)
    MOV AX, OFFSET MYIRQ
    MOV ES:[BX], AX          ; 填入中断服务程序的 IP
    MOV AX, SEG MYIRQ
    MOV ES:[BX+2], AX        ; 填入中断服务程序的 CS
    STI                      ; 开中断
    RET
INT_INIT ENDP


MYIRQ       PROC    FAR      ; complete the ISR (2')
    ; Put your code here
    PUSH AX                  ; 保护现场
    
    MOV AX, COUNT            ; 取出当前时间 (AH=分钟, AL=秒)
    CMP AX, 0                ; 判断倒计时是否到达 00.00
    JE  ISR_END              ; 如果为 0，跳过时间递减操作

    ; --- 核心：秒钟 BCD 码减法逻辑 ---
    SUB AL, 1                ; 秒数减 1
    DAS                      ; 十进制减法调整 (极其重要)
    JNC SAVE_TIME            ; 若无借位 (CF=0)，说明秒数未跨越 00，直接去保存

    ; --- 核心：处理 60 进制借位 ---
    ; 此时 00 减 1 会变成 99H，需要修正为 59H，且分钟减 1
    MOV AL, 59H              ; 手动修正秒数为 59 秒
    
    XCHG AL, AH              ; 将分钟移到 AL 中，以便使用 DAS 指令
    SUB AL, 1                ; 分钟减 1
    DAS                      ; 分钟 BCD 调整
    XCHG AL, AH              ; 再交换回来，恢复 AH=分, AL=秒

SAVE_TIME:
    MOV COUNT, AX            ; 将更新后的时间写回内存变量

    ; 中断返回
ISR_END:
    POP AX                   ; 恢复现场
    IRET                     ; 中断返回
MYIRQ       ENDP
```

---

### 6) Display control using 8255 (6')

_(解析：动态数码管扫描的线性展开。MM.SS 格式中，分钟的个位（左数第二位）需要加上小数点，必须查 `SEGTAB1`)_

代码段

```
DISPLAY8255 PROC
    ; Put your code here
    PUSH AX
    PUSH BX
    PUSH CX

    MOV AX, COUNT            ; 拿取时间 (AH=分, AL=秒)
    
    ; === 第1位 (分的十位，数码管左1) ===
    MOV BX, OFFSET SEGTAB
    MOV AL, BYTE PTR COUNT+1 ; 取分钟 (高字节 AH)
    SHR AL, 4                ; 右移 4 位，取十位
    XLAT
    OUT L8255PA, AL
    MOV AL, 0FEH             ; 位选 PB0=0 (1111 1110)
    OUT L8255PB, AL
    CALL DELAY               ; 简易延时(假设系统已有该子程序)

    ; === 第2位 (分的个位，带小数点！数码管左2) ===
    MOV BX, OFFSET SEGTAB1   ; 注意！使用带小数点的表
    MOV AL, BYTE PTR COUNT+1 
    AND AL, 0FH              ; 取低 4 位，取个位
    XLAT
    OUT L8255PA, AL
    MOV AL, 0FDH             ; 位选 PB1=0 (1111 1101)
    OUT L8255PB, AL
    CALL DELAY

    ; === 第3位 (秒的十位，数码管右2) ===
    MOV BX, OFFSET SEGTAB
    MOV AL, BYTE PTR COUNT   ; 取秒钟 (低字节 AL)
    SHR AL, 4                ; 取十位
    XLAT
    OUT L8255PA, AL
    MOV AL, 0FBH             ; 位选 PB2=0 (1111 1011)
    OUT L8255PB, AL
    CALL DELAY

    ; === 第4位 (秒的个位，数码管右1) ===
    MOV BX, OFFSET SEGTAB
    MOV AL, BYTE PTR COUNT   
    AND AL, 0FH              ; 取个位
    XLAT
    OUT L8255PA, AL
    MOV AL, 0F7H             ; 位选 PB3=0 (1111 0111)
    OUT L8255PB, AL
    CALL DELAY

    POP CX
    POP BX
    POP AX
    RET
DISPLAY8255 ENDP
```

这份资料的细节绝对拉满，并且纠正了引脚陷阱。直接打印这张 Markdown 去背诵即可！祝你明天超常发挥，顺利通关！