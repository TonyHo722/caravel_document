---
title: caravel_soc FPGA create_clock example
type: slide
slideOptions:
    theme: night
    transition: slide
    progress: true
---


## caravel_soc FPGA create_clock Example
by Tony Ho

----


## Topic
- setup caravel_soc
- caravel_soc FGPA timing issue after synthsis (without constrains)
- hold time issue in ext_spi_clock
- How Viavado synthesis clock?
- xdc constrain by tcl command


---


### setup the project for check hold time issue
- download the vvd_caravel_soc_202230502.zip(project file) from [this link in slack]( 
https://boledu.slack.com/files/U04S0QP0F38/F056555HWSD/vvd_caravel_soc_202230502.zip) provided by Willy 
- use vivado 2020.2 to open the project

---


### caravel_soc FGPA timing issue after synthsis (without constrains)
![](https://i.imgur.com/VIYKBeT.png)


----

### Report Clock Network
![](https://hackmd.io/_uploads/B1ZgXHLEh.png)


----

### schematic example

![](https://hackmd.io/_uploads/B1817HU42.png)


----

### Constraints Wizard(1)
![](https://hackmd.io/_uploads/ryaGEHLVn.png)


----

### Constraints Wizard(2)
![](https://hackmd.io/_uploads/SyPQEHIN3.png)


----


### Result - after add constrains (1)


	create_clock -period 100.000 -name core_clock -waveform {0.000 50.000} [get_ports clock]
	create_clock -period 1000.000 -name ext_spi_clock -waveform {0.000 500.000} [get_ports {mprj_i[4]}]
	set_clock_groups -name core_clock_group -asynchronous -group [get_clocks core_clock]
	set_clock_groups -name ext_spi_clock_group -asynchronous -group [get_clocks ext_spi_clock]


----


### Result - after add constrains (2)

![](https://i.imgur.com/pPh7kCI.png)



---


## hold time issue in ext_spi_clock
    predata -> gpio_conifugration
![](https://i.imgur.com/WYwbXVd.png)


----


### csclk comes from wbbd or mgmt_gpio_in[4]
    assign csclk = (wbbd_busy) ? wbbd_sck : ((spi_is_active) ? mgmt_gpio_in[4] : 1'b0);

[housekeeping.v](https://github.com/bol-edu/caravel-soc/blob/main/rtl/soc/housekeeping.v#L1078)
![](https://i.imgur.com/4tTkdvb.png)


----


### code trace : predata -> odata
predata -> odata -> idata -> cdata -> gpio_configure

    assign odata = {predata, SDI};

[housekeeping_spi.v](https://github.com/bol-edu/caravel-soc/blob/main/rtl/soc/housekeeping_spi.v#L120)

![](https://i.imgur.com/4tTkdvb.png)


----


### code trace : SCK is the clock of predata
SCK = mgmt_gpio_in[4]

    always @(posedge SCK or posedge csb_reset) begin
        ...
        predata <= 7'b0000000;
        ...
    end else begin
[predata](https://github.com/bol-edu/caravel-soc/blob/main/rtl/soc/housekeeping_spi.v#L172-L189)

![](https://i.imgur.com/4tTkdvb.png)


----


### code trace : odata -> idata
SCK come from mgmt_gpio_in[4]
predata -> odata -> idata -> cdata -> gpio_configure

    housekeeping_spi hkspi (
        ...
        .SCK(mgmt_gpio_in[4]),
        .odata(idata),
        ...
    );
    
[housekeeping.v](https://github.com/bol-edu/caravel-soc/blob/main/rtl/soc/housekeeping.v#L811-L829)


----


### idata -> cdata
predata -> odata -> idata -> cdata -> gpio_configure

    assign cdata = (wbbd_busy) ? wbbd_data : idata;
[housekeeping.v](https://github.com/bol-edu/caravel-soc/blob/main/rtl/soc/housekeeping.v#L1079)


----


### cdata -> gpio_configure
predata -> odata -> idata -> cdata -> gpio_configure

    always @(posedge csclk or negedge porb) begin
        if (cwstb == 1'b1) begin
            case (caddr)
            ...
                8'h1e: begin
                    gpio_configure[0][7:0] <= cdata;
                end                
            ...
        end                
    end                
csclk is the clock for gpio_configure

[housekeeping.v](https://github.com/bol-edu/caravel-soc/blob/main/rtl/soc/housekeeping.v#L1227)
[case (caddr)](https://github.com/bol-edu/caravel-soc/blob/main/rtl/soc/housekeeping.v#L1150-L1497)

----

### issue: predata and gpio_configure use different clock
    assign csclk = (wbbd_busy) ? wbbd_sck : ((spi_is_active) ? mgmt_gpio_in[4] : 1'b0);

- predata and use mgmt_gpio_in[4] as clock
- gpio_configure use csclk as clock
    - csclk comes from wbbd_sck(core_clock) or mgmt_gpio_in[4]

[housekeeping.v](https://github.com/bol-edu/caravel-soc/blob/main/rtl/soc/housekeeping.v#L1078)
![](https://i.imgur.com/4tTkdvb.png)

----


### detail schematic
![](https://i.imgur.com/4tTkdvb.png)


----


### Question
should I set csclk as a generated_clock? and with a MUX to select the source from core_clock or external_spi_clock?

----

### reference - Multiplexed Clock Example
[Multiplexed Clock Example](https://support.xilinx.com/s/article/62488?language=en_US)
![](https://i.imgur.com/8IGgkXu.png)
When the paths A, B and C do not exist

    set_clock_groups -logically_exclusive -group clk0 -group clk1

----

### reference - Multiplexed Clock Example
![](https://i.imgur.com/8IGgkXu.png)
When the paths A or B or C exist

    create_generated_clock -name clk0mux -divide_by 1 -source [get_pins mux/I0] [get_pins mux/O]
    create_generated_clock -name clk1mux -divide_by 1 -add -master_clock clk1 -source [get_pins mux/I1] [get_pins mux/O]
    set_clock_groups -physically_exclusive -group clk0mux -group clk1mux

----


### csclk schematic

![](https://i.imgur.com/D5JMF27.png)



----


### update xdc file
	create_clock -period 100.000 -name core_clock -waveform {0.000 50.000} [get_ports clock]
	create_clock -period 1000.000 -name ext_spi_clock -waveform {0.000 500.000} [get_ports {mprj_i[4]}]
	set_clock_groups -name core_clock_group -asynchronous -group [get_clocks core_clock]
	set_clock_groups -name ext_spi_clock_group -asynchronous -group [get_clocks ext_spi_clock]
	
	create_generated_clock -name core_clock_csclk_mux -source [get_pins housekeeping/hkspi_disable_i_5/I0] -multiply_by 1 -add -master_clock core_clock [get_pins housekeeping/hkspi_disable_i_5/O]
	create_generated_clock -name ext_spi_clock_csclk_mux -source [get_pins housekeeping/hkspi_disable_i_5/I5] -multiply_by 1 -add -master_clock ext_spi_clock [get_pins housekeeping/hkspi_disable_i_5/O]
	set_clock_groups -name cs_clk_mux -physically_exclusive -group [get_clocks core_clock_csclk_mux] -group [get_clocks ext_spi_clock_csclk_mux]

![](https://i.imgur.com/D5JMF27.png)

----

#### hold time issue in ext_spi_clock fixed after update xdc file
before update xdc file
![](https://i.imgur.com/tkB4jiy.png)
after update xdc file
![](https://i.imgur.com/l6w3ppE.png)

---

### check hold time issue for core_clock
Overview
![](https://i.imgur.com/WZP3pGU.png)

----

#### Why user_project_wrapper clock come from la_out_storage_reg?
 ![](https://i.imgur.com/vyLHhVN.png)

----

#### Why user_project_wrapper clock come from la_out_storage_reg?

    assign clk = (~la_oenb[64]) ? la_data_in[64]: wb_clk_i;
    assign rst = (~la_oenb[65]) ? la_data_in[65]: wb_rst_i;

    counter #(
        .BITS(BITS)
    ) counter(
        .clk(clk),
        ...
    );
    
[user_proj_example.counter.v](https://github.com/bol-edu/caravel-soc/blob/main/rtl/user/user_proj_example.counter.v#L104-L121)

----

#### la_out_storage_reg

![](https://i.imgur.com/XyuXlQU.png)

----

#### Path 11
![](https://i.imgur.com/zUJXv22.png)

----

#### try to workarond this issue by update code to move the mux
It is work.
![](https://i.imgur.com/CL81WGZ.png)


----


#### Question
- la_data_in come from core_clock and change by CPU
- wb_clk_i = core_clock
- How to add constrain for it? use set_multicycle_path?


----

#### schematic of clk

    assign clk = (~la_oenb[64]) ? la_data_in[64]: wb_clk_i;

![](https://i.imgur.com/LRXcM3O.png)


----

#### fixed by add constrain
	create_generated_clock -name core_clock_la_mux -source [get_pins soc/core/ready_i_5/I0] -divide_by 2 -add -master_clock core_clock [get_pins soc/core/ready_i_5/O]
	create_generated_clock -name core_clock_source_mux -source [get_pins soc/core/ready_i_5/I2] -divide_by 1 -add -master_clock core_clock [get_pins soc/core/ready_i_5/O]
	set_clock_groups -name cs_clk_mux -physically_exclusive -group [get_clocks core_clock_la_mux] -group [get_clocks core_clock_source_mux]		
    
![](https://i.imgur.com/UfcX1PY.png)


----

#### remain issues

![](https://hackmd.io/_uploads/HkCwDI8En.png)

---

### referenc sdc file from caravan

[caravan.sdc](https://github.com/efabless/caravel/blob/main/signoff/caravan/caravan.sdc)

```
### Caravan Signoff SDC
### Rev 2
### Date: 28/10/2022

## MASTER CLOCKS
create_clock -name clk -period 25 [get_ports {clock}] 

create_clock -name hkspi_clk -period 100 [get_pins {housekeeping/mgmt_gpio_in[4]} ] 
create_clock -name hk_serial_clk -period 50 [get_pins {housekeeping/serial_clock}]
create_clock -name hk_serial_load -period 1000 [get_pins {housekeeping/serial_load}]
# hk_serial_clk period is x2 core clock

set_clock_uncertainty 0.1000 [get_clocks {clk hkspi_clk hk_serial_clk hk_serial_load}]

set_clock_groups \
   -name clock_group \
   -logically_exclusive \
   -group [get_clocks {clk}]\
   -group [get_clocks {hk_serial_clk}]\
   -group [get_clocks {hk_serial_load}]\
   -group [get_clocks {hkspi_clk}]

# clock <-> hk_serial_clk/load no paths
# future note: CDC stuff
# clock <-> hkspi_clk no paths with careful methods (clock is off)

set_propagated_clock [get_clocks {clk}]
set_propagated_clock [get_clocks {hk_serial_clk}]
set_propagated_clock [get_clocks {hk_serial_load}]
set_propagated_clock [get_clocks {hkspi_clk}]

```


### result - 94 Failing Endpoints

![](https://hackmd.io/_uploads/rkVzUdDN3.png)


----


### update code to workaround it - 62 Failing Endpoints

![](https://hackmd.io/_uploads/Hyx9LdPVn.png)

----

#### Why it is hold time issue?
- the two filp-flop use the same clock source.

![](https://hackmd.io/_uploads/H15k_ODEh.png)


---



### How Viavado synthesis clock?
- when no constrain
![](https://i.imgur.com/yINcC6t.png)


----

### How Viavado synthesis clock?
- when no constrain
    - auto-gen IBUF & BUFG for clock pin in Filp-flop
![](https://i.imgur.com/qFHMivA.png)


---




### xdc constrain by tcl command
![](https://i.imgur.com/qRSXZb5.png)



----

### xdc constrain can be add by 
- tcl command
- or update .*.xdc file
- or use UI


----

#### get the name of a cell (1)
![](https://i.imgur.com/KjDDKy1.png)


----


#### Find the name of a cell (2)
![](https://i.imgur.com/uPIKPbY.png)

----

#### How to report clock network?
- after update xdc file result in below
![](https://i.imgur.com/boCqhIj.png)

----


## End

