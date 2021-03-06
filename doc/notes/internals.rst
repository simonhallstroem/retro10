====================
Implementation Notes
====================

Overview
--------
Retro is not a standard-compliant Forth. It's significantly
different in many areas. This section will help explain these
differences and show how Retro works internally.

Threading Model
---------------
Retro uses subroutine threading with inline machine code for
select words. This model has been used by Retro since 2001
as it is simple, clean, and allows for optimization to be
done by the compiler if desired.

Taking a look at the subroutine threaded code generated by
Retro:

::

  : foo 1 2 + . ;

Will compile to:

::

  lit 1
  lit 2
  +
  call .
  ;

Simple operations that map to single instructions can
(optionally) be inlined by the Retro compiler, saving
some call/return overhead. Other optimizations are also
possible.

Interpreting and Compiling
--------------------------
Retro has a very simple intepreter loop.

::

  : listen  ( - )
    repeat ok 32 # accept search word? number? again ;

This displays a prompt (**ok**), accepts input until a space
is encountered (ASCII 32). The dictionary is searched, and if
the word is found, **word?** calls the class handler for the
word. If not found, **number?** tries to convert it to a
number. If this fails as well, an error is displayed. In any
case, exection repeats until a fatal error arises, or until
the user executes **bye**.

There is no separate compilation process. In Retro, the
**compiler** is nothing more than a state variable that the
*word classes* use to decide what to do with a word.

Word Classes
------------
As mentioned above, the interpreter loop (**listen**) passes
the words (and also data elements like numbers) to something
called a *word class*.

This is another area in which Retro's implementation differs
from standard Forths. The word class approach was created by
Helmar Wodtke and allows for the interpreter and compiler to
be extremely clean by allowing special words (*class handlers*)
to handle different types of words.

This means that the interpreter loop does not need to be
aware of the type a word has, or of any aspect of the system
state.

The standard Retro language has several classes defined.

+-----------+------------+-----------------------------------------+
| Name      | Data Stack | Address Stack                           |
+===========+============+=========================================+
| .forth    | a -        | ``-``                                   |
+-----------+------------+-----------------------------------------+
| If interpreting, call the word. If compiling, compile a call     |
| to the word.                                                     |
+-----------+------------+-----------------------------------------+
| .primitive| a -        | ``-``                                   |
+-----------+------------+-----------------------------------------+
| If interpreting, call the word. If compiling, copy the first     |
| instruction into the target definition. If word is revectored,   |
| fall back to .forth class behavior.                              |
+-----------+------------+-----------------------------------------+
| .macro    | a -        | ``-``                                   |
+-----------+------------+-----------------------------------------+
| Always call the word. This is normally used for words that lay   |
| down custom code at compile time, or which need to have          |
| different behaviors during compilation.                          |
+-----------+------------+-----------------------------------------+
| .compiler | a -        | ``-``                                   |
+-----------+------------+-----------------------------------------+
| Call the word when the compiler is on. If compiler is off, do    |
| nothing.                                                         |
+-----------+------------+-----------------------------------------+
| .data     | a -        | ``-``                                   |
+-----------+------------+-----------------------------------------+
| If interpreting, leave the address on the stack. If compiling,   |
| compile the address into the target definition as a literal.     |
+-----------+------------+-----------------------------------------+

In addition to the three core classes, it is possible to create your
own classes. As an example, we'll create a class for naming and
displaying strings. Our class has the following behavior:

- If interpreting, display the string
- If compiling, lay down the code needed to display the
  string

Retro has a convention of using a . as the first character of a
class name. In continuing this tradition, we'll call our new
class **.string**

Tip:
  On entry to a class, the address of the word or data
  structure is on the stack. The compiler state (which most
  classes will need to check) is in a variable named compiler.

A first step is to lay down a simple skeleton. Since we need to
lay down custom code at compile time, the class handler will
have two parts.

::

  : .string  ( a—)
    compiler @ 0 =if ( interpret time ) ;; then ( compile time )
  ;

We'll start with the interpret time action. We can replace this
with type, since the whole point of this class is to display a
string object.

::

  : .string ( a — )
    compiler @ 0 =if type ;; then ( compile time ) ;

The compile time action is more complex. We need to lay down
the machine code to leave the address of the string on the
stack when the word is run, and then compile a call to type. If
you look at the instruction set listing, you'll see that opcode
1 is the instruction for putting values on the stack. This
opcode takes a value from the following memory location and
puts it on the stack. So the first part of the compile time
action is:

::

  : .string ( a — )
    compiler @ 0 =if type ;; then 1 , , ;

Tip:
  Use **,** to place values directly into memory. This is the
  cornerstone of the entire compiler.

One more thing remains. We still have to compile a call to
type. We can do this by passing the address of type to
compile.

