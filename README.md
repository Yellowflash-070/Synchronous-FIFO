# Synchronous-FIFO
Design and Verification of a Synchronous 8bit-16location FIFO using System Verilog concepts.


## Verilog code for Design module

``` systemverilog

// FIFO design code

module FIFO(clk,rst,rd,wr,din,full,empty,dout);
  input clk,rst,rd,wr;
  input [7:0]din;
  output reg [7:0]dout;
  output full,empty;
  
  reg [3:0]rd_ptr = 0,wr_ptr = 0;
  reg [4:0]count = 0;
  reg [7:0]mem[15:0];
  
  
  assign full = (count == 16) ? 1'b1 : 1'b0;
  assign empty = (count == 0) ? 1'b1 : 1'b0;
  
  always @(posedge clk)
    begin
      if(rst)
        begin
          rd_ptr <= 0;
          wr_ptr <= 0;
          count <= 0;
        end
      else if (wr && !full)
        begin
          mem[wr_ptr] <= din;
          wr_ptr <= wr_ptr + 1;
          count <= count + 1;
        end
      else if (rd && !empty)
        begin
          dout <= mem[rd_ptr];
          rd_ptr <= rd_ptr + 1;
          count <= count - 1;
        end
      else
        begin
          rd_ptr <= rd_ptr ;
          wr_ptr <= wr_ptr;
          count <= count;
        end
    end
  
endmodule


// interface for FIFO design

interface fifo_if;
  
  logic clk,rst,rd,wr;
  logic full,empty;
  logic [7:0] din, dout;
  
  
endinterface
  

```

## TESTBENCH for FIFO design

