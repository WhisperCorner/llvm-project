===========================
Defining Undefined Behavior
===========================

.. contents:: Table of Contents
   :depth: 2
   :local:

The C standard leaves behavior undefined or implementation defined in many
places. Undefined behavior is behavior that the standards leave up to the
implementation. As an implementation, LLVM's libc must provide a result for any
input, including inputs for which the result is undefined. This page provides
examples of how these situations are handled in existing code, followed by
guidelines to help determine the right approach for new situations.

Guidelines
==========
Most undefined behavior is outside the scope of normal use. Follow these
guidelines and the resulting code should behave predictably even in unexpected
situations.

#. Follow the standards.
    #. If there is no standard, first ask yourself if this implementation is necessary (are there users who need this functionality?). If it truly is, then match existing implementations. Creating competing designs just causes confusion (see the history of qsort_r).
#. Avoid giving an incorrect answer.
    #. In general, correct answer > correct answer (wrong format) > no answer > crash the program >>>>>>> incorrect answer.
    #. The C library is called frequently in performance critical situations, and so can't afford to do thorough error checking and correction.
    #. It also cannot give the incorrect answer for any reasonable input, since it is so foundational.
    #. This leaves crashing or address space corruption as a probable option for a libc function in an ambiguous state.
#. Don't overcomplicate undefined situations.
    #. It's better to have a slightly confusing result for an undefined input than 100 extra lines of code that are never used for a well defined input.
    #. LLVM's libc is also used for embedded systems that care a lot about code size.
    #. Unreasonable inputs can have unreasonable outputs.
#. Match other implementations when it makes sense.
    #. Every libc has to make these choices, and sometimes others have already found the right choice.
    #. Be careful, just because there is a consensus doesn't make that consensus right.
#. LLVM's libc should be consistent with itself.
    #. Similar inputs to the same function should yield similar results, even when the inputs are undefined.
    #. The same input to similar functions should also yield similar results.
    #. The same input to the same function on different platforms should yield the same result, unless there's a specific reason not to (e.g. 64 bit long vs 32 bit long).
#. Write down the decision.
    #. Every libc has to make a decision on how to handle undefined inputs. Users should be able to find what LLVM's libc does.
    #. While users shouldn't rely on undefined behavior, it shouldn't surprise them.

Approaches
==========

Matching Behavior Against Existing Implementations
--------------------------------------------------
Existing implementations have already chosen how to handle undefined situations, and sometimes there are benefits to matching those decisions, such as in the case of atoi. The C Standard defines atoi as being equivalent to a call to strtol, with the result cast from long to int. The standard also clarifies that any input that cannot be represented as an int causes undefined behavior. For the strtol function, the standard instead defines inputs that cannot be represented in a long int as returning LONG_MAX or LONG_MIN, according to their sign. The decision of whether to cast the result from strtol or to handle integer overflow like strtol does is left to the implementation. LLVM's libc performs the raw cast, since the atoi function is fuzz tested against the implementation from glibc. By matching a known good implementation, LLVM's libc can more effectively be checked for correctness in this case.

Simplifying Handling Invalid Inputs
-----------------------------------
When handling invalid inputs, the output should be simple to code, and simple for the user to understand. An example of this is how the printf function handles invalid conversion specifiers. A conversion specifier is a segment of the format string that starts with a %. At the end of a conversion specifier is the character that determines the behavior for the conversion, called the conversion name. As an example, the conversion specifier %d has the conversion name of d which represents an integer conversion. If the conversion name is instead an invalid character such as ? then the behavior is undefined. When passed an invalid conversion specifier like %? LLVM's libc defines the output as the raw text of the conversion specifier. This simplifies the algorithm and makes the result obvious and predictable for the user.