::

  : .string ( a — )
    compiler @ 0 =if type ;; then 1 , , ['] type compile ;

And now we have a new class handler. The second part is to use
the new class.

Tip:
  New dictionary entries are made using create. The class can
  be set after creation by accessing the proper fields in the
  dictionary header. Words starting with **d->** are used to access
  fields in the dictionary headers.

::

  : displayString: ( "name" — )
    create ['] .string reclass keepString last @ d->xt ! ;

This uses **create** to make a new word, then sets the class to
**.string** and the xt of the word to the string. It also makes the
string permanent using keepString. last is a variable pointing
to the most recently created dictionary entry. The words **d->class**,
**d->xt**, and **d->name** are dictionary field accessors and are used
to provide portable access to fields in the dictionary.

We can now test the new class:

::

  " hello, world!" displayString: hello
  hello
  : foo hello cr ;
  foo


Vectors
-------
Vectors are another important concept in Retro.

Most Forth systems provide a way to define a word which can
have its meaning altered later. Retro goes a step further by
allowing all words defined using **:** to be redefined. Words
which can be redefined are called *vectors*.

Vectors can be replaced by using **is**, or returned to their
original definition with **devector**. For instance:

::

  : foo 23 . ;
  foo
  : bar 99 . ;
  ' bar is foo
  foo
  devector foo
  foo

There are also variations of **is** and **devector** which take the
addresses of the words rather than parsing for the word name.
These are **:is** and **:devector**.


I/O Devices
-----------
Retro runs on a portable virtual machine. This machine provides a
few I/O devices that can be accessed.

- keyboard
- text display
- graphical canvas
- mouse

Please note that the only target currently supporting the canvas and
mouse is *javascript*.

When talking to an I/O device, set the stack as instructed, then
write a value to the port using **out**. You then **wait**, and,
depending on the device, may *read* a value back.

+-----------+------------+-----------------------------------------+
| Port      | Send       | Stack Effect                            |
+===========+============+=========================================+
| 0         | 0          | ``-``                                   |
+-----------+------------+-----------------------------------------+
| Tell the computer that an I/O request has been made. This is done|
| by **wait**.                                                     |
+-----------+------------+-----------------------------------------+
| 1         | 1          | ``-``                                   |
+-----------+------------+-----------------------------------------+
| Wait for a keypress. Read this port after a **wait** to get the  |
| key.                                                             |
+-----------+------------+-----------------------------------------+
| 2         | 1          | ``c-``                                  |
+-----------+------------+-----------------------------------------+
| Display a character. Put the character on the stack, then        |
| **wait**.                                                        |
+-----------+------------+-----------------------------------------+
| 3         | 0          | ``-``                                   |
+-----------+------------+-----------------------------------------+
| Send 0 to force a video update. This does not require **wait**   |
+-----------+------------+-----------------------------------------+
| 4         | 1          | ``-``                                   |
+-----------+------------+-----------------------------------------+
| Save the image.                                                  |
+-----------+------------+-----------------------------------------+
| 5         | See Notes  | ``-``                                   |
+-----------+------------+-----------------------------------------+
| This is the capabilities query port. An image can use this to    |
| check the hardware supported by the VM.                          |
|                                                                  |
| Send one of the following, **wait**, then read back to get the   |
| result.                                                          |
|                                                                  |
| - -1 : Amount of memory provided                                 |
| - -2 : Is Canvas present?  0 if not, -1 if yes                   |
| - -3 : Canvas Width                                              |
| - -4 : Canvas Height                                             |
| - -5 : Stack Depth                                               |
| - -6 : Address Stack Depth                                       |
| - -7 : Is Mouse supported? 0 if not, -1 if yes                   |
+-----------+------------+-----------------------------------------+
| 6         | See Notes  | See Notes                               |
+-----------+------------+-----------------------------------------+
| This is the canvas display driver. It has multiple operations.   |
|                                                                  |
| - 1 : n- : Change color                                          |
| - 2 : xy- : Set pixel                                            |
| - 3 : xyhw- : Draw a rectangle                                   |
| - 4 : xyhw- : Draw a solid rectangle                             |
| - 5 : xyh- : Draw a vertical line                                |
| - 6 : xyw- : Draw a horizontal line                              |
| - 7 : xyw- : Draw a circle                                       |
| - 8 : xyw- : Draw a solid circle                                 |
+-----------+------------+-----------------------------------------+
| 7         | See Notes  | See Notes                               |
+-----------+------------+-----------------------------------------+
| This is the mouse device.                                        |
|                                                                  |
| - 1 : -xy : Get mouse x, y coords                                |
| - 2 : -f : Get a flag indicating the up/down state of the button |
+-----------+------------+-----------------------------------------+