``` systemverilog

// TESTBENCH for FIFO design


// -----------------------------transaction class-------------------------------

class transaction;
  
  rand bit oper;
  bit rd, wr;
  bit [7:0]din, dout;
  bit full, empty;
  
  constraint oper_ctrl {
    oper dist{ 1 :/ 50, 0 :/ 50};
  
  }
  
endclass


// -----------------------------generator class---------------------------------

class generator;
  
  transaction tr; // transaction obj
  mailbox #(transaction) mbx;  // parameterized mailbox for transaction data
  
  int count = 0;
  int i = 0; // for iteration count
  
  event next; // event to send next transaction
  event done; // event to signify the completion of all transaction
  
  // custom constructor with mbx argument
  function new(mailbox #(transaction) mbx);
    this.mbx = mbx;
    tr = new();
  endfunction
  
  task run();
    
    repeat(count)
      begin
        assert(tr.randomize) else $error("RANDOMIZATION FAILED!!");
        i++;
        mbx.put(tr); // put transaction data into the mailbox
        
        $display("[GEN] : Oper : %0d Iteration : %0d", tr.oper, i);
        @(next); // wait till the scoreboard operation is complete
      end
    
    ->done; // triggered when generation operation is finished. 
    
  endtask
  
endclass

// --------------------------------driver class---------------------------------

class driver;
  
  virtual fifo_if fif; //interface obj declaration
  
  mailbox #(transaction) mbx;
  
  transaction datac; //transaction obj to store data from generator.
  
  
  function new (mailbox #(transaction) mbx);
    this.mbx = mbx;
  endfunction
  
  //task to reset DUT
  task reset();
    fif.rst <= 1'b1;
    fif.rd <= 1'b0;
    fif.wr <= 1'b0;
    fif.din <= 0;
    repeat(5)@(posedge fif.clk);
    fif.rst <= 1'b0;
    $display("[DRV] : DUT RESET DONE!!");
    $display("--------------------------------------------------------");
  endtask
  
  
  //task to write data
  task write();
    
    @(posedge fif.clk);
    fif.rst <= 1'b0;
    fif.rd <= 1'b0;
    fif.wr <= 1'b1; // enable wr
    fif.din <= $urandom_range(1,10); // random value for din
    @(posedge fif.clk);
    fif.wr <= 1'b0; //disable wr
    $display("[DRV] : DATA WRITE data : %0d", fif.din);
    @(posedge fif.clk);

  endtask
  
  
  //task to read data
  task read();
    
    @(posedge fif.clk);
    fif.rst <= 1'b0;
    fif.wr <= 1'b0;
    fif.rd <= 1'b1; // enable rd
    @(posedge fif.clk);
    fif.rd <= 1'b0; // disable rd
    $display("[DRV] : DATA READ");
    @(posedge fif.clk);

  endtask
  
  //main task for driver
  task run();
    datac = new();
   
    forever begin
       mbx.get(datac);
      if(datac.oper == 1'b1)
        write();
      else
        read();
    end
    
  endtask
    
endclass
  

// ----------------------------------monitor class------------------------------

class monitor;
  
  transaction tr;
  
  virtual fifo_if fif; //interface obj declaration
  
  mailbox #(transaction) mbx;
  
  
  //custom constructor
  function new( mailbox #(transaction) mbx);
    this.mbx = mbx;
  endfunction
  
  //main task for monitor
  task run();
    tr = new();
   
    forever begin
      
      repeat(2) @(posedge fif.clk);
      tr.wr = fif.wr;
      tr.rd = fif.rd;
      tr.din = fif.din;
      tr.full = fif.full;
      tr.empty = fif.empty;
      
      @(posedge fif.clk);
      tr.dout = fif.dout;
      
      mbx.put(tr); //put transaction data in mailbox
      $display("[MON] : wr: %0d rd : %0d din : %0d, dout : %0d full : %0d empty %0d", tr.wr, tr.rd, tr.din, tr.dout, tr.full, tr.empty);
      
    end
    
  endtask
  
endclass

// ----------------------------------scoreboard class---------------------------

class scoreboard;
  
  transaction tr;
  
  mailbox #(transaction) mbx;
  
  event next;
  
  //queue to store golden data
  bit [7:0] din[$];
  bit[7:0] temp;
  int error = 0;
  
  
  //custom constructor
  function new(mailbox #(transaction) mbx);
    this.mbx = mbx;
  endfunction
  
  //main task for scoreboard
  task run();
    
    forever begin
      
      mbx.get(tr);
      
      $display("[SCO] : wr: %0d rd : %0d din : %0d, dout : %0d full : %0d empty %0d", tr.wr, tr.rd, tr.din, tr.dout, tr.full, tr.empty);
       
      //for write         
      if(tr.wr)
        begin
          if(!tr.full)
            begin
              din.push_front(tr.din);  //push data into the queue
              $display("[SCO] : DATA STORED IN QUEUE : %0d", tr.din);
            end
          else
            begin
              $display("[SCO] : FIFO IS FULL!!");
            end
          $display("-----------------------------------------------------");
        end
      
      // for read
      if(tr.rd)
        begin
          if(!tr.empty)
            begin
              temp = din.pop_back(); //pop data from the queue and store it in temp
              
              if(tr.dout == temp)
                $display("[SCO] : DATA MATCHED!!");
              else begin
                $error("[SCO] : DATA MISMATCHED!!");
                error++;
              end
            end
          else
            begin
              $display("[SCO] : FIFO IS EMPTY!!");
            end
          $display("-----------------------------------------------------");
        end
               
     ->next; //trigger next event once a single read/write is complete
                 
    end
  endtask
  
endclass
               
//------------------------------------environment class----------------------------
               
class environment;
  
  //instances for all the class
  generator gen;
  driver drv;
  monitor mon;
  scoreboard sco;
  
  //mailboxes between different classes
  mailbox #(transaction) gdmbx; //b/w gen and drv
  mailbox #(transaction) msmbx; //b/w mon and sco
  
  event nextgs;
  
  virtual fifo_if fif;
  
  function new(virtual fifo_if fif);
    
    //for gen and drv
    gdmbx = new();
    gen = new(gdmbx);
    drv = new(gdmbx);
    
    
    //for mon and sco
    msmbx = new();
    mon = new(msmbx);
    sco = new(msmbx);
    
    this.fif = fif;
    
    drv.fif = this.fif;
    mon.fif = this.fif;
    
    gen.next = nextgs;
    sco.next = nextgs;
    
  endfunction
  
  //task done prior to applying stimuli to DUT
  task pre_test();
    drv.reset();
  endtask
  
  //task done while application of stimuli to DUT
  task test();
    fork
      gen.run();
      drv.run();
      mon.run();
      sco.run();
    join_any
  endtask
  
  //task done after application of all stimuli to DUT
  task post_test();
    wait(gen.done.triggered); // wait for gen to send all stimuli to drv to trigger done event
    $display("----------------------------------------------------------");
    $display("Error Count : %0d", sco.error);
    $display("----------------------------------------------------------");
    
    $finish();
  endtask
  
  //main task for environment
  task run();
    pre_test();
    test();
    post_test();
  endtask
  
endclass
               
// -------------------------------------testbench top------------------------------
               
module tb;
  
  fifo_if fif(); //interface instantiation
  
  //connection of interface signal to DUT
  FIFO dut(.clk(fif.clk),.rst(fif.rst),.rd(fif.rd),.wr(fif.wr),.din(fif.din),.dout(fif.dout),.full(fif.full),.empty(fif.empty));
  
  //clk generation
  initial begin
    fif.clk <= 0;
  end
  
  always #10 fif.clk = ~fif.clk;
  
  environment env;
  
  initial begin
    env = new(fif);
    env.gen.count = 10;
    env.run();
  end
  
  initial begin
    $dumpfile("dump.vcd");
    $dumpvars;
  end
  
endmodule


```

## Console Output

![image](https://github.com/user-attachments/assets/9efabe1b-802a-42ad-b4b2-ad4373bdaa54)

## Simulation Waveform
![image](https://github.com/user-attachments/assets/0216b3e0-8d24-4e66-ac49-57d6395e4331)


