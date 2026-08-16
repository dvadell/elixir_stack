3 SW Engineering Principles
3.1 Export as few functions as possible from a module
Modules are the basic code structuring entity in Erlang. A module can
contain a large number of functions but only functions which are included
in the export list of the module can be called from outside the module.
Seen from the outside, the complexity of a module depends upon the
number of functions which are exported from the module. A module
which exports one or two functions is usually easier to understand than a
module which exports dozens of functions.
Modules where the ratio of exported/non-exported functions is low
are desirable in that a user of the module only needs to understand the
functionality of the functions which are exported from the module.
In addition, the writer or maintainer of the code in the module can
change the internal structure of the module in any appropriate manner
provided the external interface remains unchanged.
3.2 Try to reduce intermodule dependencies
A module which calls functions in many dicerent modules will be more
diecult to maintain than a module which only calls functions in a few
dicerent modules.
This is because each time we make a change to a module interface,
we have to check all places in the code where this module is called. Re-
ducing the interdependencies between modules simplifies the problem of
maintaining these modules.
We can simplify the system structure by reducing the number of dicer-
ent modules which are called from a given module.
Note also that it is desirable that the inter-module calling dependencies
form a tree and not a cyclic graph.

3.3 Put commonly used code into libraries
Commonly used code should be placed into libraries. The libraries should
be collections of related functions. Great ecort should be made in ensur-
ing that libraries contain functions of the same type. Thus a library such
as lists containing only functions for manipulating lists is a good choice,
whereas a library, lists_and_maths containing a combination of func-
tions for manipulating lists and for mathematics is a very bad choice.
The best library functions have no side ecects. Libraries with functions
with side ecects limit the re-usability.

3.4 Isolate “tricky” or “dirty” code into separate modules
Oden a problem can be solved by using a mixture of clean and dirty code.
Separate the clean and dirty code into separate modules.
Dirty code is code that does dirty things. Example:
• Uses the process dictionary.
• Uses erlang:process_info/1 for strange purposes.
• Does anything that you are not supposed to do (but have to do).
Concentrate on trying to maximize the amount of clean code and
minimize the amount of dirty code. Isolate the dirty code and clearly
comment or otherwise document all side ecects and problems associated
with this part of the code.

3.5 Don’t make assumptions about what the caller will do with the results of a function
Don’t make assumptions about why a function has been called or about
what the caller of a function wishes to do with the results.
For example, suppose we call a routine with certain arguments which
may be invalid. The implementer of the routine should not make any
assumptions about what the caller of the function wishes to happen when
the arguments are invalid.

Thus we should not write code like:
do_something(Args) ->
case check_args(Args) of
ok ->
{ok, do_it(Args)};
{error, What} ->
String = format_the_error(What),
%% Don’t do this
io:format("* error:~s\n", [String]),
error
end.
Instead write something like:
do_something(Args) ->
case check_args(Args) of
ok ->
{ok, do_it(Args)};
{error, What} ->
{error, What}
end.
error_report({error, What}) ->
format_the_error(What).
In the former case the error string is always printed on standard output,
in the latter case an error descriptor is returned to the application. The
application can now decide what to do with this error descriptor.
By calling error_report/1 the application can convert the error de-
scriptor to a printable string and print it if so required. But this may not
be the desired behaviour - in any case the decision as to what to do with
the result is led to the caller.

3.6 Abstract out common patterns of code or behaviour
Whenever you have the same pattern of code in two or more places in the
code try to isolate this in a common function and call this function instead
of having the code in two dicerent places. Copied code requires much
ecort to maintain.
If you see similar patterns of code (i.e. almost identical) in two or more
places in the code it is worth taking some time to see if one cannot change
the problem slightly to make the dicerent cases the same and then write
a small amount of additional code to describe the dicerences between the
two.
Avoid “copy” and “paste” programming, use functions!

