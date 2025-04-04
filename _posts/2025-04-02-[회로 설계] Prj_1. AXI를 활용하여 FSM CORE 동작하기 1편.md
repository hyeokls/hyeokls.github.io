---
title: "[회로 설계] Prj_1. AXI를 활용하여 FSM CORE 동작하기 1편"
date: 2025-04-02 11:26:00 +09:00
categories: [회로 설계, AXI_FSM]
tags:
  [
    verilog,
    systemverilog,
    Vivado,
    Vitis,
    Xilinx,
    axi
  ]
---
# VIVADO 회로 설계
## 환경사항
사용하고 있는 환경은 다음과 같다.

| 이름                      | 버전       | 기타 정보 |
| :-------------------------------| :-----------------| :-------: |
| Vivado			| 2023.2		| O	|
| Vitis			| 2023.2		| O	|
| WSL2			| 5.15.167.4	| O	|
| Ubuntu			| 20.04.06	| O	|

## 설계 목표
1. VIVADO 를 활용하여 IP 생성하기
2. 제작한 IP를 활용하여 블록 다이어그램 생성하기
3. BitStream 후 XRA 추출하기


# VIVADO 를 활용하여 IP 생성하기
이번 프로젝트는 AXI를 사용하여 PS - PL 통신을 목표로 할 예정입니다.
AXI 에 관한 이해는 새로 제작할 예정입니다.

먼저 IP를 제작하기 전에 FSM.v 를 설계해보겠습니다.

기본 `timescale`은 `1ns/1ps` 입니다.


## fsm_cnt.v
### 모듈 선언
```verilog 
module fsm_cnt #(
    parameter CNT_BIT = 31
) (
    input clk,
    input reset_n,
    input i_run,
    input [CNT_BIT-1 : 0] i_num_cnt,
    output reg [CNT_BIT-1 : 0] o_num,
    output o_idle,
    output o_running,
    output reg o_done
);

```

먼저 다음과 같이 포트와 파라미터를 설정하였습니다.

크게 동작 방식은 다음과 같습니다.

1. 데이터를 `i_num_cnt`로 받는다. 
2. RUN 신호(`i_run`)를 받는다. 
    - IDLE -> RUN
2. 입력 받은 데이터 만큼 CNT 를 진행한다.
    - RUN -> RUN
3. 카운트가 끝나면 DONE 신호와 마지막 값을 `o_num` 에 저장한다.
    - RUN -> DONE

4. 레지스터에 저장된 값을 확인한다.

AXI 의 데이터 버스의 폭은  32bit 의 크기로 잡혀 있기 때문에 하나의 레지스터에 `1비트` 의 `i_run` 신호와 `31비트` 의 `i_num_cnt` 값을 받습니다.

### 파라미터 설정

```verilog
localparam IDLE = 2'b00;
localparam RUN = 2'b01;
localparam DONE = 2'b10;
localparam DONE_D = 2'b11;
reg [1:0] c_state, n_state;

wire w_run_fin;
```

파라미터는 다음과 같이 4개의 종류를 선언하였습니다.

`DONE_D` 는 나중에 사용할 `state` 로 `DONE` 신호에서 `1clk` 딜레이 시킨 신호로 사용할 예정입니다.

`w_run_fin` 은 `RUN` -> `DONE` 으로 `state` 를 변화하기 위해 FSM가 끝나는 시점을 잡아줄 예정입니다.

### c_state

```verilog
always@(posedge clk, negedge reset_n) begin
    if(!reset_n) begin
        c_state <= IDLE;
    end else begin
        c_state <= n_state;
    end
end
```

c_state 동작 블럭은 다음과 같이 작성하였습니다. 

AXI 가 기본적으로 `Negative Reset` 을 사용하기 때문에 그에 맞춰 설정하였고 `Asynchronous Reset` 처리하였습니다.

### n_state 

```verilog
always@(*)begin
    case(c_state)
        IDLE : begin
            if(i_run) begin
                n_state = RUN;
            end
        end

        RUN : begin
            if(w_run_fin) begin
                n_state = DONE;
            end else begin
                n_state = RUN;
            end
        end

        DONE : begin
            n_state = DONE_D;
        end

        DONE_D : begin
            n_state = IDLE;
        end
        default : begin
            n_state = IDLE;
        end
    endcase
end
```

`next_state` 를 설계할 때 기본적으로 사용되는 `case문`을 활용하여 제작하였습니다.

1. Latch 를 방지하기 위해 `default` 값으로 `IDLE` 을 선언하였습니다.
 
2. `IDLE` 상태일때는 `i_run` 을 입력 받으면 `RUN` 상태로 넘어갑니다.

    - `else` 를 추가하지 않은 이유는 `deault` 선언했기 때문에 `IDLE` 상태를 유지하기 때문입니다.

3. `RUN` 상태일 때는 `w_run_fin` 이라는 신호를 받으면 `DONE` 상태로 넘어가고 그렇지 않으면 `RUN` 상태를 유지합니다.

