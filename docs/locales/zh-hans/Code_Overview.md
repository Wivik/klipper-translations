# Code overview

本文档将描述Klipper的代码总体结构和代码流。

## 文件夹结构

**src/**包含微控制器的C源码。其中**src/atsam/**, **src/atsamd/**, **src/avr/**, **src/linux/**, **src/lpc176x/**, **src/pru/**, and **src/stm32/** 为对应微处理器架构的源码。 **src/simulator/** 包含有用于交叉编译、测试目标微处理器的代码。**src/generic/**为对不同架构均有用的代码。编译"board/somefile.h"时，编译器会优先使用 架构特定的目录 (即src/avr/somefile.h)随后找寻通用目录(即 src/generic/somefile.h)。

**klippy/**目录包含了上位机软件。软件大部分由Python实现，同时**klippy/chelper/** 目录包含了由C实现的有用代码。**klippy/kinematics/**目录包含机械运动学的实现代码。**klippy/extras/** 目录包含了上位机的扩建模块("modules")。

**lib/**包含了构建必须的第三方库代码。

**config/**包含了打印机配置的实例文件。

**scripts/**目录包含了编译微控制器代码时有用的脚本。

**test/**目录包含了自动测试示例。

在编译过程重，编译器会构建**out/**目录。该目录包含构建时的临时文件。对于AVR架构，编译器输出的为**out/klipper.elf.hex**，而对ARM架构则为**out/klipper.bin**。

## 微处理器的代码流

微控制器的代码从对应架构的代码(即**src/avr/main.c**)开始执行，前述代码会持续调用**src/sched.c**中的 sched_main() 函数。sched_main() 代码会先运行经 DECL_INIT() 宏标注的所有函数。之后它将不断重复运行由 DECL_TASK() 宏所标注的函数。

其中一个主要的任务函数为**src/command.c** 中的command_dispatch()。上述函数经由微处理器特定的 输入/输出 代码调用(即**src/avr/serial.c**, **src/generic/serial_irq.c**)，并执行输入流中的命令所对应的命令函数。命令函数通过 DECL_COMMAND() 宏进行定义 (详情参照[协议](Protocol.md) 文档)。

任务、初始化和命令函数总是在中断启用的情况下运行（然而，可根据需要将中断功能停用）。这些函数不应出现暂停、延迟或执行持续事件长于数微秒的任务。这些函数应由调度定时器在特定的事件进行调用。

Timer functions are scheduled by calling sched_add_timer() (located in **src/sched.c**). The scheduler code will arrange for the given function to be called at the requested clock time. Timer interrupts are initially handled in an architecture specific interrupt handler (eg, **src/avr/timer.c**) which calls sched_timer_dispatch() located in **src/sched.c**. The timer interrupt leads to execution of schedule timer functions. Timer functions always run with interrupts disabled. The timer functions should always complete within a few micro-seconds. At completion of the timer event, the function may choose to reschedule itself.

In the event an error is detected the code can invoke shutdown() (a macro which calls sched_shutdown() located in **src/sched.c**). Invoking shutdown() causes all functions tagged with the DECL_SHUTDOWN() macro to be run. Shutdown functions always run with interrupts disabled.

Much of the functionality of the micro-controller involves working with General-Purpose Input/Output pins (GPIO). In order to abstract the low-level architecture specific code from the high-level task code, all GPIO events are implemented in architecture specific wrappers (eg, **src/avr/gpio.c**). The code is compiled with gcc's "-flto -fwhole-program" optimization which does an excellent job of inlining functions across compilation units, so most of these tiny gpio functions are inlined into their callers, and there is no run-time cost to using them.

## 代码总览

上位机程序（klippy）运行在廉价计算机（如 树莓派）上，配搭mcu使用。该程序的主要编程语言为Python，同时部分功能通过CFFI在C语言上实现。

上位机程序通过** klippy/klippy.py**初始化。该文件会读取命令行参数，打开打印机的设置文件，实例化打印机的主要模块，并启用串口通讯。G代码命令的执行则通过 **klippy/gcode.py**中的 process_commands() 方法实现。此代码将G代码转化为打印机的对象调用，它将频繁地将G代码命令转化为微控制器的行动指令（通过微控制器代码中的 DECL_COMMAND 进行声明）。

Klippy上位机程序包含四个进程。主线程用于处理输入的G代码命令。第二线程通过串口实现底层IO的处理（代码位于 **klippy/chelper/serialqueue.c **以C语言实现）。第三线程则通过Python代码处理微控制器返回的信息（参照 klippy/serialhdl.py）。第四线程则负责将Debug信息写入到日志文件(见 **klippy/queuelogger.py**)，由此，其他线程的执行将不会阻塞日志的写入。

## 典型运动命令的代码流

典型的打印机运动始于klipper上位机接收到"G1"命令，并在微控制器发出对应的步进脉冲结束。本节将简述典型运动命令的代码流。[运动学](Kinematics.md)文档将更为细致的描述运动的机械原理。

* Processing for a move command starts in gcode.py. The goal of gcode.py is to translate G-code into internal calls. A G1 command will invoke cmd_G1() in klippy/extras/gcode_move.py. The gcode_move.py code handles changes in origin (eg, G92), changes in relative vs absolute positions (eg, G90), and unit changes (eg, F6000=100mm/s). The code path for a move is: `_process_data() -> _process_commands() -> cmd_G1()`. Ultimately the ToolHead class is invoked to execute the actual request: `cmd_G1() -> ToolHead.move()`
* The ToolHead class (in toolhead.py) handles "look-ahead" and tracks the timing of printing actions. The main codepath for a move is: `ToolHead.move() -> MoveQueue.add_move() -> MoveQueue.flush() -> Move.set_junction() -> ToolHead._process_moves()`.
   * ToolHead.move() creates a Move() object with the parameters of the move (in cartesian space and in units of seconds and millimeters).
   * The kinematics class is given the opportunity to audit each move (`ToolHead.move() -> kin.check_move()`). The kinematics classes are located in the klippy/kinematics/ directory. The check_move() code may raise an error if the move is not valid. If check_move() completes successfully then the underlying kinematics must be able to handle the move.
   * MoveQueue.add_move() places the move object on the "look-ahead" queue.
   * MoveQueue.flush() determines the start and end velocities of each move.
   * Move.set_junction() implements the "trapezoid generator" on a move. The "trapezoid generator" breaks every move into three parts: a constant acceleration phase, followed by a constant velocity phase, followed by a constant deceleration phase. Every move contains these three phases in this order, but some phases may be of zero duration.
   * When ToolHead._process_moves() is called, everything about the move is known - its start location, its end location, its acceleration, its start/cruising/end velocity, and distance traveled during acceleration/cruising/deceleration. All the information is stored in the Move() class and is in cartesian space in units of millimeters and seconds.
* Klipper uses an [iterative solver](https://en.wikipedia.org/wiki/Root-finding_algorithm) to generate the step times for each stepper. For efficiency reasons, the stepper pulse times are generated in C code. The moves are first placed on a "trapezoid motion queue": `ToolHead._process_moves() -> trapq_append()` (in klippy/chelper/trapq.c). The step times are then generated: `ToolHead._process_moves() -> ToolHead._update_move_time() -> MCU_Stepper.generate_steps() -> itersolve_generate_steps() -> itersolve_gen_steps_range()` (in klippy/chelper/itersolve.c). The goal of the iterative solver is to find step times given a function that calculates a stepper position from a time. This is done by repeatedly "guessing" various times until the stepper position formula returns the desired position of the next step on the stepper. The feedback produced from each guess is used to improve future guesses so that the process rapidly converges to the desired time. The kinematic stepper position formulas are located in the klippy/chelper/ directory (eg, kin_cart.c, kin_corexy.c, kin_delta.c, kin_extruder.c).
* Note that the extruder is handled in its own kinematic class: `ToolHead._process_moves() -> PrinterExtruder.move()`. Since the Move() class specifies the exact movement time and since step pulses are sent to the micro-controller with specific timing, stepper movements produced by the extruder class will be in sync with head movement even though the code is kept separate.
* After the iterative solver calculates the step times they are added to an array: `itersolve_gen_steps_range() -> stepcompress_append()` (in klippy/chelper/stepcompress.c). The array (struct stepcompress.queue) stores the corresponding micro-controller clock counter times for every step. Here the "micro-controller clock counter" value directly corresponds to the micro-controller's hardware counter - it is relative to when the micro-controller was last powered up.
* The next major step is to compress the steps: `stepcompress_flush() -> compress_bisect_add()` (in klippy/chelper/stepcompress.c). This code generates and encodes a series of micro-controller "queue_step" commands that correspond to the list of stepper step times built in the previous stage. These "queue_step" commands are then queued, prioritized, and sent to the micro-controller (via stepcompress.c:steppersync and serialqueue.c:serialqueue).
* Processing of the queue_step commands on the micro-controller starts in src/command.c which parses the command and calls `command_queue_step()`. The command_queue_step() code (in src/stepper.c) just appends the parameters of each queue_step command to a per stepper queue. Under normal operation the queue_step command is parsed and queued at least 100ms before the time of its first step. Finally, the generation of stepper events is done in `stepper_event()`. It's called from the hardware timer interrupt at the scheduled time of the first step. The stepper_event() code generates a step pulse and then reschedules itself to run at the time of the next step pulse for the given queue_step parameters. The parameters for each queue_step command are "interval", "count", and "add". At a high-level, stepper_event() runs the following, 'count' times: `do_step(); next_wake_time = last_wake_time + interval; interval += add;`

The above may seem like a lot of complexity to execute a movement. However, the only really interesting parts are in the ToolHead and kinematic classes. It's this part of the code which specifies the movements and their timings. The remaining parts of the processing is mostly just communication and plumbing.

## 添加上位机模块

The Klippy host code has a dynamic module loading capability. If a config section named "[my_module]" is found in the printer config file then the software will automatically attempt to load the python module klippy/extras/my_module.py . This module system is the preferred method for adding new functionality to Klipper.

The easiest way to add a new module is to use an existing module as a reference - see **klippy/extras/servo.py** as an example.

The following may also be useful:

* Execution of the module starts in the module level `load_config()` function (for config sections of the form [my_module]) or in `load_config_prefix()` (for config sections of the form [my_module my_name]). This function is passed a "config" object and it must return a new "printer object" associated with the given config section.
* During the process of instantiating a new printer object, the config object can be used to read parameters from the given config section. This is done using `config.get()`, `config.getfloat()`, `config.getint()`, etc. methods. Be sure to read all values from the config during the construction of the printer object - if the user specifies a config parameter that is not read during this phase then it will be assumed it is a typo in the config and an error will be raised.
* Use the `config.get_printer()` method to obtain a reference to the main "printer" class. This "printer" class stores references to all the "printer objects" that have been instantiated. Use the `printer.lookup_object()` method to find references to other printer objects. Almost all functionality (even core kinematic modules) are encapsulated in one of these printer objects. Note, though, that when a new module is instantiated, not all other printer objects will have been instantiated. The "gcode" and "pins" modules will always be available, but for other modules it is a good idea to defer the lookup.
* Register event handlers using the `printer.register_event_handler()` method if the code needs to be called during "events" raised by other printer objects. Each event name is a string, and by convention it is the name of the main source module that raises the event along with a short name for the action that is occurring (eg, "klippy:connect"). The parameters passed to each event handler are specific to the given event (as are exception handling and execution context). Two common startup events are:
   * klippy:connect - This event is generated after all printer objects are instantiated. It is commonly used to lookup other printer objects, to verify config settings, and to perform an initial "handshake" with printer hardware.
   * klippy:ready - This event is generated after all connect handlers have completed successfully. It indicates the printer is transitioning to a state ready to handle normal operations. Do not raise an error in this callback.
* If there is an error in the user's config, be sure to raise it during the `load_config()` or "connect event" phases. Use either `raise config.error("my error")` or `raise printer.config_error("my error")` to report the error.
* Use the "pins" module to configure a pin on a micro-controller. This is typically done with something similar to `printer.lookup_object("pins").setup_pin("pwm", config.get("my_pin"))`. The returned object can then be commanded at run-time.
* If the module needs access to system timing or external file descriptors then use `printer.get_reactor()` to obtain access to the global "event reactor" class. This reactor class allows one to schedule timers, wait for input on file descriptors, and to "sleep" the host code.
* Do not use global variables. All state should be stored in the printer object returned from the `load_config()` function. This is important as otherwise the RESTART command may not perform as expected. Also, for similar reasons, if any external files (or sockets) are opened then be sure to register a "klippy:disconnect" event handler and close them from that callback.
* Avoid accessing the internal member variables (or calling methods that start with an underscore) of other printer objects. Observing this convention makes it easier to manage future changes.
* If submitting the module for inclusion in the main Klipper code, be sure to place a copyright notice at the top of the module. See the existing modules for the preferred format.

## Adding new kinematics

This section provides some tips on adding support to Klipper for additional types of printer kinematics. This type of activity requires excellent understanding of the math formulas for the target kinematics. It also requires software development skills - though one should only need to update the host software.

Useful steps:

1. Start by studying the "[code flow of a move](#code-flow-of-a-move-command)" section and the [Kinematics document](Kinematics.md).
1. Review the existing kinematic classes in the klippy/kinematics/ directory. The kinematic classes are tasked with converting a move in cartesian coordinates to the movement on each stepper. One should be able to copy one of these files as a starting point.
1. Implement the C stepper kinematic position functions for each stepper if they are not already available (see kin_cart.c, kin_corexy.c, and kin_delta.c in klippy/chelper/). The function should call `move_get_coord()` to convert a given move time (in seconds) to a cartesian coordinate (in millimeters), and then calculate the desired stepper position (in millimeters) from that cartesian coordinate.
1. Implement the `calc_position()` method in the new kinematics class. This method calculates the position of the toolhead in cartesian coordinates from the position of each stepper. It does not need to be efficient as it is typically only called during homing and probing operations.
1. Other methods. Implement the `check_move()`, `get_status()`, `get_steppers()`, `home()`, and `set_position()` methods. These functions are typically used to provide kinematic specific checks. However, at the start of development one can use boiler-plate code here.
1. Implement test cases. Create a g-code file with a series of moves that can test important cases for the given kinematics. Follow the [debugging documentation](Debugging.md) to convert this g-code file to micro-controller commands. This is useful to exercise corner cases and to check for regressions.

## Porting to a new micro-controller

This section provides some tips on porting Klipper's micro-controller code to a new architecture. This type of activity requires good knowledge of embedded development and hands-on access to the target micro-controller.

Useful steps:

1. Start by identifying any 3rd party libraries that will be used during the port. Common examples include "CMSIS" wrappers and manufacturer "HAL" libraries. All 3rd party code needs to be GNU GPLv3 compatible. The 3rd party code should be committed to the Klipper lib/ directory. Update the lib/README file with information on where and when the library was obtained. It is preferable to copy the code into the Klipper repository unchanged, but if any changes are required then those changes should be listed explicitly in the lib/README file.
1. Create a new architecture sub-directory in the src/ directory and add initial Kconfig and Makefile support. Use the existing architectures as a guide. The src/simulator provides a basic example of a minimum starting point.
1. The first main coding task is to bring up communication support to the target board. This is the most difficult step in a new port. Once basic communication is working, the remaining steps tend to be much easier. It is typical to use an RS-232 style serial port during initial development as these types of hardware devices are generally easier to enable and control. During this phase, make liberal use of helper code from the src/generic/ directory (check how src/simulator/Makefile includes the generic C code into the build). It is also necessary to define timer_read_time() (which returns the current system clock) in this phase, but it is not necessary to fully support timer irq handling.
1. Get familiar with the the console.py tool (as described in the [debugging document](Debugging.md)) and verify connectivity to the micro-controller with it. This tool translates the low-level micro-controller communication protocol to a human readable form.
1. Add support for timer dispatch from hardware interrupts. See Klipper [commit 970831ee](https://github.com/KevinOConnor/klipper/commit/970831ee0d3b91897196e92270d98b2a3067427f) as an example of steps 1-5 done for the LPC176x architecture.
1. Bring up basic GPIO input and output support. See Klipper [commit c78b9076](https://github.com/KevinOConnor/klipper/commit/c78b90767f19c9e8510c3155b89fb7ad64ca3c54) as an example of this.
1. Bring up additional peripherals - for example see Klipper commit [65613aed](https://github.com/KevinOConnor/klipper/commit/65613aeddfb9ef86905cb1dade9e773a02ef3c27), [c812a40a](https://github.com/KevinOConnor/klipper/commit/c812a40a3782415e454b04bf7bd2158a6f0ec8b5), and [c381d03a](https://github.com/KevinOConnor/klipper/commit/c381d03aad5c3ee761169b7c7bced519cc14da29).
1. Create a sample Klipper config file in the config/ directory. Test the micro-controller with the main klippy.py program.
1. Consider adding build test cases in the test/ directory.

## Coordinate Systems

Internally, Klipper primarily tracks the position of the toolhead in cartesian coordinates that are relative to the coordinate system specified in the config file. That is, most of the Klipper code will never experience a change in coordinate systems. If the user makes a request to change the origin (eg, a `G92` command) then that effect is obtained by translating future commands to the primary coordinate system.

However, in some cases it is useful to obtain the toolhead position in some other coordinate system and Klipper has several tools to facilitate that. This can be seen by running the GET_POSITION command. For example:

```
Send: GET_POSITION
Recv: // mcu: stepper_a:-2060 stepper_b:-1169 stepper_c:-1613
Recv: // stepper: stepper_a:457.254159 stepper_b:466.085669 stepper_c:465.382132
Recv: // kinematic: X:8.339144 Y:-3.131558 Z:233.347121
Recv: // toolhead: X:8.338078 Y:-3.123175 Z:233.347878 E:0.000000
Recv: // gcode: X:8.338078 Y:-3.123175 Z:233.347878 E:0.000000
Recv: // gcode base: X:0.000000 Y:0.000000 Z:0.000000 E:0.000000
Recv: // gcode homing: X:0.000000 Y:0.000000 Z:0.000000
```

The "mcu" position (`stepper.get_mcu_position()` in the code) is the total number of steps the micro-controller has issued in a positive direction minus the number of steps issued in a negative direction since the micro-controller was last reset. The value reported is only valid after the stepper has been homed. If the robot is in motion when the query is issued then the reported value includes moves buffered on the micro-controller, but does not include moves on the look-ahead queue.

The "stepper" position (`stepper.get_commanded_position()`) is the position of the given stepper as tracked by the kinematics code. This generally corresponds to the position (in mm) of the carriage along its rail, relative to the position_endstop specified in the config file. (Some kinematics track stepper positions in radians instead of millimeters.) If the robot is in motion when the query is issued then the reported value includes moves buffered on the micro-controller, but does not include moves on the look-ahead queue. One may use the `toolhead.flush_step_generation()` or `toolhead.wait_moves()` calls to fully flush the look-ahead and step generation code.

The "kinematic" position (`kin.calc_position()`) is the cartesian position of the toolhead as derived from "stepper" positions and is relative to the coordinate system specified in the config file. This may differ from the requested cartesian position due to the granularity of the stepper motors. If the robot is in motion when the "stepper" positions are taken then the reported value includes moves buffered on the micro-controller, but does not include moves on the look-ahead queue. One may use the `toolhead.flush_step_generation()` or `toolhead.wait_moves()` calls to fully flush the look-ahead and step generation code.

The "toolhead" position (`toolhead.get_position()`) is the last requested position of the toolhead in cartesian coordinates relative to the coordinate system specified in the config file. If the robot is in motion when the query is issued then the reported value includes all requested moves (even those in buffers waiting to be issued to the stepper motor drivers).

The "gcode" position is the last requested position from a `G1` (or `G0`) command in cartesian coordinates relative to the coordinate system specified in the config file. This may differ from the "toolhead" position if a g-code transformation (eg, bed_mesh, bed_tilt, skew_correction) is in effect. This may differ from the actual coordinates specified in the last `G1` command if the g-code origin has been changed (eg, `G92`, `SET_GCODE_OFFSET`, `M221`). The `M114` command (`gcode_move.get_status()['gcode_position']`) will report the last g-code position relative to the current g-code coordinate system.

The "gcode base" is the location of the g-code origin in cartesian coordinates relative to the coordinate system specified in the config file. Commands such as `G92`, `SET_GCODE_OFFSET`, and `M221` alter this value.

The "gcode homing" is the location to use for the g-code origin (in cartesian coordinates relative to the coordinate system specified in the config file) after a `G28` home command. The `SET_GCODE_OFFSET` command can alter this value.

## Time

Fundamental to the operation of Klipper is the handling of clocks, times, and timestamps. Klipper executes actions on the printer by scheduling events to occur in the near future. For example, to turn on a fan, the code might schedule a change to a GPIO pin in a 100ms. It is rare for the code to attempt to take an instantaneous action. Thus, the handling of time within Klipper is critical to correct operation.

There are three types of times tracked internally in the Klipper host software:

* System time. The system time uses the system's monotonic clock - it is a floating point number stored as seconds and it is (generally) relative to when the host computer was last started. System times have limited use in the software - they are primarily used when interacting with the operating system. Within the host code, system times are frequently stored in variables named *eventtime* or *curtime*.
* Print time. The print time is synchronized to the main micro-controller clock (the micro-controller defined in the "[mcu]" config section). It is a floating point number stored as seconds and is relative to when the main mcu was last restarted. It is possible to convert from a "print time" to the main micro-controller's hardware clock by multiplying the print time by the mcu's statically configured frequency rate. The high-level host code uses print times to calculate almost all physical actions (eg, head movement, heater changes, etc.). Within the host code, print times are generally stored in variables named *print_time* or *move_time*.
* MCU clock. This is the hardware clock counter on each micro-controller. It is stored as an integer and its update rate is relative to the frequency of the given micro-controller. The host software translates its internal times to clocks before transmission to the mcu. The mcu code only ever tracks time in clock ticks. Within the host code, clock values are tracked as 64bit integers, while the mcu code uses 32bit integers. Within the host code, clocks are generally stored in variables with names containing *clock* or *ticks*.

Conversion between the different time formats is primarily implemented in the **klippy/clocksync.py** code.

Some things to be aware of when reviewing the code:

* 32bit and 64bit clocks: To reduce bandwidth and to improve micro-controller efficiency, clocks on the micro-controller are tracked as 32bit integers. When comparing two clocks in the mcu code, the `timer_is_before()` function must always be used to ensure integer rollovers are handled properly. The host software converts 32bit clocks to 64bit clocks by appending the high-order bits from the last mcu timestamp it has received - no message from the mcu is ever more than 2^31 clock ticks in the future or past so this conversion is never ambiguous. The host converts from 64bit clocks to 32bit clocks by simply truncating the high-order bits. To ensure there is no ambiguity in this conversion, the **klippy/chelper/serialqueue.c** code will buffer messages until they are within 2^31 clock ticks of their target time.
* Multiple micro-controllers: The host software supports using multiple micro-controllers on a single printer. In this case, the "MCU clock" of each micro-controller is tracked separately. The clocksync.py code handles clock drift between micro-controllers by modifying the way it converts from "print time" to "MCU clock". On secondary mcus, the mcu frequency that is used in this conversion is regularly updated to account for measured drift.