3.7 Top-down
Write your program using the top-down fashion, not bottom-up (starting
with details). Top-down is a nice way of successively approaching details
of the implementation, ending up with defining primitive functions. The
code will be independent of representation since the representation is not
known when the higher levels of code are designed.

3.8 Don’t optimize code
Don’t optimize your code at the first stage. First make it right, then (if
necessary) make it fast (while keeping it right).

3.9 Use the principle of “least astonishment”
The system should always respond in a manner which causes the “least
astonishment” to the user - i.e. a user should be able to predict what will
happen when they do something and not be astonished by the result.
This has to do with consistency, a consistent system where dicerent
modules do things in a similar manner will be much easier to understand
than a system where each module does things in a dicerent manner.
If you get astonished by what a function does, either your function
solves the wrong problem or it has a wrong name.

3.10 Try to eliminate side efects
Erlang has several primitives which have side ecects. Functions which use
these cannot be easily re-used since they cause permanent changes to their
environment and you have to know the exact state of the process before
calling such routines.
Write as much as possible of the code with side-ecect free code.
Maximize the number of pure functions.
Collect together the functions which have side ecects and clearly doc-
ument all the side ecects.
With a little care most code can be written in a side-ecect free manner
- this will make the system a lot easier to maintain, test and understand.
3.11 Don’t allow private data structure to “leak” out of a
module
This is best illustrated by a simple example. We define a simple module
called queue - to implement queues:
-module(queue).
-export([add/2, fetch/1]).
add(Item, Q) ->
lists:append(Q, [Item]).
fetch([H|T]) ->
{ok, H, T};
fetch([]) ->
empty.
This implements a queue as a list, but unfortunately to use this the user
must know that the queue is represented as a list. A typical program to
use this might contain the following code fragment:
NewQ = [], % Don’t do this
Queue1 = queue:add(joe, NewQ),
Queue2 = queue:add(mike, Queue1), ....

This is bad - since the user a) needs to know that the queue is repre-
sented as a list and b) the implementer cannot change the internal repre-
sentation of the queue (they might want to do this later to provide a better
version of the module).
Better is:
-module(queue).
-export([new/0, add/2, fetch/1]).
new() ->
[].
add(Item, Q) ->
lists:append(Q, [Item]).
fetch([H|T]) ->
{ok, H, T};
fetch([]) ->
empty.
Now we can write:
NewQ = queue:new(),
Queue1 = queue:add(joe, NewQ),
Queue2 = queue:add(mike, Queue1), ...
Which is much better and corrects this problem. Now suppose the user
needs to know the length of the queue, they might be tempted to write:
Len = length(Queue) % Don’t do this
since they know that the queue is represented as a list. This is bad
programming practice which leads to code which is very diecult to main-
tain and understand. If they need to know the length of the queue then a
length function must be added to the module, thus:

