# TASK 2 - Sine and Cosine Dual Wave

``` SystemVerilog
module counter #(
    parameter WIDTH = 8
)(
    // interface signals
    input logic                     clk,    // clock
    input logic                     rst,    // reset
    input logic                     en,     // counter enable
    input logic     [WIDTH-1:0]     incr,
    output logic    [WIDTH-1:0]     count    // count output
);

always_ff @ (posedge clk)
    if (rst)    count <= {WIDTH{1'b0}};
    else        count <= en ? count + incr : count;

endmodule
```


```SystemVerilog
module sinegen # (
    parameter   A_WIDTH = 8,
                D_WIDTH = 8
)(
    input logic                     clk,
    input logic                     rst,
    input logic                     en,
    input logic     [D_WIDTH-1:0]   incr,
    output logic    [D_WIDTH-1:0]   dout   
);

    logic[A_WIDTH-1:0]              address;

    counter addrCounter (
        .clk (clk),
        .rst (rst),
        .en (en),
        .incr (incr),
        .count (address)
    );

    rom sineRom (
        .clk (clk),
        .addr (address),
        .dout (dout)
    );

endmodule
```

```C++
#include "Vsinegen.h"
#include "verilated.h"
#include "verilated_vcd_c.h"
#include "vbuddy.cpp"

#define MAX_SIM_CYC 1000000

int main(int argc, char **argv, char **env)
{
    int tick;
    int simcyc;

    Verilated::commandArgs(argc, argv);

    // init top verilog instance, sinegen
    Vsinegen *top = new Vsinegen;

    // init trace dump
    Verilated::traceEverOn(true);
    VerilatedVcdC *tfp = new VerilatedVcdC;
    top->trace(tfp, 99);
    tfp->open("sinegen.vcd");

    // init Vbuddy
    if (vbdOpen() != 1)
        return (-1);
    vbdHeader("Lab 2: Sine Generator");

    // init simulation inputs
    top->clk = 1;
    top->rst = 0;
    top->en = 1;
    top->incr = 1;

    // run simulation for N samples
    for (simcyc = 0; simcyc < MAX_SIM_CYC; simcyc++)
    {

        for (tick = 0; tick < 2; tick++)
        {
            tfp->dump(2 * simcyc + tick);
            top->clk = !top->clk;
            top->eval();
        }

        // ++++ Send count value to Vbuddy
        vbdPlot(int(top->dout), 0, 256);
        vbdCycle(simcyc + 1);

        if ((Verilated::gotFinish()) || (vbdGetkey() == 'q'))
        {
            exit(0);
        }
    }

    vbdClose();
    tfp->close();
    exit(0);
}
```

```Shell
#!/bin/sh

#cleanup
rm -rf obj_dir
rm -f counter.vcd

#run Verilator to translate Verilog into C++, including C++ testbench
verilator -Wall --cc --trace sinegen.sv --exe sinegen_tb.cpp
verilator -Wall --cc --trace rom.sv
verilator -Wall --cc --trace counter.sv

#build c++ project via make autormatically generated by Verilator
make -j -C obj_dir/ -f Vsinegen.mk Vsinegen

#run executable simulation file
obj_dir/Vsinegen
```

```Python
import math
import string
f = open("sinerom.mem", "w")
for i in range(256):
    v = int(math.cos(2*3.1416*i/256)*127+127)
    if (i+1) % 16 == 0:
        s = "{hex:2X}\n"
    else:
        s = "{hex:2X} "
    f.write(s.format(hex=v))

f.close()
```