Conforming to Existing Practice
-------------------------------
There are some behaviors that are technically undefined, but are otherwise consistent across implementations, such as how printf handles length modifiers on inappropriate conversions. For each conversion name there is a list of length modifiers that can apply to it. If a length modifier is applied to a conversion specifier that it doesn't apply to, then the behavior is undefined. For most conversions, LLVM's libc ignores any length modifier that doesn't apply. As an example, a conversion of %hf would be read as an f float conversion with the h length modifier. The h length modifier doesn't apply to floating point conversions and so %hf is the same as %f. There is one exception, which is the L length modifier on integer conversions. Many libcs handle the L length modifier like the ll length modifier when applied to integer conversions, despite L only applying to float conversions in the standard. LLVM's libc follows this convention because it is a useful feature that is simple to implement and has a predictable outcome for the user.

Interpreting the Standard's Reasoning
-------------------------------------
Often the standard will imply an intended behavior through what it states is undefined, such as in the case of printf's handling of the %% conversion. The %% conversion is used to write a % character, since it's used as the start of a conversion specifier. The standard specifies that %% must be the complete conversion specifier, and any options would make the conversion undefined. The conversion specifier %10% can therefore be interpreted as a % conversion with a width of 10, but the standard implies that this is not necessary. By making the options undefined, the standard implies a desired behavior for %% with options. The implied behavior is to ignore all options and always print %. This still leaves the behavior of %*% ambiguous, since the star normally consumes an argument to be used as the width. Since % conversions ignore the width, it would be reasonable to not read the argument in this case, but it would add additional complexity to the parsing logic. For that reason, the implementation in LLVM's libc will consume an argument for %*%, although the value is ignored. Adding additional logic for unreasonable edge cases, such as this one, is unnecessary.

Ignoring Bug-For-Bug Compatibility
----------------------------------
Any long running implementations will have bugs and deviations from the standard. Hyrum's Law states that “all observable behaviors of your system will be depended on by somebody” which includes these bugs. An example of a long-standing bug is glibc's scanf float parsing behavior. The behavior is specifically defined in the standard, but it isn't adhered to by all libc implementations. There is a longstanding bug in glibc where it incorrectly parses the string 100er and this caused the C standard to add that specific example to the definition for scanf. The intended behavior is for scanf, when parsing a float, to parse the longest possibly valid prefix and then accept it if and only if that complete parsed value is a float. In the case of 100er the longest possibly valid prefix is 100e but the float parsed from that string is only 100. Since there is no number after the e it shouldn't be included in the float, so scanf should return a parsing error. For LLVM's libc it was decided to follow the standard, even though glibc's version is slightly simpler to implement and this edge case is rare. Following the standard must be the first priority, since that's the goal of the library. If there is no standard, then matching another implementation (even bug-for-bug) may be necessary, but before you implement an unstandardized function first consider if anyone will actually use it at all.

Design Decisions
================

Resizable Tables for hsearch
----------------------------
The POSIX.1 standard does not delineate the behavior consequent to invoking hsearch or hdestroy without prior initialization of the hash table via hcreate. Furthermore, the standard does not specify the outcomes of successive invocations of hsearch absent intervening hdestroy calls. Libraries such as MUSL and Glibc do not apply checks to these scenarios, potentially leading to memory corruption or leakage. Conversely, FreeBSD's libc and Bionic automatically initialize the hash table to a minimal size if it is found uninitialized, and proceeding to destroy the table only if initialization has occurred. This approach also avoids redundant table allocation if an initialized hash table is already present. Given that the hash table starts with a minimal size, resizing becomes necessary to accommodate additional user insertions. LLVM's libc mirrors the approach of FreeBSD's libc and Bionic, owing to its enhanced robustness and user-friendliness. Notably, such resizing behavior itself aligns with POSIX.1 standards, which explicitly permit implementations to modify the capacity of the hash table.

Path without Leading Slashs in shm_open
----------------------------------------
POSIX.1 leaves that when the name of a shared memory object does not begin with a slash, the behavior is implementation defined. In such cases, the shm_open in LLVM libc is implemented to behave as if the name began with a slash.

Handling of NULL arguments to the 's' format specifier
------------------------------------------------------
The C standard does not specify behavior for ``printf("%s", NULL)``. We will
print the string literal ``(null)`` unless using the
``LIBC_COPT_PRINTF_NO_NULLPTR_CHECKS`` option described in :ref:`printf
behavior<printf_behavior>`.
TODO: Move this to printf_behavior.