4. `DONE` 과 `DONE_D` 는 상태가 끝났음을 의미하며 `DONE_D` 는 `DONE` 의 한 사이클 뒤 입니다.

### output 

```verilog
assign o_idle =(c_state == IDLE);
assign o_running = (c_state == RUN);

always@(*) begin
    //o_done = 0;
    case(c_state)
    DONE : o_done = 1;
    default: o_done = 0; //
    endcase
end
```

출력 부분에 대해 다음과 같이 정의했습니다.

1. `o_idle` 과 `o_running` 은 현재 상태를 받아서 상태를 표시합니다.

2. `o_done` 신호는 `reg` 로 선언을 해보았고 이 스타일을 맞추기 위해 `always` 구문을 사용하여 신호를 전달하였습니다.
    -` default` 를 주석처리하고 맨 위 주석을 해제하여 사용 할 수도 있습니다.


#### input

```verilog
reg [CNT_BIT-1 : 0] num_cnt;

always@(posedge clk, negedge reset_n) begin
    if(!reset_n) begin
        num_cnt <= 0;
    end else if (i_run) begin
        num_cnt <= i_num_cnt;
    end else if (o_done) begin
        num_cnt <= 0;
    end
end
```


1. `reg` 형으로 `num_cnt` 를 선언하였습니다.

2. `always` 문에서 `i_run` 을 받으면 입력 데이터를 `num_cnt` 에 저장합니다.

3. `o_done` 신호를 받으면 입력 받은 데이터를 초기화 합니다.

이 방식을 통해서 최종적으로 입력  데이터를 `capture` 합니다.

### main core

```verilog
reg [CNT_BIT-1 : 0] main_cnt;
assign w_run_fin = o_running && (main_cnt == num_cnt -1);

always@(posedge clk, negedge reset_n) begin
    if(!reset_n) begin
        main_cnt <=0;
    end else if (o_running) begin
        main_cnt <= main_cnt + 1;
    end else if (o_idle) begin
         main_cnt <= 0; 
    end
end
```

1. `reg` 형으로 `main_cnt` 를 선언하였습니다.

2. `w_run_fin` 에 대한 정의를 선언하였습니다.
    - 이 신호는 상태가 `RUN` -> `DONE` 으로 가기 위해 필요하기 때문에 동작 중일 때와 동작의 끝 신호를 `&&` 를 통해 선언하였습니다.

    - 따라서 `o_running` 일때와 코어에서 종료시점 인 `main_cnt = num_cnt -1` 을 종료점으로 잡아 `w_run_fin` 신호를 생성하였습니다.

3. `main_cnt` 코어에서는 상태가 `o_running` 중일 때 동작을 시작하고 `w_run_fin` 일 때 종료하게 설정하였습니다. 


### data output

```verilog
wire [CNT_BIT-1 :0]w_num;
assign w_num =  main_cnt;
// wire [CNT_BIT-1:0] w_num = main_cnt;


always@(posedge clk, negedge reset_n) begin
    if(!reset_n) begin
        o_num <= 0;
    end else if (o_done) begin
        o_num <= w_num;  
    end else if (o_running) begin
        o_num <= 0;
    end
end


endmodule
```

최종적으로 데이터를 출력하는 부분입니다.

1. `w_num` 은 `main_cnt` 의 값을 가져옵니다.

2. `always` 문에서 `o_done` 신호를 받으면 저장되있던 신호를 `o_num` 으로 가져옵니다.

3. 이때 상태가 `DONE` -> `DONE_D` 으로 변할 때 값을 가져옵니다.




## axi_ slave.v 

AXI_SLAVE 부분은 코드가 길기 때문에 중요 부분만 가져와서 사용하겠습니다.


### module 선언

```verilog
	module myip_v1_0_S00_AXI #
	(
		// 여기에 사용자 매개변수를 추가하세요.
		parameter CNT_BIT = 31,

		// 사용자 매개변수 끝.
		// 이 선을 넘어 매개변수를 수정하지 마세요.

		// S_AXI data bus 의 폭
		parameter integer C_S_AXI_DATA_WIDTH	= 32,
		// S_AXI address bus 의 폭
		parameter integer C_S_AXI_ADDR_WIDTH	= 4
	)
	(
		// 여기에 사용자 포트를 추가하세요.

		output axi_run,  // axi: o_run -> fsm : i_run  , 마스터에서 온 신호를 보내줘야함
		output [CNT_BIT-1 : 0] axi_num_cnt, 
		input [CNT_BIT-1 : 0] fsm_num,
		input fsm_idle,    // fsm 의 신호를 받아야 함 
		input fsm_running, // 이 신호들은 axi 를 통해 PS 로 넘어감
		input fsm_done,
		// 사용자 포트 끝.
		// 이 선을 넘어 포트를 수정하지 마세요.

        // 이하 생략...
    )
```

1. 파라미터로 `CNT_BIT` 값을 그대로 선언하였습니다.

