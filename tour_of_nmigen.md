# A (Very) Quick Tour of nMigen as of May 25th, 2021

> Note: This document expects basic familiarity of digital design and Python.
  Examples copied/modified from other sources such as the official documentation/comments
  primarily written by [whitequark](https://github.com/whitequark).
  This documentation is not exhaustive, but should be sufficient for starting out.

# Language

> Note: This section assumes you have installed nmigen and have
  `from nmigen import *` at the top of your code

## Modules
* Fundamental building block of nMigen
* Python class that inherits from `Elaboratable`
  - `__init__` method for initialization of resources
  - `elaborate` method describing logic of module (should instantiate and a
    return a Module object like below)
    ```python
    def elaborate(self, platform):
        m = Module()
        # cool logic goes here
        return m
    ```
* Has one or more clock domains (and a combinatorial domain)
  - Combinatorial domain named `m.d.comb`
  - Default synchronous domain named `m.d.sync`
  - Statements can be added to each domain with the += operator (more on this later)
  - Clock domains can be added to modules with the following syntax
    ```python
    m = Module()
    cd_por = ClockDomain(
        name='optional_name', # Default: (variable name).removeprefix('cd_')
        reset_less=True,      # Default: False
        clk_edge='neg',       # Options: {'pos', 'neg'}; Default: 'pos'
        async_reset=True,     # Default: False
        local=True,           # Default: False
    )
    m.domains += cd_por
    ```
  - Use `ClockDomain.clk` to access the raw clock signal from a domain
  - Use `ClockDomain.rst` to access the raw reset signal from a domain
* Can have submodules
  - Instantiate the submodule in `elaborate`
  - Add it to the explicit list of submodules with `m.submodules['explicit_name'] = submodule`
* Provides control structures:
  - Control structures used in `with` context manager blocks
  - `m.If()`, `m.Elif()`, and `m.Else()` provide simple control flow
    ```python
    with m.If(self.a):
        pass
    with m.Elif(self.b):
        pass
    with m.Else():
        pass
    ```
  - `m.Switch()`, `m.Case()`, and `m.Default()` emulate a switch statement
    ```python
    with m.Switch(self.a):
        with m.Case("1--"): # matches {100, 101, 110, 111}
            pass
        with m.Case("001"):
            pass
        with m.Default():
            pass 
    ```
  - `m.FSM()`, `m.State()`, and `m.next` can be used to instantiate a finite state machine.
    ```python
    with m.FSM(
        domain='clock_domain', # Default 'sync'
        name='optional_name', # Default 'fsm'
        reset='STATE_ONE', # If not specified, reset state unkown
    ) as name:
        with m.State('STATE_ONE'):
            m.next = 'STATE_TWO'
        with m.State('STATE_TWO'):
            m.next = 'STATE_ONE'
    ```

> Note: module control structures are different than Python control structures.
> Module control structures describe the behavior of the FPGA while it's running.
> Python's control structures are more meta, and describe what hardware to generate
  (akin to `#ifdef` in C/C++ or `generate` in Verilog).

## Instance
* Provides a mechanism for communication with FPGA-specific primitives
* Looks like 'opaque-box' module
* Constructor takes key-value arguments associating a named primitive port with some local connection
  - Naming convention (prefix must be added to primitive port name):
    | Prefix      | Means         |
    |-------------|---------------|
    |`i_`         | input         |
    |`o_`         | output        |
    |`io_`        | input/output  |
    |`p_`         | parameter     |
  ```python
  primitive = Instance(
      'Name_Of_Primitive',
      'i_Name_Of_An_Input_Port'=self.signal_a,
      'o_Name_Of_An_Output_Port'=self.signal_b,
      'p_Parameter'='0b111',
  )
  m.submodules['explicit_name'] = primitive
  ```

## Constants
* Constants can be created with `Const()` or `C()`
* Can be signed or unsigned and have a bit width
  - This underlying shape can be observed with a call to `Const.shape()`.

## Signals
* `Signal()` represents a potentially varying number
* Signals can be signed or unsigned and have a bit width
  - This underlying shape can be observed with a call to `Signal.shape()`.
* Signals can have named (`name=`) and reset values (`reset=`) that are set in the constructor
* Statements can be constructed representing operations on and about Signals
* The following operations can be performed on Signals:
  - Arithmetic:
    | Operation | Meaning        | Notes |
    |-----------|----------------|-------|
    | `a + b`   | addition       | |
    | `a - b`   | subtraction    | |
    | `a * b`   | multiplication | |
    | `a // b`  | floor division | `b` must be unsigned |
    | `a % b`   | modulo         | `b` must be unsigned |
    | `abs(a)`  | absolute value | |
    | `-a`      | unary negation | |
  - Bit Manipulation:
    | Operation           | Meaning              | Notes |
    |---------------------|----------------------|-------|
    | `~a`                | bitwise not          | |
    | `a & b`             | bitwise and          | |
    | `a \| b`            | bitwise or           | |
    | `a ^ b`             | bitwise xor          | |
    | `a.implies(b)`      | bitwise imply        | |
    | `a >> b`            | arith. right shift   | `b` must be unsigned |
    | `a << b`            | left shift           | `b` must be unsigned |
    | `a.rotate_left(n)`  | rotate left (const)  | `n < 0` reverses direction |
    | `a.rotate_right(n)` | rotate right (const) | `n < 0` reverses direction |
    | `a.shift_left(n)`   | shift left (const)   | `n < 0` reverses direction |
    | `a.shift_right(n)`  | shift right (const)  | `n < 0` reverses direction |
    | `a.all()`           | all bits set?        | |
    | `a.any()`           | any bits set?        | |
    | `a.xor()`           | odd number bits set? | |
  - Logical:
    | Operation                | Meaning      | Notes |
    |--------------------------|--------------|-------|
    | `a.bool()`               | boolean cast | |
    | `~(a).bool()`            | logical not  | |
    | `(a).bool & (b).bool()`  | logical and  | |
    | `(a).bool \| (b).bool()` | logical or   | |
  - Comparison:
    | Operation | Meaning                  | Notes |
    |-----------|--------------------------|-------|
    | `a == b`  | equality                 | |
    | `a != b`  | inequality               | |
    | `a < b`   | less than                | |
    | `a <= b`  | less than or equal to    | |
    | `a > b`   | greater than             | |
    | `a >= b`  | greater than or equal to | |
  - Sequence:
    | Operation             | Meaning                 | Notes |
    |-----------------------|-------------------------|-------|
    | `len(a)`              | length/width of `a`     | |
    | `a[i:j:k]`            | Python slice of `a`     | |
    | `Cat(a, b)`           | Concatenate `a` and `b` | `a` is LSBs, `b` is MSBs |
    | `Repl(a, n)`          | Replicate `a` n times   | |
    | `a.bit_select(b, w)`  | `a[b:b+w]`              | |
    | `a.word_select(b, w)` | `a[b*w:b*w+w]`          | |
  - Miscellaneous:
    | Operation            | Meaning                  | Notes |
    |----------------------|--------------------------|-------|
    | `a.as_signed()`      | Cast `a` signed          | |
    | `a.as_unsigned()`    | Cast `a` unsigned        | |
    | `Mux(s, a, b)`       | if `s` then `a` else `b` | |
* Signals can be assigned with the `Signal.eq()` method
  - Assignment must be associated with a clock domain (or `comb`)
  - A Signal can only be written from one domain
  - Assignment may truncate the most significant bits
  - Assignment syntax:
    ```python
    m.d.comb += [
        self.example.eq(self.other),
        Cat(self.a, self.b).eq(0b1100), # self.a is 0b00, self.b is 0b11 (surprise?)
        # More statements here...
    ]
    m.d.sync += self.c.eq(self.c + 1) # Don't need [] if just 1 statement
    ```

## Array
* Arrays are addressable multiplexers
* Can be indexed like Python Lists
* Error to modify the contents of the Array
  - Assignment to/from contents of Array is allowed
    ```python
    gpios = Array(Signal() for _ in range(10))
    with m.If(bus.we):
        m.d.sync += gpios[bus.addr].eq(bus.w_data)
    with m.Else():
        m.d.sync += bus.r_data.eq(gpios[bus.addr])
    ```
* Arrays can be multidimensional

## Memory
* Word addressable storage
* Parameterized at creation:
  ```python
  mem = Memory(
      width=32, # Bit Width of each address
      depth=64, # Number of memory addresses
      init=int_list, # Optional, default: [0....]
      name='optional_name', # Default: variable name
  )
  m.submodules["read_port"] = read_port = mem.read_port(transparent=False)
  m.submodules["write_port"] = write_port = mem.write_port()
  ```
* Read port usage:
  - `ReadPort.addr`: Signal for selections of address
  - `ReadPort.data`: Signal for data response
  - `ReadPort.en`: enable Signal (or Const)
* Write port usage :
  - `WritePort.addr`: Signal for selections of address
  - `WritePort.data`: Signal for data to write
  - `WritePort.en`: enable Signal (or Const)

## Record
* Acts as a convenient way to pass around related Signals (like a hardware `struct` from C)
* Can be instantiated from a `Layout` or nested list of iterators:
  ```python
  from nmigen.hdl.rec import Layout
  # ... 
  out_leds = Layout([
      ('rgb', 3),
      ('led_yellow', 1),
  ])
  self.o_0 = Record(out_leds, name='optional_name')
  self.o_1 = Record([
      ('rgb', 3),
      ('led_yellow', 1),
  ], name='optional_name')

  # self.o_0 and self.o_1 are functionally equivalent
  # self.o_0.rgb is a signal of shape `unsigned(3)` (as is self.o_1.rgb)
  ``` 

# Simulation and Verilog Generation

> Note: This section assumes you have `from nmigen.sim import *` at the top of your code.
  The simulator is NOT imported using the regular `from nmigen import *`.

## Simulator
* `Simulator` is the fundamental class for the nMigen simulation
* Instantiated with a module to be tested
  ```python
  dut = ModuleToBeTested() # dut is convention for "Device Under Test"
  sim = Simulator(dut)
  ```
* The Simulator runs (synchronously or asynchronously) generator functions that
verify signals from the module under test (more on that later)
  - Synchronous verification functions should be added to the
    simulator with `Simulator.add_sync_process(function_name)`
  - Asynchronous verification functions should be added to the
    simulator with `Simulator.add_process(function_name)`
* Clocks for domains (except the `comb` domain!) can be added with
  `Simulator.add_clock(period_in_secs, domain='name_of_domain')`
* The simulator can be run with `Simulator.run()`
  - Optionally, the simulator can generate vcd files (for viewing waveforms) if it
    is run under under a context manager
    ```python
    with sim.write_vcd('filename.vcd'):
        sim.run()
    ```

## Verification Functions
* In order to verify values, verification functions should be added as described above
* These functions must be generators, and `yield` is used to retrieve values during the simulation
  - `yield Tick('domain_name')` in a function will move the simulation forward one clock cycle
  - `yield Settle()` in a function will allow combinatorial to "settle" to their correct value (use before checking combinatorial signals)
  - `yield Delay(n)` in a function will wait `n` seconds in the simulator
  - Use `yield from` when calling helper functions in your verification function

## Example:
  ```python
  def helper_func(dut, in_0, in_1, expect_out):
      yield dut.input_0.eq(in_0)
      yield dut.input_1.eq(in_1)
      yield Tick('sync') # 'sync' not required, it's the default
      out = yield dut.output_0
      if (out != expect_out):
          print('Failed test!')
      else:
          print('Passed')

  def main_test(dut):
      yield from helper_func(dut, 1, 1, 0) # don't forget the 'yield from'
      yield from helper_func(dut, 0, 0, 0)
      return

  def main():
      dut = TestModule()
      sim = Simulator(dut)
      with sim.write_vcd('test.vcd'):
          sim.add_clock(1e-6)
          def proc():
              yield from main_test(dut)
          sim.add_sync_process(proc)
          sim.run()
  ```

## Generating Verilog
* nMigen can easily generate your design in Verilog
 ```python
 from nmigen.back import verilog
 m = TopModule()
 with open('Verilog_file.v', 'w') as f:
     f.write(verilog.convert(m, ports=[m.out, m.in_0, m.in_1]))
 ```