Unknown Math Rounding Direction
-------------------------------
The C23 standard states that if the value of the ``rnd`` argument of the
``fromfp``, ``ufromfp``, ``fromfpx`` and ``ufromfpx`` functions is not equal to
the value of a math rounding direction macro, the direction of rounding is
unspecified. LLVM's libc chooses to use the ``FP_INT_TONEAREST`` rounding
direction in this case.

Non-const Constant Return Values
--------------------------------
Some libc functions, like ``dlerror()``, return ``char *`` instead of ``const char *`` and then tell the caller they promise not to to modify this value. Any modification of this value is undefined behavior.

Unrecognized ``clockid_t`` values for ``pthread_rwlock_clock*`` APIs
----------------------------------------------------------------------
POSIX.1-2024 only demands support for ``CLOCK_REALTIME`` and ``CLOCK_MONOTONIC``. Currently,
as in LLVM libc, if other clock ids are used, they will be treated as monotonic clocks.

PThread SpinLock Destroy
------------------------
POSIX.1 Issue 7 updates the spinlock destroy behavior description such that the return code for
uninitialized spinlock and invalid spinlock is left undefined. We follow the recommendation as in
POSIX.1-2024, where EINVAL is returned if the spinlock is invalid (here we only check for null pointers) or
EBUSY is returned if the spinlock is currently locked. The lock is poisoned after a successful destroy. That is,
subsequent operations on the lock object without any reinitialization will return EINVAL.

Strftime
--------
In the C Standard, it provides a list of modifiers, and the conversions these
are valid on. It also says that a modifier on an unspecified conversion is
undefined. For LLVM-libc, the conversion is treated as if the modifier isn't
there.

If a struct tm with values out of the normal range is passed, the standard says
the result is undefined. For LLVM-libc, the result may be either the normalized
value (e.g. weekday % 7) or the actual, out of range value. For any numeric
conversion where the result is just printing a value out of the struct
(e.g. "%w" prints the day of the week), no normalization occurs ("%w" on a
tm_wday of 32 prints "32"). For any numeric conversion where the value is
calculated (e.g. "%u" prints the day of the week, starting on monday), the
value is normalized (e.g. "%u" on a tm_wday of 32 prints "4"). For conversions
that result in strings, passing an out of range value will result in "?".

Posix adds padding support to strftime, but says "the default padding character
is unspecified." For LLVM-libc, the default padding character is ' ' (space)
for all string-type conversions and '0' for integer-type conversions. Composite
conversions pass the padding to the first (leftmost) conversion. In practice
this is always a numeric conversion, so it pads with '0'. For the purposes of
padding, composite conversions also assume the non-leading conversions have
valid inputs and output their expected number of characters. For %c this means
that the padding will be off if the year is outside of the range -999 to 9999.

The %e conversion is padded with spaces by default, but pads with 0s if the '0'
flag is set.

Posix also adds flags and a minimum field width, but leaves unspecified what
happens for most combinations of these. For LLVM-libc:
An unspecified minimum field width defaults to 0.
More specific flags take precedence over less specific flags (i.e. '+' takes precedence over '0')
Any conversion with a minimum width is padded with the padding character until it is at least as long as the minimum width.
Modifiers are applied, then the result is padded if necessary.
Any composite conversion will pass along all flags to the component conversions.

a64l and l64a
-------------
These functions convert to and from a posix-specified base64 encoding. There are
a few cases left undefined. For a64l, the behavior is undefined if the input
pointer (s) is a null pointer. For LLVM-libc this will cause a null pointer
dereference. It's also undefined if the input pointer to a64l wasn't generated
by l64a. For LLVM-libc, if the user passes a valid base 64 string, it will be
parsed as normal. For l64a it's unspecified what happens if the input value is
negative. For LLVM-libc, all inputs to l64a are treated as unsigned 32 bit ints.
Additionally, the return of l64a is in a thread-local buffer that's overwritten
on each call.
