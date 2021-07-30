A Lingua Franca file, which has a .lf extension, contains the following:

- One [**target** specification](target-specification).
- Zero or more [**import** statements](#import-statement).
- One or more [**reactor**](#reactor-block) blocks, which contain [**reaction** declarations](#reaction-declaration).

If one of the reactors in the file is designated `main` or `federated`, then the file defines an executable application. Otherwise, it defines one or more library reactors that can be imported into other LF files. For example, an LF file might be structured like this:

```
target C;
main reactor C {
    a = new A();
    b = new B();
    a.y -> b.x;
}
reactor A {
    output y;
    ...
}
reactor B {
    input x;
    ...
}
```
The name of the main reactor (`C` above) is optional. If given, it must match the filename (`C.lf` in the above example).

This example specifies and instantiates two reactors, one of which sends messages to the other. A minimal but complete Lingua Franca file with one reactor is this:

```
target C; 
main reactor HelloWorld {
    reaction(startup) {=
        printf("Hello World.\n");
    =}
}
```

See the [C target documentation](Writing-Reactors-in-C#a-minimal-example) for details about this example.

## Target Specification

Every Lingua Franca program begins with a [[target specification]] that specifies the language in which reactions are written. This is also the language of the program(s) generated by the Lingua Franca compiler.

## Import Statement

An import statement has the form:

> **import** { *reactor1*, *reactor2* **as** *alias2*, [...] } **from** "*path*";

where *path* specifies another Lingua Franca file relative to the location of the current file.

## Reactor Block

A **reactor** is a software component that reacts to input events, timer events, and internal events. It has private state variables that are not visible to any other reactor. Its reactions can consist of altering its own state, sending messages to other reactors, or affecting the environment through some kind of actuation or side effect.

The general structure of a reactor block is as follows:

> **reactor** *name* (*[parameters](#parameter-declaration)*) {<br/>
> &nbsp;&nbsp;*[state declarations](#state-declaration)*<br/>
> &nbsp;&nbsp;*[method declarations](#method-declaration)*<br/>
> &nbsp;&nbsp;*[input declarations](#input-declaration)*<br/>
> &nbsp;&nbsp;*[output declarations](#output-declaration)*<br/>
> &nbsp;&nbsp;*[timer declarations](#timer-declaration)*<br/>
> &nbsp;&nbsp;*[action declarations](#action-declaration)*<br/>
> &nbsp;&nbsp;*[reaction declarations](#reaction-declaration)*<br/>
> &nbsp;&nbsp;*[contained reactors](#contained-reactors)*<br/>
> &nbsp;&nbsp; ... <br/>
> }

Parameter, inputs, outputs, timers, actions, and contained reactors all have names, and the names are required to be distinct from one another. 

If the **reactor** keyword is preceded by **main**,  then this reactor will be instantiated and run by the generated code. If an imported LF file contains a main reactor, that reactor is ignored. Only reactors that not designated `main` are imported. This makes it easy to create a library of reusable reactors that each come with a test case or demonstration in the form of a main reactor.

### Parameter Declaration

A reactor class definition can define parameters as follows:

> **reactor** *ClassName*(*paramName1*:*type*(*value*), *paramName2*:*type*(*value*)) {<br/>
> &nbsp;&nbsp; ... <br/>
> }

The *type* of the parameter is a type in the target language, with one exception, the **time** type. If the type is **time**, then the *value* is either `0` or a number annotated with time units. For example,

```
reactor Foo(period:time(100 msec)) {
    ...
}
```

The following units are allowed:

- **nsec**: nanoseconds
- **usec**: microseconds
- **msec**: milliseconds
- **sec** or **secs**: seconds
- **minute** or **minutes**: 60 seconds
- **hour** or **hours**: 60 minutes
- **day** or **days**: 24 hours
- **week** or **weeks**: 7 days

Lingua Franca also provides a convenient syntax for initializing arrays and lists. If the type in the target language is an array, vector, or list of some sort, then its initial value can be given as a list of values. For example, in the C target, you can initialize an array parameter as follows:
```
reactor Foo(my_array:int[](1, 2, 3)) {
   ...
}
```
For all types other than **time** and arrays, if the type designation is a simple identifier string, like 'int' or 'double', then it can be given simply using that string. For example:

```
reactor Foo(size:int(100)) {
    ...
}
```

If, however, the type requires syntax special to the target language, then the type designator must be enclosed in the target-code delimiters `{= ... =}`. For example, the following can be used for the C target language to declare a parameter with a string type:

```
reactor Foo(message:{=char*=}("Hello World")) {
    ...
}
```

Because of the awkwardness of this specification, however, the Lingua Franca C target defines a `string` datatype that is equivalent to `char*`, so you can write instead:
```
reactor Foo(message:string("Hello World")) {
    ...
}
```
(Types ending with a `*` are treated specially by the C target. See [Sending and Receiving Arrays and Structs](Writing-Reactors-in-C#sending-and-receiving-arrays-and-structs) in the C target documentation.)
Similarly, if the value requires particular syntax from the target language, then it should also be enclosed in target-code delimiters `{= ... =}`. For example, to specify an array-valued parameter in the C target, you can write:

```
reactor Foo(param:int[]({= {1, 2, 3, 4} =})) {
    ...
}
```
The datatype here is `int[]`, an array of integers of unspecified length, and the default value `{1, 2, 3, 4}` is C syntax for a static initializer for an array of length 4. Again, however, this syntax is awkward, and the pattern is rather common. Lingua Franca supports a **list** type for initializers, so the above can be written more cleanly as:
```
reactor Foo(param:int[](1, 2, 3, 4)) {
    ...
}
```
The code generator will convert this into a C static initializer for the array.

How a parameter is used in the body of a reaction depends on the target. For example, in the [C target](writing-reactors-in-c#using-parameters), a `self` struct is provided that contains the parameter values. The following example illustrates this:

```
target C;
reactor Gain(scale:int(2)) {
    input x:int;
    output y:int;
    reaction(x) -> y {=
        SET(y, x->value * self->scale);
    =}
}
```

This reactor, given any input event `x` will produce an output `y` with value equal to the input scaled by the `scale` parameter. The default value of the `scale` parameter is 2, but this can be changed when the `Gain` reactor is [instantiated](#contained-reactors). The `SET()` is the mechanism provided by the [C target](Writing-Reactors-in-C#reaction-body) for setting the value of outputs. The parameter `scale` and input `x` are just referenced in the C code as shown above.

Note that in the C++ target, curly braces `{...}` can be used instead of the paranethesis `(...)` to initialize parameters. See the [C++ documentation for parameters](https://github.com/icyphy/lingua-franca/wiki/Writing-Reactors-in-Cpp#using-parameters).

### State Declaration

A state declaration has one of the forms:

> **state** *name*:*type*(*initial_value*);  
> **state** *name*(*parameter*);  
> **state** *name*:**time**(*number* *units*);  

In the second form, the state variable inherits its type from the specified *parameter*, which also gives the initial value of the state variable. In the third form, the state variable represents a time, and the meaning of the *number* and *units* is the same as for parameters.

How a parameter is used in the body of a reaction depends on the target. For example, in the [C target](writing-reactors-in-c#using-state-variables), a `self` struct is provided that contains the state values. The following example illustrates this:

```
reactor Count {
	output c:int;
	timer t(0, 1 sec);
	state i:int(0);
	reaction(t) -> c {=
		(self->i)++;
		SET(c, self->i);
	=}
}
```

Note that in the C++ target, curly braces `{...}` can be used instead of the paranethesis `(...)` to specify the inital value. See the [C++ documentation for state variables](https://github.com/icyphy/lingua-franca/wiki/Writing-Reactors-in-Cpp#using-state-variables).

### Method Declaration

A method declaration has one of the forms:

> **method** *name*();  
> **method** *name*():*type*;  
> **method** *name*(*arg1_name*:arg1_type, *arg2_name*:arg2_type, ...);
> **method** *name*(*arg1_name*:arg1_type, *arg2_name*:arg2_type, ...):*type*;  

The first form defines a method with no arguments and no return value. The second form defines a method with the return type *type* but no arguments. The third form defines a method with arguments given by their name and type, but without a return value. Finally, the fourth form is similar to the third, but adds a return type.

The **method** keywork can optionally be prefixed with the **const** qualifier, which indicates that the method is "read-only". This is relvant for some target languages such as C++.

See the [C++ documentation](https://github.com/icyphy/lingua-franca/wiki/Writing-Reactors-in-Cpp#using-methods) for a usage example.

### Input Declaration

An input declaration has the form:

> **input** *name*:*type*;

The `Gain` reactor given above provides an example. The *type* is just like parameter types.

An input may have the modifier **mutable**, as follows:

> **mutable input** *name*:*type*

This is a directive to the code generator indicating that reactions that read this input will also modify the value of the input. Without this modifier, inputs are **immutable**; modifying them is disallowed.  The precise mechanism for making use of mutable inputs is target-language specific. See, for example, the [C language target](writing-reactors-in-c#Sending-and-Receiving-Arrays-and-Structs).

An input port may have more than one **channel**. See [multiports documentation](Multiports-and-Banks-of-Reactors#multiports).

### Output Declaration

An output declaration has the form:

> **output** *name*:*type*;

The `Gain` reactor given above provides an example. The *type* is just like parameter types.

An output port may have more than one **channel**. See [multiports documentation](Multiports-and-Banks-of-Reactors#multiports).

### Timer Declaration

A timer, like an input and an action, causes reactions to be invoked. Unlike an action, it is triggered automatically by the scheduler. This declaration is used when you want to invoke reactions once at specific times or periodically. A timer declaration has the form:


> **timer** *name*(*offset*, *period*);

For example,

```
timer foo(10 msec, 100 msec);
```

This specifies a timer named `foo` that will first trigger 10 milliseconds after the start of execution and then repeatedly trigger at intervals of 100 ms.  The units are optional, and if they are not included, then the number will be interpreted in a target-dependent way.  The units supported are the same as in [parameter declarations](#parameter-declaration) described above.

The times specified are logical times. Specifically, if two timers have the same *offset* and *period*, then they are logically simultaneous.  No observer will be able to see that one timer has triggered and the other has not. Even though these are logical times, the runtime system will make an effort to align those times to physical times. Such alignment can never be perfect, and its accuracy will depend on the execution platform.

Both arguments are optional, with both having default value zero. An *offset* of zero or greater specifies the minimum time delay between the time at the start of execution and when the action is triggered. The *period* is zero or greater, where a value of zero specifies that the reactions should be triggered exactly once,
whereas a value greater than zero specifies that they should be triggered repeatedly with the period given.

To cause a reaction to be invoked at the start of execution, a special **startup** trigger is provided:

```
reactor Foo {
    reaction(startup) {=
        ... perform initialization ...
    =}
}
```
The **startup** trigger is equivalent to a timer with no *offset* or *period*.

### Action Declaration

An **action**, like an input, can cause reactions to be invoked. Whereas inputs are provided by other reactors, actions are scheduled by this reactor itself, either in response to some observed external event or as a delayed response to some input event. The action can be scheduled by a reactor by invoking a [**schedule** function](#scheduling-future-reactions) in a reaction or in an asynchronous callback function.

An action declaration is either physical or logical:

> **physical action** *name*(*min_delay*, *min_spacing*, *policy*):*type*;<br>
> **logical action** *name*(*min_delay*, *min_spacing*, *policy*):*type*;<br>

The *min_delay*, *min_spacing*, and *policy* are all optional. If only one argument is given in parentheses, then it is interpreted as an *min_delay*, if two are given, then they are interpreted as *min_delay* and *min_spacing*, etc. The *min_delay* and *min_spacing* have to be a time value. The *policy* argument is a string that can be one of the following: `'defer'` (default), `'drop'`, or `'replace'`.

An action will trigger at a logical time that depends on the arguments given to the schedule function, the *min_delay*, *min_spacing*, and *policy* arguments above, and whether the action is physical or logical.

If the **logical** keyword is given, then the tag assigned to the event resulting from a call to [**schedule** function](#scheduling-future-reactions) is computed as follows. First, let *t* be the *current logical time*. For a logical action, the `schedule` function must be invoked from within a reaction (synchronously), so *t* is just the logical time of that reaction.

The (preliminary) tag of the action is then just *t* plus *min_delay* plus the *offset* argument to [**schedule** function](#scheduling-future-reactions). 

If the **physical** keyword is given, then the physical clock on the local platform is used as the timestamp assigned to the action. Moreover, for a physical action, unlike a logical action, the `schedule` function can only be invoked from outside of any reaction (asynchronously), e.g. from an interrupt service routine or callback function.

If a *min_spacing* has been declared, then a minimum distance between the tags of two subsequently scheduled events on the same action is enforced. If the preliminary tag is closer to the tag of the previously scheduled event (if there is one), then *policy* determines how the given constraints is enforced.
* `'drop'`: the new event is dropped and `schedule` returns without having modified the event queue.
* `'replace'`: the payload of the new event is assigned to the preceding event if it is still pending in the event queue; no new event is added to the event queue in this case. If the preceding event has already been pulled from the event queue, the default `'defer'` policy is applied.
* `'defer'`: the event is added to the event queue with a tag that is equal to earliest time that satisfies the minimal spacing requirement. Assuming the tag of the preceding event is *t_prev*, then the tag of the new event simply becomes *t_prev* + *min_spacing*.

Note that while the `'defer'` policy is conservative in the sense that it does not discard events, it could potentially cause an unbounded growth of the event queue.

In all cases, the logical time of a new event will always be strictly greater than the logical time at which it is scheduled by at least one microstep (see the [Time](#Time) section).

The default *min_delay* is zero. The default *min_spacing* is undefined (meaning that no minimum spacing constraint is enforced). If a `min_spacing` is defined, it has to be strictly greater than zero, and greater than or equal to the time precision of the target (for the C target, it is one nanosecond).

The *min_delay* parameter in the **action** declaration is static (set at compile time), while the *offset* parameter given to the schedule function may be dynamically set at runtime. Hence, for static analysis and scheduling, the **action**'s' *min_delay* parameter can be assumed to be a *minimum delay* for analysis purposes.

#### Discussion
Logical actions are used to schedule events at a future logical time relative to the current logical time. Physical time is ignored. They must be scheduled within reactions, and the timestamp of the scheduled event will be relative to the current logical time of the reaction that schedules them. It is an error to schedule a logical action asynchronously, outside of the context of a reaction. Asynchronous actions are required to be **physical**.

Physical actions are typically used to assign timestamps to externally triggered events, such as the arrival of a network message or the acquisition of sensor data, where the time at which these external events occurs is of interest. There are (at least) three interesting use cases:
1. An asynchronous event, such as a callback function or interrupt service routine (ISR), is invoked at a physical time *t* and schedules an action with timestamp *T*=*t*.  To get this behavior, just set the physical action to have *min_delay* = 0 and call the schedule function  with *offset* = 0. The *min_spacing* can be useful here to prevent these external events from overwhelming the software system.
2. A periodic task that is occasionally modified by a sporadic sensor. In this case, you can set  *min_delay* = *period* and call schedule with *offset* = 0. The resulting timestamp of the sporadic sensor event will always align with the periodic events. This is similar to periodic polling, but without the overhead of polling the sensor when nothing interesting is happening.
3. You can impose a minimum physical time delay between an event's occurrence, such as a push of a button, and system response by adjusting the *offset*.

### Actions With Values

If an action is declared with a *type*, then it can carry a **value**, a data value passed to the **schedule** function. This value will be available to any reaction that is triggered by the action. The specific mechanism, however, is target-language dependent. See the [C target](Writing-reactors-in-C#actions-with-values) for an example.

## Reaction Declaration

A reaction is defined within a reactor using the following syntax:

> **reaction**(*triggers*) *uses* -> *effects* {=<br/>
> &nbsp;&nbsp; ... target language code ... <br/>
> =}

The *uses* and *effects* fields are optional. A simple example appears in the "hello world" example given above:

```
    reaction(t) {=
        printf("Hello World.\n");
    =}
```

In this example, `t` is a **trigger** (a timer named `t`). When that timer fires, the reaction will be invoked. Triggers can be [timers](#timer-declaration), [inputs](#input-declaration), [outputs](#output-declaration) of contained reactors, or [actions](#action-declaration). A comma-separated list of triggers can be given, in which case any of the specified triggers can trigger the reaction. If, at any logical time instant, more than one of the triggers fires, the reaction will nevertheless be invoked only once. 

The *uses* field specifies [inputs](#input-declaration) that the reaction observes but that do not trigger the reaction. This field can also be a comma-separated list of inputs. Since the input does not trigger the reaction, the body of the reaction will normally need to test for presence of the input before using it. How to do this is target specific. See [how this is done in the C target](Writing-Reactors-in-C#Inputs-and-Outputs).

The *effects* field, occurring after the right arrow, declares which [outputs](#output-declaration) and [actions](#action-declaration) the target code *may* produce or schedule. The *effects* field may also specify [inputs](#input-declaration) of contained reactors, provided that those inputs do not have any other sources of data. These declarations make it *possible* for the reaction to send outputs or enable future actions, but they do not *require* that the reaction code do that.

### Target Code

The body of the reaction is code in the target language surrounded by `{=` and `=}`. This code is not parsed by the Lingua Franca compiler. It is used verbatim in the program that is generated.

The target provides language-dependent mechanisms for referring to inputs, outputs, and actions in the target code. These mechanisms can be different in each target language, but all target languages provide the same basic set of mechanisms. These mechanisms include:

- Obtaining the current logical time (logical time does not advance during the execution of a reaction, so the execution of a reaction is logically instantaneous).
- Determining whether inputs are present at the current logical time and reading their value if they are. If a reaction is triggered by exactly one input, then that input will always be present. But if there are multiple triggers, or if the input is specified in the *uses* field, then the input may not be present when the reaction is invoked.
- Setting output values. Reactions in a reactor may set an output value more than once at any instant of logical time, but only the last of the values set will be sent on the output port.
- Scheduling future actions.

In the [C target](Writing-Reactors-in-C#Reaction-Body), for example, the following reactor will add two inputs if they are present at the time of a reaction:
```
reactor Add {
    input in1:int;
    input in2:int;
    output out:int;
    reaction(in1, in2) -> out {=
        int result = 0;
        if (in1->is_present) result += in1->value;
        if (in2->is_present) result += in2->value;
        SET(out, result);
    =}
}
```

See the [C target](Writing-Reactors-in-C#Reaction-Body) for an example of how these things are specified in C.

**NOTE:** if a reaction fails to test for the presence of an input and reads its value anyway, then the result it will get is undefined and may be target dependent. In the C target, as of this writing, the value read will be the most recently seen input value, or, if no input event has occurred at an earlier logical time, then zero or NULL, depending on the datatype of the input. In the TS target, the value will be **undefined**, a legitimate value in TypeScript.

### Scheduling Future Reactions

Each target language provides some mechanism for scheduling future reactions. Typically, this takes the form of a ```schedule``` function that takes as an argument an [action](#Action-Declaration), a time interval, and (perhaps optionally), a payload.  For example, in the [C target](Writing-Reactors-in-C#Reaction-Body), in the following program, each reaction to the timer ```t``` schedules another reaction to occur 100 msec later:

```
target C;
main reactor Schedule {
    timer t(0, 1 sec);
    logical action a;
    reaction(t) -> a {=
        schedule(a, MSEC(100));
    =}
    reaction(a) {=
        printf("Nanoseconds since start: %lld.\n", get_elapsed_logical_time());
    =}
}
```
When executed, this will produce the following output:
```
Start execution at time Sun Aug 11 04:11:57 2019
plus 919310000 nanoseconds.
Nanoseconds since start: 100000000.
Nanoseconds since start: 1100000000.
Nanoseconds since start: 2100000000.
...
```
This action has no datatype and carries no value, but, as explained below, an action can carry a value.

### Asynchronous Callbacks

In targets that support multitasking, the ```schedule``` function, which schedules future reactions, may be safely invoked on a **physical action** in code that is not part of a reaction. For example, in the multithreaded version of the  [C target](Writing-Reactors-in-C#Reaction-Body), ```schedule``` may be invoked in an interrupt service routine. The reaction(s) that are scheduled are guaranteed to occur at a time that is strictly larger than the current logical time of any reactions that are being interrupted.

### Superdense Time

Lingua Franca uses a concept known as **superdense time**, where two time values that appear to be the same are not logically simultaneous. At every logical time value, for example midnight on January 1, 1970, there exist a logical sequence of **microsteps** that are not simultaneous. The [Microsteps](https://github.com/icyphy/lingua-franca/blob/master/test/C/src/Microsteps.lf) example illustrates this:
```
target C;
reactor Destination {
    input x:int;
    input y:int;
    reaction(x, y) {=
        printf("Time since start: %lld.\n", get_elapsed_logical_time());
        if (x->is_present) {
            printf("  x is present.\n");
        }
        if (y->is_present) {
            printf("  y is present.\n");
        }
    =}
}
main reactor Microsteps {
    timer start;
    logical action repeat;
    d = new Destination();
    reaction(start) -> d.x, repeat {=
        SET(d.x, 1);
        schedule(repeat, 0);
    =}
    reaction(repeat) -> d.y {=
        SET(d.y, 1);
    =}
}
```
The ```Destination``` reactor has two inputs, ```x``` and ```y```, and it simply reports at each logical time where either is present what is the logical time and which is present.  The ```Microsteps``` reactor initializes things with a reaction to the one-time timer event ```start``` by sending data to the ```x``` input of ```Destination```. It then schedules a ```repeat``` action.

Note that time delay in the call to ```schedule``` is zero. However, any reaction scheduled by ```schedule``` is required to occur **strictly later** than current logical time. In Lingua Franca, this is handled by scheduling the ```repeat``` reaction to occur one **microstep** later. The output printed, therefore, will look like this:

```
Time since start: 0.
  x is present.
Time since start: 0.
  y is present.
```
Note that the numerical time reported by ```get_elapsed_logical_time()``` has not advanced in the second reaction, but the fact that ```x``` is not present in the second reaction proves that the first reaction and the second are not logically simultaneous. The second occurs one microstep later.

Note that it is possible to write code that will prevent logical time from advancing except by microsteps. For example, we could replace the reaction to ```repeat``` in ```Main``` with this one:
```
    reaction(repeat) -> d.y, repeat {=
        SET(d.y, 1);
        schedule(repeat, 0);
    =}
```
This would create what is known as a **stuttering Zeno** condition, where logical time cannot advance.  The output will be an unbounded sequence like this:
```
Time since start: 0.
  x is present.
Time since start: 0.
  y is present.
Time since start: 0.
  y is present.
Time since start: 0.
  y is present.
...
```

### Startup and Shutdown Reactions

Two special triggers are supported, **startup** and **shutdown**. A reaction that specifies the **startup** trigger will be invoked at the start of execution of the model.  The following two syntaxes have exactly the same effect:
```
    reaction(startup) {= ... =}
```
and
```
    timer t;
    reaction(t) {= ... =}
```
In other words, **startup** is a timer that triggers once at the first logical time of execution.  As with any other reaction, the reaction can also be triggered by inputs and can produce outputs or schedule actions.

The **shutdown** trigger is slightly different.  A shutdown reaction is specified as follows:
```
   reaction(shutdown) {= ... =}
```
This reaction will be invoked when the program terminates normally (there are no more events, some reaction has called a `request_stop()` utility provided in the target language, or the execution was specified to last a finite logical time). The reaction will be invoked at a logical time one microstep *later* than the last logical time of the execution. In other words, the presence of this reaction means that the program will execute one extra logical time cycle beyond what it would have otherwise, and that logical time is one microstep later than what would have otherwise been the last logical time.

If the reaction produces outputs, then downstream reactors will also be invoked at that later logical time. If the reaction schedules future reactions, those will be ignored. After the completion of this final logical time cycle, one microstep later than the normal termination, the program will exit.

## Contained Reactors

Reactors can contain instances of other reactors defined in the same file or in an imported file. Assuming the above [Count reactor](#state-declaration) is stored in a file [Count.lf](https://github.com/icyphy/lingua-franca/blob/master/test/C/src/lib/Count.lf), then [CountTest](https://github.com/icyphy/lingua-franca/blob/master/test/C/src/CountTest.lf) is an example that imports and instantiates it to test the reactor:
```
target C;
import Count.lf;
reactor Test {
    input c:int;
    state i:int(0);
    reaction(c) {=
        printf("Received %d.\n", c->value);
        (self->i)++;
        if (c->value != self->i) {
            printf("ERROR: Expected %d but got %d\n.", self->i, c->value);
            exit(1);
        }
    =}
    reaction(shutdown) {=
        if (self->i != 4) {
            printf("ERROR: Test should have reacted 4 times, but reacted %d times.\n", self->i);
            exit(2);
        }
    =}
}

main reactor CountTest {
    count = new Count();
    test = new Test();
    count.out -> test.c;
}
```
An instance is created with the syntax:

> *instance_name* = **new** *class_name*(*parameters*);

A bank with several instances can be created in one such statement, as explained in the [banks of reactors documentation](Multiports-and-Banks-of-Reactors#banks-of-reactors).

The *parameters* argument has the form:

> *parameter1_name* = *parameter1_value*,  *parameter2_name* = *parameter2_value*,  ...

Connections between ports are specified with the syntax:

> *output_port* -> *input_port*

where the ports are either *instance_name.port_name* or just *port_name*, where the latter form denotes a port belonging to the reactor that contains the instances. 

### Physical Connections

A subtle and rarely used variant is a **physical connection**, denoted `~>`. In such a connection, the logical time at the recipient is derived from the local physical clock rather than being equal to the logical time at the sender. The physical time will always exceed the logical time of the sender, so this type of connection incurs a nondeterministic positive logical time delay. Physical connections are useful sometimes in [[Distributed-Execution]] in situations where the nondeterministic logical delay is tolerable. Such connections are more efficient because timestamps need not be transmitted and messages do not need to flow through through a centralized coordinator (if a centralized coordinator is being used).

### Connections with Delays

Connections may include a **logical delay** using the **after** keyword, as follows:

> *output_port* -> *input_port* **after** 10 **msec**

This means that the logical time of the message delivered to the *input_port* will be 10 milliseconds larger than the logical time of the reaction that wrote to *output_port*. If the time value is greater than zero, then the event will appear at microstep 0. If it is equal to zero, then it will appear at the current microstep plus one.

When there are multiports or banks of reactors, several channels can be connected with a single connection statement. See [Multiports and Banks of Reactors](Multiports-and-Banks-of-Reactors#banks-of-reactors).

The following example defines a reactor that adds a counting sequence to its input. It uses the above Count and Add reactors (see [Hierarchy2](https://github.com/icyphy/lingua-franca/blob/master/test/C/src/Hierarchy2.lf)):

```
import Count.lf;
import Add.lf;
reactor AddCount {
    input in:int;
    output out:int;
    count = new Count();
    add = new Add();
    in -> add.in1;
    count.out -> add.in2;
    add.out -> out;
}
```

A reactor that contains other reactors may, within a reaction, send data to the contained reactor. The following example illustrates this (see [SendingInside](https://github.com/icyphy/lingua-franca/blob/master/test/C/src/SendingInside.lf)):
```
target C;
reactor Printer {
	input x:int;
	reaction(x) {=
		printf("Inside reactor received: %d\n", x->value);
	=}
}
main reactor SendingInside {
	p = new Printer();
	reaction(startup) -> p.x {=
		SET(p.x, 1);
	=}
}
```
Running this will print:
```
Inside reactor received: 1
```

## Deadlines

Lingua Franca includes a notion of a **deadline**, which is a relation between logical time and physical time. Specifically, a program may specify that the invocation of a reaction must occur within some physical-time interval of the logical timestamp of the message. If a reaction is invoked at logical time 12 noon, for example, and the reaction has a deadline of one hour, then the reaction is required to be invoked before the physical-time clock of the execution platform reaches 1 PM. If the deadline is violated, then the specified deadline handler is invoked instead of the reaction.  For example (see [Deadline](https://github.com/icyphy/lingua-franca/blob/master/test/C/src/Deadline.lf)):
```
reactor Deadline() {
    input x:int;
    output d:int; // Produced if the deadline is violated.
    reaction(x) -> d {=
        printf("Normal reaction.\n");
    =} deadline(10 msec) {=
        printf("Deadline violation detected.\n");
        SET(d, x->value);
    =}
```
This reactor specifies a deadline of 10 milliseconds (this can be a parameter of the reactor). If the reaction to `x` is triggered later in physical time than 10 msec past the timestamp of `x`, then the second body of code is executed instead of the first. That second body of code has access to anything the first body of code has access to, including the input `x` and the output `d`.  The output can be used to notify the rest of the system that a deadline violation occurred.

The amount of the deadline, of course, can be given by a parameter.

A sometimes useful pattern is when a container reactor reacts to deadline violations in a contained reactor. The [DeadlineHandledAbove](https://github.com/icyphy/lingua-franca/blob/master/test/C/src/DeadlineHandledAbove.lf) example illustrates this:
```
target C;
reactor Deadline() {
    input x:int;
    output deadline_violation:bool;
    reaction(x) -> deadline_violation {=
        ... normal code to execute ...
    =} deadline(100 msec) {=
        printf("Deadline violation detected.\n");
        SET(deadline_violation, true);
    =}
}
main reactor DeadlineHandledAbove {
    d = new Deadline();
    ...
    reaction(d.deadline_violation) {=
        ... handle the deadline violation ...
    =}
}
```

## Comments

Lingua Franca files can have C/C++/Java-style comments and/or Python-style comments. All of the following are valid comments:
```
    // Single-line C-style comment.
    /*
       Multi-line C-style comment.
     */
    # Single-line Python-style comment.
    '''
       Multi-line Python-style comment.
    '''
```