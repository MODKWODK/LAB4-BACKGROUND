# SOC LAB4 背景知識
lab4 在SOC上驗證 因為各IP仍是由軟體控制所以稱之為 **"軟體驗證"**
參考lab4檔案 testbench/counter_la_fir/counter_la_fir_tb.v
![image](https://hackmd.io/_uploads/rJigZde86.png)
1. Carvel uut : chip design call
2. Spiflash 
CPU需要執行軟體CODE(firmware)，firmware(counter_la.hex)放入spiflash後透過spiflash的io pin 跟cpu溝通
3. Initial begin
開始執行CPU，CPU透過SPILASH IO PIN 讀取指令(firmware)後與user_project硬體溝通
4. checkbits 
CPU執行後結果會由mprj_io[37:0]output，而後由testbench做比對，確認結果是否正確

## Firmware Code in this Project

如何將firmware Code .c .h檔案轉成 verilog 可以使用的檔案型式
每個資料夾內都有執行檔，如makefile、run_sim 等等，以run_sim為例
以下段落分別將firmware Code(.c)轉換為.hex(verilog可應用的檔案)

```
riscv32-unknown-elf-gcc -Wl,--no-warn-rwx-segments -g \
	--save-temps \
	-Xlinker -Map=output.map \
	-I../../firmware \
	-march=rv32i -mabi=ilp32 -D__vexriscv__ \
	-Wl,-Bstatic,-T,../../firmware/sections.lds,--strip-discarded \
	-ffreestanding -nostartfiles -o counter_la_fir.elf ../../firmware/crt0_vex.S ../../firmware/isr.c fir.c counter_la_fir.c
# -nostartfiles	
riscv32-unknown-elf-objcopy -O verilog counter_la_fir.elf counter_la_fir.hex
riscv32-unknown-elf-objdump -D counter_la_fir.elf > counter_la_fir.out

# to fix flash base address
sed -ie 's/@10/@00/g' counter_la_fir.hex
```
## 什麼是RISC-V
CPU 的指令集可簡單分為 2 種
RISC（reduced instruction set computer, 精簡指令集電腦）和 CISC（Complex Instruction Set Computer, 複雜指令集電腦）

   | 比較 |    RISC    |   CISC    |
   | ---- |:----------:|:---------:|
   | 耗電 |     低     |    高     |
   | 面積 |     小     |    大       |
   | 性能 |     弱     |    強       |
   | 應用 | ARM,RISV-V | Inter x86   |
* RISC 架構簡單，因此面積小且功耗低。 因為硬體每個功能都需要事先寫好刻成一個電路，所以指令格式愈一致，需要的電路就愈簡單，面積也就愈小，功耗也會愈低，且資料從進去 CPU 到出來的時間也會較短，所以每秒可以做的運算也愈多，而 RISC-V 就是設計更為精簡的 RISC 指令集。因為現在 CPU 已經算得很快了，要讓它更快的成本要比以前更多，為了讓同一代 CPU 能用更久，RISC-V是一個很好的選擇
參考 https://weikaiwei.com/riscv/riscv-1/  
  
## FIR 架構
lab3已經使用過此架構
參考 https://hackmd.io/S9EMDwwtS-qvOTiwAhrXWw
![image](https://hackmd.io/_uploads/S1njldeLT.png)
* Data_Width 32
* Tape_Num 11
* Data_Nun XX
* Communication Interface
[1] data_in (Xn): stream
[2] data_out(Yn): stream
[3] coef[Tape_Num-1:0]: axi-lite
[4] length: stream
[5] ap_start: axi-lite
[6] ap_done: axi-lite


## Interface - LA 架構
User project 是被包含在user_project wrapper中的其中一個design，
可以連接到的interface 都提供在User Project Wrapper.V
* ### User Project Wrapper
參考檔案位置: rtl/user/User Project Wrapper.V
![image](https://hackmd.io/_uploads/S1fXOqU8p.png)
#### 1. Wishbond  
   user_project 偵測到 "cycle & strb"都等於1，user_project 對wishbond響應
#### 2. Logic Analyzer[127:0]
   user_project 接受 caravel-soc "la model"資料  
   注意: user_project只有看到"**la_oenb(la_output enable) 為 1**"，才可以從**la_data_out發送資料給la_module**
#### 3. MPRJ_IO[37:0]  
user_project 接受 caravel-soc "mprj_io"資料，同時user_project也可以透過user_project wrapper取得user_clock2 & user_irq  
#### 4. User_Clock 
#### 5. User_irq 


## User Project - Counter Design
### user_proj_example.counter 
參考檔案位置: user/user_proj_example.counter.v
* #### interface
![image](https://hackmd.io/_uploads/Bk02q58Up.png)  
1. Wishbond  
2. Logic Analyzer[127:0]
3. MPRJ_IO[37:0]  
4. User_Clock 
5. User_irq(no used) 

* #### interface communcation & relationship
![image](https://hackmd.io/_uploads/SJEDRqIUp.png) 
1. counter clk   根據la_oenb[64] 決定clk 
2. counter rst   根據la_oenb[65] 決定rst  

* #### counter design
![image](https://hackmd.io/_uploads/r1p-esI8p.png)

#### [1]ready(wbs_ack_o)
counter ready 連結到wishbond ack => 因此counter 可以透過 ready 響應wishbond transation 
#### [2]valid(valid) 
wishbond cycle  && wishbond strb，當為1時counter 對wishbond響應
#### [3]rdata(wbs_dat_o)  
cpu 打送wishbond read transition 取得count value
#### [4]wdata(wbs_data_i)
cpu 打送wishbond write transition 取得count value
#### [5]wstrb 
若此時為read transition => 4'b0000 若為write transition => wbs_sel_i
#### [6].la_write 
valid & la_oeb  當沒有valid情況下，la_oeb的反向
#### [7].la_input
la_data_in[63:32] CPU訪問la_module將count value 寫入counter
#### [8]count
output count value, then count value assign to la_data_out[31:0]
cpu 可以透過訪問la_module取得counter 的count value
counter 的count value 也被assign 到io_out 因此可以透過訪問MPRJ_IO get count value
## Counter 使用LA interface 
![image](https://hackmd.io/_uploads/B1iTZ2I8T.png)
1. CPU透過wishbone bus 提取firmwave Code,而後firmwave code 透過wishbond操作LA_module 控制counter_design
2. 在testbench 中檢測MPRJ-IO output 決定是否測試完成 ex start:0XAB40 receive data: 0xAB41 finish test: 0xAB51 
## LAB4 實驗步驟
LAB4基礎知識   
https://hackmd.io/@861r6vBbQbu3FLGNe3SkyQ/BknfcXy86    
https://github.com/MODKWODK/LAB4-BACKGROUND  
LAB4-0  
https://hackmd.io/@861r6vBbQbu3FLGNe3SkyQ/SkC0vJ1UT  
https://github.com/MODKWODK/LAB4-0-caravel-SOC-simulation-  
LAB4-1  
https://hackmd.io/@861r6vBbQbu3FLGNe3SkyQ/HybFQWkIT  
https://github.com/MODKWODK/LAB4-1-Firmware-User-project  
LAB4-2  
https://hackmd.io/@861r6vBbQbu3FLGNe3SkyQ/BJXPryyIp  
https://github.com/MODKWODK/LAB4-2-MIX-LAB3-FIR-LAB4-1-WITH-WISHBOND 

## 參考資料
#### 1. user_design interface - teatbench & firmware
> https://www.youtube.com/watch?v=0neIx5DOK1g&list=PL5CoDA0gtOHVeF4FkotwDY-buLVIoeGzf
#### 2. GPIO & SPI & MMIO(Memory-mapped IO address)
> https://www.youtube.com/watch?v=Vw3TGc-YV8E&list=PL5CoDA0gtOHVeF4FkotwDY-buLVIoeGzf&index=3
> 
spi_enable == 1 <-> mprj_io[32:35]
pass thru -> connect flash可以直接透過mprjio控制flash
* 將mprj_io[1:4]call spi-flash command，connect flash[1:4]
* 將mprj_io[1:4]<->mprj_io[8:11](user mode flash)
* docs/other/memory_map.txt 標示 memory map address
* GPIO:gpio將data input mprj_io做配置決定是做user_project signals or management signal 
檔案位置: rtl/header/user_define.v
<1>1bit(management soc wrapper) 
<2>mprj_io[37:0]
![image](https://hackmd.io/_uploads/BJAI-POD6.png)
#### 3.soc的整體架構
> https://www.youtube.com/watch?v=MBXKNe3kahw&list=PLTA_T2FLzYNDubjvfj4OqfmsxQiUB4CrD
## LAB4 實驗步驟
LAB4基礎知識   
https://hackmd.io/@861r6vBbQbu3FLGNe3SkyQ/SkC0vJ1UT  
https://github.com/MODKWODK/LAB4-BACKGROUND  
LAB4-0  
https://hackmd.io/@861r6vBbQbu3FLGNe3SkyQ/SkC0vJ1UT  
https://github.com/MODKWODK/LAB4-0-caravel-SOC-simulation-  
LAB4-1  
https://hackmd.io/@861r6vBbQbu3FLGNe3SkyQ/HybFQWkIT  
https://github.com/MODKWODK/LAB4-1-Firmware-User-project  
LAB4-2  
https://hackmd.io/@861r6vBbQbu3FLGNe3SkyQ/BJXPryyIp  
https://github.com/MODKWODK/LAB4-2-MIX-LAB3-FIR-LAB4-1-WITH-WISHBOND  
 