2. 앞서 설명한 것 처럼 `AXI_LITE` 의 데이터 버스의 폭과 주소 폭으로 기본 값을 사용하였습니다.

3. 포트를 설정하였습니다.
    - `FSM_CNT.v` 에서 처리한 상태를 `Master`에 보내기 위해 `AXI-slave.v` 로 상태 값을 입력 받습니다.

    - 반대로 `FSM_CNT.v` 로 입력 받은 데이터와 `i_run` 을 보내기 위해 `output` 으로 선언하였습니다.


### i_run 신호 처리

```verilog
	//  user logic 을 여기다 추가하세요.
	reg r_i_run;
	reg r_o_done;
	//1. i_run 신호가 들어왔을 때
	always@(posedge S_AXI_ACLK, negedge S_AXI_ARESETN) begin
		if(!S_AXI_ARESETN) begin
			r_i_run <= 0;
		end else begin
			r_i_run <= slv_reg0[CNT_BIT];  // i_run 
		end
	end
```


| 주소   | R/W      |이름      | 비트 필드     | 설명                     |
|--------|----------|----------|---------------|--------------------------|
| `0x00` |  R/W    |CONTROL       | [31] : i_run [30:0] : Num_cnt        | 제어 레지스터            |
| `0x04` |   R   |STATUS        | [0]:IDLE <br> [1]: RUN <br>[2] : DONE <br> [3] : DONE_D | FSM 상태 플러그    |


다음과 같은 주소로 데이터를 처리할 예정이며 그러기 위해 표를 작성하였습니다.

1. `reg` 형으로 2개의 신호를 선언하였습니다.

2. `0x00` 주소로 `i_run` 신호와 `i_num_cnt` 값을 받아올 예정임으로 `slv_reg0[CNT_BIT]` 을 통해 `slv_reg0[31]` 인 `i_run` 신호를 받아올 예정입니다.


### output 신호 처리

```verilog
	assign axi_run = ~r_i_run & slv_reg0[CNT_BIT];
	assign axi_num_cnt = slv_reg0[CNT_BIT-1 : 0];
```

1. `axi_run` 은 `i_run` 신호를 의미하며 이 신호를 1 사이클만 사용하기 위해 `rising edge tick capture` 방식을 사용하였습니다.

    - `slv_reg0[CNT_BIT]` 가 `1`일 때 `r_i_run` 은 `0`이고 한 사이클 뒤 `slv_reg0[CNT_BIT]` 가 `0`일 때 `r_i_run` 은 `1`입니다.

    - 이를 활용하여 `slv_reg0[CNT_BIT]` 와 `axi_run` 를 동기화 하기 위해서 `~r_i_run & slv_reg0[CNT_BIT]` 을 사용합니다.

2. `axi_num_cnt` 는 `i_num_cnt` 로 `slv_reg0[30:0]` 을 사용하여 나머지 부분을 받았습니다.



### input 신호 처리

```verilog
always @(posedge S_AXI_ACLK, negedge S_AXI_ARESETN) begin
		if(!S_AXI_ARESETN) begin
			r_o_done <= 0;
		end else if(fsm_done) begin
			r_o_done <= 1;
		end else if(axi_run) begin
			r_o_done <= 0;
		end

	end
```

`fsm_done` 은 `o_done` 이며 `reg` 형으로 선언되었기 때문에 클럭에 동기화 되어 신호가 들어옵니다. 

`r_o_done` 은 다음 사이클에 1로 활성화 되며 상태 `DONE_D` 와 일치하게 됩니다.


```verilog
always @(posedge S_AXI_ACLK, negedge S_AXI_ARESETN) begin
		if(!S_AXI_ARESETN) begin
		slv_reg1 <= 32'b0;
		end else begin
			slv_reg1[0] <= fsm_idle;
			slv_reg1[1] <= fsm_running;
			slv_reg1[2] <= r_o_done;
			slv_reg1[3] <= fsm_done;
		end
	end
```

`slv_reg1` 는 `0x04` 주소를 사용하고 있으며 `32비트` 중 `0~4` 만 상요하며 상태 값을 전달 하는 역할을 합니다.


```verilog
	always@(posedge S_AXI_ACLK, negedge S_AXI_ARESETN) begin
		if(!S_AXI_ARESETN) begin
		slv_reg2 <= 32'b0;
		end else if (r_o_done)begin
			slv_reg2 <= fsm_num;
		end	
	end

	// User logic 끝

	endmodule

```

`slv_reg2` 는 `0x08` 주소를 사용하고 있으며 `r_o_done` 상태일 때 저장된 값을 넘깁니다.

#### 상태를 이렇게 설계한 이유

설계를 할 때 `DONE` 과 `DONE_D` 의 2가지 상태를 설정하였는데 상태의 여유를 두고 값을 넘기고 싶었기 때문에 이렇게 설계하였습니다.

또한 값의 타이밍에 맞게 설계하기 위해 다음과 같은 방법을 사용하였습니다.




---