-module(queue).
-export([new/0, add/2, fetch/1, len/1]).
new() -> [].
add(Item, Q) ->
lists:append(Q, [Item]).
fetch([H|T]) ->
{ok, H, T};
fetch([]) ->
empty.
len(Q) ->
length(Q).
Now the user can call queue:len(Queue) instead.
Here we say that we have “abstracted out” all the details of the queue
(the queue is in fact what is called an “abstract data type”).
Why do we go to all this trouble? The practice of abstracting out inter-
nal details of the implementation allows us to change the implementation
without changing the code of the modules which call the functions in the
module we have changed. So, for example, a better implementation of the
queue is as follows:
-module(queue).
-export([new/0, add/2, fetch/1, len/1]).
new() ->
{[],[]}.
add(Item, {X,Y}) -> % Faster addition of elements
{[Item|X], Y}.
225
fetch({X, [H|T]}) ->
{ok, H, {X,T}};
fetch({[], []) ->
empty;
fetch({X, []) ->
% Perform this heavy computation only sometimes.
fetch({[],lists:reverse(X)}).
len({X,Y}) ->
length(X) + length(Y).

3.12 Make code as deterministic as possible
A deterministic program is one which will always run in the same man-
ner no matter how many times the program is run. A non-deterministic
program may deliver dicerent results each time it is run. For debugging
purposes it is a good idea to make things as deterministic as possible. This
helps make errors reproducible.
For example, suppose one process has to start five parallel processes
and then check that they have started correctly, suppose further that the
order in which these five are started does not matter.
We could then choose to either start all five in parallel and then check
that they have all started correctly but it would be better to start them one
at a time and check that each one has started correctly before starting the
next one.

3.13 Do not program “defensively”
A defensive program is one where the programmer does not “trust” the
input data to the part of the system they are programming. In general one
should not test input data to functions for correctness. Most of the code
in the system should be written with the assumption that the input data to
the function in question is correct. Only a small part of the code should
actually perform any checking of the data. This is usually done when data
“enters” the system for the first time, so once data has been checked as it
enters the system it should thereader be assumed correct.
Example:
%% Args: Option is all | normal
get_server_usage_info(Option, AsciiPid) ->
Pid = list_to_pid(AsciiPid),
case Option of
all -> get_all_info(Pid);
normal -> get_normal_info(Pid)
end.
The function will crash if Option neither normal nor all, and it
should do that. The caller is responsible for supplying correct input.

3.14 Isolate hardware interfaces with a device driver
Hardware should be isolated from the system through the use of device
drivers. The device drivers should implement hardware interfaces which
make the hardware appear as if they were Erlang processes. Hardware
should be made to look and behave like normal Erlang processes. Hard-
ware should appear to receive and send normal Erlang messages and
should respond in the conventional manner when errors occur.
3.15 Do and undo things in the same function
Suppose we have a program which opens a file, does something with it
and closes it later. This should be coded as:
do_something_with(File) ->
case file:open(File, read) of,
{ok, Stream} ->
doit(Stream),
file:close(Stream) % The correct solution
Error -> Error
end.

Note how we open the ﬁle (file:open)and close it (file:close) in
the same routine. The solution below is much harder to follow and it is
not obvious which ﬁle is closed. Don’t program it like this:
do_something_with(File) ->
case file:open(File, read) of,
{ok, Stream} ->
doit(Stream)
Error -> Error
end.
doit(Stream) ->
....,
func234(...,Stream,...).
...
func234(..., Stream, ...) ->
...,
file:close(Stream) %% Don’t do this


4 Error Handling
4.1 Separate error handling and normal case code
Don’t clutter code for the “normal case” with code designed to handle
exceptions. As far as possible you should only program the normal case.
If the code for the normal case fails, your process should report the error
and crash as soon as possible. Don’t try to ﬁx up the error and continue.
The error should be handled in a dicerent process. (See “Each process
should only have one role” on page 229).
Clean separation of error recovery code and normal case code should
greatly simplify the overall system design.
The error logs which are generated when a sodware or hardware error
is detected will be used at a later stage to diagnose and correct the error.
A permanent record should be kept of any information that will be helpful
in this process.

4.2 Identify the error kernel
One of the basic elements of system design is identifying which part of the
system has to be correct and which part of the system does not have to be
correct.
In conventional operating system design the kernel of the system is
assumed to be, and must be, correct, whereas all user application programs
do not necessarily have to be correct. If a user application program fails
this will only concern the application where the failure occurred but should
not acect the integrity of the system as a whole.
The ﬁrst part of the system design must be to identify that part of the
system which must be correct; we call this the error kernel. Oden the
error kernel has some kind of real-time memory resident data base which
stores the state of the hardware.

5 Processes, Servers and Messages
5.1 Implement a process in one module
Code for implementing a single process should be contained in one mod-
ule. A process can call functions in any library routines but the code for
the “top loop” of the process should be contained in a single module. The
code for the top loop of a process should not be split into several modules -
this would make the ﬂow of control extremely diecult to understand. This
does not mean that one should not make use of generic server libraries,
these are for helping structuring the control ﬂow.
Conversely, code for no more than one kind of process should be
implemented in a single module. Modules containing code for several
dicerent processes can be extremely diecult to understand. The code for
each individual process should be broken out into a separate module.

5.2 Use processes for structuring the system
Processes are the basic system structuring elements. But don’t use pro-
cesses and message passing when a function call can be used instead.

5.3 Registered processes
Registered processes should be registered with the same name as the mod-
ule. This makes it easy to ﬁnd the code for a process.
Only register processes that should live a long time.

5.4 Assign exactly one parallel process to each true concur-
rent activity in the system
When deciding whether to implement things using sequential or parallel
processes then the structure implied by the intrinsic structure of the prob-
lem should be used. The main rule is:
“Use one parallel process to model each truly concurrent activity in the
real world.”
If there is a one-to-one mapping between the number of parallel pro-
cesses and the number of truly parallel activities in the real world, the
program will be easy to understand.

5.5 Each process should only have one “role”
Processes can have dicerent roles in the system, for example in the client-
server model.
As far as possible a process should only have one role, i.e. it can be a
client or a server but should not combine these roles.
Other roles which processes might have are:
Supervisor watches other processes and restarts them if they fail.
Worker a normal work process (can have errors).
Trusted Worker not allowed to have errors.

5.6 Use generic functions for servers and protocol handlers
wherever possible
In many circumstances it is a good idea to use generic server programs
such as the generic server implemented in the standard libraries. Con-
230 APPENDIX B. PROGRAMMING RULES AND CONVENTIONS
sistent use of a small set of generic servers will greatly simplify the total
system structure.
The same is possible for most of the protocol handling sodware in the
system.

5.7 Tag messages
All messages should be tagged. This makes the order in the receive state-
ment less important and the implementation of new messages easier.
Don’t program like this:
loop(State) ->
receive
...
{Mod, Funcs, Args} -> % Don’t do this
apply(Mod, Funcs, Args},
loop(State);
...
end.
The new message {get_status_info, From, Option} will intro-
duce a conﬂict if it is placed below the {Mod, Func, Args} message.
If messages are synchronous, the return message should be tagged
with a new atom, describing the returned message. Example: if the incom-
ing message is tagged get_status_info, the returned message could be
tagged status_info. One reason for choosing dicerent tags is to make
debugging easier.
This is a good solution:
loop(State) ->
receive
...
% Use a tagged message.
{execute, Mod, Funcs, Args} ->
apply(Mod, Funcs, Args},
231
loop(State);
{get_status_info, From, Option} ->
From ! {status_info,
get_status_info(Option, State)},
loop(State);
...
end.

5.8 Flush unknown messages
Every server should have an Other alternative in at least one receive
statement. This is to avoid ﬁlling up message queues. Example:
main_loop() ->
receive
{msg1, Msg1} ->
...,
main_loop();
{msg2, Msg2} ->
...,
main_loop();
Other -> % Flushes the message queue.
error_logger:error_msg(
"Error: Process ~w got unknown msg ~w~n.",
[self(), Other]),
main_loop()
end.
5.9 Write tail-recursive servers
All servers must be tail-recursive, otherwise the server will consume mem-
ory until the system runs out of it.
Don’t program like this:
loop() ->
receive
232 APPENDIX B. PROGRAMMING RULES AND CONVENTIONS
{msg1, Msg1} ->
...,
loop();
stop ->
true;
Other ->
error_logger:log({error, {process_got_other,
self(), Other}}),
loop()
end,
% Don’t do this!
% This is NOT tail-recursive
io:format("Server going down").
This is a correct solution:
loop() ->
receive
{msg1, Msg1} ->
...,
loop();
stop ->
io:format("Server going down");
Other ->
error_logger:log({error, {process_got_other,
self(), Other}}),
loop()
end. % This is tail-recursive
If you use some kind of server library, for example generic, you
automatically avoid doing this mistake.

5.10 Interface functions
Use functions for interfaces whenever possible, avoid sending messages
directly. Encapsulate message passing into interface functions. There are
cases where you can’t do this.
The message protocol is internal information and should be hidden to
other modules.
Example of interface function:
-module(fileserver).
-export([start/0, stop/0, open_file/1, ...]).
open_file(FileName) ->
fileserver ! {open_file_request, FileName},
receive
{open_file_response, Result} -> Result
end.
...<code>...

5.11 Time-outs
Be careful when using after in receive statements. Make sure that
you handle the case when the message arrives later (See “Flush unknown
messages” on page 231).

5.12 Trapping exits
As few processes as possible should trap exit signals. Processes should
either trap exits or they should not. It is usually very bad practice for a
process to “toggle” trapping exits.

6 Various Erlang Speciﬁc Conventions
6.1 Use records as the principle data structure
Use records as the principle data structure. A record is a tagged tuple and
was introduced in Erlang version 4.3 and thereader (see EPK/NP 95:034).
It is similar to struct in C or record in Pascal.
If the record is to be used in several modules, its deﬁnition should be
placed in a header ﬁle (with suex .hrl) that is included from the modules.
If the record is only used from within one module, the deﬁnition of the
record should be in the beginning of the ﬁle where the module is deﬁned.
The record features of Erlang can be used to ensure cross module
consistency of data structures and should therefore be used by interface
functions when passing data structures between modules.
6.2 Use selectors and constructors
Use selectors and constructors provided by the record feature for managing
instances of records. Don’t use matching that explicitly assumes that the
record is a tuple. Example:
demo() ->
P = #person{name = "Joe", age = 29},
#person{name = Name1} = P,% Use matching, or...
Name2 = P#person.name. % like this.
Don’t program like this:
demo() ->
P = #person{name = "Joe", age = 29},
% Don’t do this
{person, Name, _Age, _Phone, _Misc} = P.

6.3 Use tagged return values
Use tagged return values.
Don’t program like this:
keysearch(Key, [{Key, Value}|_Tail]) ->
Value; %% Don’t return untagged values!
keysearch(Key, [{_WrongKey,_WrongValue}|Tail]) ->
keysearch(Key, Tail);
keysearch(Key, []) ->
false.

Then the Key, Value cannot contain the false value. This is the correct
solution:
keysearch(Key, [{Key, Value}|_Tail]) ->
{value, Value}; %% Correct. Returns tagged value.
keysearch(Key, [{_WrongKey, _WrongValue}|Tail]) ->
keysearch(Key, Tail);
keysearch(Key, []) ->
false.

6.4 Use catch and throw with extreme care
Do not use catch and throw unless you know exactly what you are doing!
Use catch and throw as little as possible.
Catch and throw can be useful when the program handles compli-
cated and unreliable input (from the outside world, not from your own
reliable program) that may cause errors in many places deeply within the
code. One example is a compiler.

6.5 Use the process dictionary with extreme care
Do not use get and put etc. unless you know exactly what you are doing!
Use get and put etc. as little as possible.
A function that uses the process dictionary can be rewritten by intro-
ducing a new argument.
Example:
Don’t program like this:
tokenize([H|T]) ->
...;
tokenize([]) ->
% Don’t use get/1 (like this)
case get_characters_from_device(get(device)) of
eof -> [];
{value, Chars} ->
236 APPENDIX B. PROGRAMMING RULES AND CONVENTIONS
tokenize(Chars)
end.
The correct solution:
tokenize(_Device, [H|T]) ->
...;
tokenize(Device, []) ->
% This is better
case get_characters_from_device(Device) of
eof -> [];
{value, Chars} ->
tokenize(Device, Chars)
end.
The use of get and put might cause a function to behave dicerently
when called with the same input arguments. This makes the code hard
to read since it is non-deterministic. Debugging will be more complicated
since a function using get and put is a function of not only of its input
arguments, but also of the process dictionary. Many of the run time errors
in Erlang (for example bad_match) include the arguments to a function,
but never the process dictionary.
6.6 Don’t use import
Don’t use -import, using it makes the code harder to read since you
cannot directly see in what module a function is defined. Use exref
(Cross Reference Tool) to find module dependencies.

6.7 Exporting functions
Make a distinction of why a function is exported. A function can be
exported for the following reasons (for example):
• It is a user interface to the module.
• It is an interface function for other modules.
• It is called from apply, spawn etc. but only from within its module.
Use dicerent -export groupings and comment them accordingly. Ex-
ample:
%% user interface
-export([help/0, start/0, stop/0, info/1]).
%% intermodule exports
-export([make_pid/1, make_pid/3]).
-export([process_abbrevs/0, print_info/5]).
%% exports for use within module only
-export([init/1, info_log_impl/1]).

7 Specific Lexical and Stylistic Conventions
7.1 Don’t write deeply nested code
Nested code is code containing case/if/receive statements within other
case/if/receive statements. It is bad programming style to write deeply
nested code - the code has a tendency to drid across the page to the
right and soon becomes unreadable. Try to limit most of your code to a
maximum of two levels of indentation. This can be achieved by dividing
the code into shorter functions.

7.2 Don’t write very large modules
A module should not contain more than 400 lines of source code. It is
better to have several small modules than one large one.

7.3 Don’t write very long functions
Don’t write functions with more than 15 to 20 lines of code. Split large
functions into several smaller ones. Don’t solve the problem by writing
long lines.

7.4 Don’t write very long lines
Don’t write very long lines. A line should not have more than 80 charac-
ters. (It will for example fit into an A4 page.)
In Erlang 4.3 and thereader string constants will be automatically con-
catenated. Example:
io:format("Name: ~s, Age: ~w, Phone: ~w ~n"
"Dictionary: ~w.~n", [Name, Age, Phone, Dict])

7.5 Variable names
Choose meaningful variable names - this is very diecult.
If a variable name consists of several words, use “ ” or a capitalized
letter to separate them. Example: My_variable or MyVariable.
Avoid using “ ” as don’t care variable, use variables beginning with
“ ” instead. Example: _Name. If at a later stage you need the value of
the variable, you just remove the leading underscore. You will have no
problem finding what underscore to replace and the code will be easier to
read.

7.6 Function names
The function name must agree exactly with what the function does. It
should return the kind of arguments implied by the function name. It
should not surprise the reader. Use conventional names for conventional
functions ( start, stop, init, main_loop).
Functions in dicerent modules that solve the same problem should
have the same name. Example: Module:module_info().

Bad function names are one of the most common programming errors
- good choice of names is very diecult!
Some kind of naming convention is very useful when writing lots of
dicerent functions. For example, the name preﬁx “is_” could be used to
signify that the function in question returns the atom true or false.
is_...() -> true | false
check_...() -> {ok, ...} | {error, ...}

7.7 Module names
Erlang has a ﬂat module structure (i.e. there are no modules within mod-
ules). Oden, however, we might like to simulate the ecect of a hierarchical
module structure. This can be done with sets of related modules having
the same module preﬁx.
If, for example, an ISDN handler is implemented using ﬁve dicerent
and related modules. These module should be given names such as:
isdn_init
isdn_partb
isdn_...

7.8 Format programs in a consistent manner
A consistent programming style will help you, and other people, to under-
stand your code. Dicerent people have dicerent styles concerning inden-
tation, usage of spaces etc.
For example you might like to write tuples with a single comma be-
tween the elements:
{12,23,45}
Other people might use a comma followed by a blank:
{12, 23, 45}
Once you have adopted style - stick to it.
Within a larger project, the same style should be used in all parts.

8 Documenting Code
8.1 Attribute code
You must always correctly attribute all code in the module header. Say
where all ideas contributing to the module came from - if your code was
derived from some other code say where you got this code from and who
wrote it.
Never steal code - stealing code is taking code from some other module
editing it and forgetting to say who wrote the original.
Examples of useful attributes are:
-revision(’Revision: 1.14 ’).
-created(’Date: 1995/01/01 11:21:11 ’).
-created_by(’eklas@erlang’).
-modified(’Date: 1995/01/05 13:04:07 ’).
-modified_by(’mbj@erlang’).

8.2 Provide references in the code to the speciﬁcations
Provide cross references in the code to any documents relevant to the
understanding of the code. For example, if the code implements some
communication protocol or hardware interface give an exact reference
with document and page number to the documents that were used to
write the code.

8.3 Document all the errors
All errors should be listed together with an English description of what
they mean in a separate document (See “Error Messages” on page 246.)
By errors we mean errors which have been detected by the system.
At a point in your program where you detect a logical error call the
error logger thus:
error_logger:error_msg(Format,
{Descriptor, Arg1, ....})
And make sure that the line {Descriptor, Arg1,...} is added to
the error message documents.

8.4 Document all the principle data structures in messages
Use tagged tuples as the principle data structure when sending messages
between dicerent parts of the system.
The record features of Erlang (introduced in Erlang versions 4.3 and
thereader) can be used to ensure cross module consistency of data struc-
tures.
An English description of all these data structure should be docu-
mented (See “Message Descriptions” on page 246.)

8.5 Comments
Comments should be clear and concise and avoid unnecessary wordiness.
Make sure that comments are kept up to date with the code. Check that
comments add to the understanding of the code. Comments should be
written in English.
Comments about the module should not be indented and should start
with three percent characters (%%%), (See “File Header, description” on
page 243).
Comments about a function should not be indented and start with two
percent characters (%%), (See “Comment each function” on page 242).
Comments within Erlang code should start with one percent character
(%). If a line only contains a comment, it should be indented as Erlang
code. This kind of comment should be placed above the statement it
refers to. If the comment can be placed on the same line as the statement,
this is preferred.
%% Comment about function
some_useful_functions(UsefulArgugument) ->
another_functions(UsefulArgugument), % Comment at end of line
% Comment about complicated_stmnt at the same level of indentation
complicated_stmnt,
......

8.6 Comment each function
The important things to document are:
• The purpose of the function.
• The domain of valid inputs to the function. That is, data structures
of the arguments to the functions together with their meaning.
• The domain of the output of the function. That is, all possible data
structures of the return value together with their meaning.
• If the function implements a complicated algorithm, describe it.
• The possible causes of failure and exit signals which may be gener-
ated by exit/1, throw/1 or any non-obvious run time errors. Note
the dicerence between failure and returning an error.
• Any side ecect of the function.
Example:
%%----------------------------------------------------------------------
%% Function: get_server_statistics/2
%% Purpose: Get various information from a process.
%% Args: Option is normal|all.
%% Returns: A list of {Key, Value}
%% or {error, Reason} (if the process is dead)
%%----------------------------------------------------------------------
get_server_statistics(Option, Pid) when pid(Pid) ->
......

8.7 Data structures
The record should be deﬁned together with a plain text description. Ex-
ample:
%% File: my_data_structures.h
%%---------------------------------------------------------------------
%% Data Type: person
%% where:
%% name: A string (default is undefined).
%% age: An integer (default is undefined).
%% phone: A list of integers (default is []).
%% dict: A dictionary containing various information about the person.
%% A {Key, Value} list (default is the empty list).
%%----------------------------------------------------------------------
-record(person, {name, age, phone = [], dict = []}).

8.8 File headers, copyright
Each ﬁle of source code must start with copyright information, for example:
%%%---------------------------------------------------------------------
%%% Copyright Ericsson Telecom AB 1996
%%%
%%% All rights reserved. No part of this computer programs(s) may be
%%% used, reproduced,stored in any retrieval system, or transmitted,
%%% in any form or by any means, electronic, mechanical, photocopying,
%%% recording, or otherwise without prior written permission of
%%% Ericsson Telecom AB.
%%%---------------------------------------------------------------------

8.9 File headers, revision history
Each ﬁle of source code must be documented with its revision history
which shows who has been working with the ﬁles and what they have
done to it.
%%%---------------------------------------------------------------------
%%% Revision History
%%%---------------------------------------------------------------------
%%% Rev PA1 Date 960230 Author Fred Bloggs (ETXXXXX)
%%% Initial pre release. Functions for adding and deleting foobars
%%% are incomplete
%%%---------------------------------------------------------------------
%%% Rev A Date 960230 Author Johanna Johansson (ETXYYY)
%%% Added functions for adding and deleting foobars and changed
%%% data structures of foobars to allow for the needs of the Baz
%%% signalling system
%%%---------------------------------------------------------------------
8.10 File Header, description
Each ﬁle must start with a short description of the module contained in
the ﬁle and a brief description of all exported functions.
%%%---------------------------------------------------------------------
%%% Description module foobar_data_manipulation
%%%---------------------------------------------------------------------
%%% Foobars are the basic elements in the Baz signalling system. The
%%% functions below are for manipulating that data of foobars and for
%%% etc etc etc
%%%---------------------------------------------------------------------
%%% Exports
%%%---------------------------------------------------------------------
%%% create_foobar(Parent, Type)
%%% returns a new foobar object
%%% etc etc etc
%%%---------------------------------------------------------------------

If you know of any weakness, bugs, badly tested fea-
tures, make a note of them in a special comment, don’t
try to hide them. If any part of the module is incomplete,
add a special comment. Add comments about anything which will
be of help to future maintainers of the module. If the product of which the
module you are writing is a success, it may still be changed and improved
in ten years time by someone you may never meet.
8.11 Do not comment out old code - remove it
Add a comment in the revision history to that ecect. Remember the source
code control system will help you!
8.12 Use a source code control system
All non trivial projects must use a source code control system such as RCS,
CVS or Clearcase to keep track of all modules.
9 The Most Common Mistakes
• Writing functions which span many pages (See “Don’t write very
long functions” on page 238).
• Writing functions with deeply nested ifs receives, cases etc (See
“Don’t write deeply nested code” on page 237).
• Writing badly typed functions (See “Use tagged return values” on
page 234).
• Function names which do not reﬂect what the functions do (See
“Function names” on page 238).
• Variable names which are meaningless (See “Variable names” on
page 238).

• Using processes when they are not needed (See “Assign exactly one
parallel process to each true concurrent activity in the system” on
page 229).
• Badly chosen data structures (Bad representations).
• Bad comments or no comments at all (always document arguments
and return value).
• Unindented code.
• Using put/get (See “Use the process dictionary with extreme care”
on page 235).
• No control of the message queues (See “Flush unknown messages”
on page 231 and “Time-outs” on page 233).
10 Required Documents
This section describes some of the system level documents which are nec-
essary for designing and maintaining system programmed using Erlang.
10.1 Module Descriptions
One chapter per module. Contains description of each module, and all
exported functions as follows:
• the meaning and data structures of the arguments to the functions
• the meaning and data structure of the return value
• the purpose of the function
• the possible causes of failure and exit signals which may be gener-
ated by explicit calls to exit/1.
Format of document to be deﬁned later:

10.2 Message Descriptions
The format of all inter-process messages except those deﬁned inside one
module.
Format of document to be deﬁned later:

10.3 Process
Description of all registered servers in the system and their interface and
purpose.
Description of the dynamic processes and their interfaces.
Format of document to be deﬁned later:
10.4 Error Messages
Description of error messages
Format of document 
to be deﬁned later:
