MyTAP 0.01
==========

MyTAP is a unit testing framework for MySQL 5.x written using fuctions and
procedures. It includes a collection of TAP-emitting assertion functions, as
well as the ability to integrate with other TAP-emitting test frameworks.

Installation
============

To install MyTAP into a MySQL database, just run `mytap.sql`:

    mysql -u root < mytap.sql

This will install all of the assertion functions, as well as a cache table,
into a database named "tap".

MyTAP Test Scripts
==================

Here's an example of how to write a MyTAP test script:

    -- Start a transaction.
    BEGIN;

    -- Plan the tests.
    SELECT tap.plan(1);

    -- Run the tests.
    SELECT tap.pass( 'My test passed, w00t!' );

    -- Finish the tests and clean up.
    CALL tap.finish();
    ROLLBACK;

Note how the TAP test functions are reference from another database so as to
keep them separate from your application database.

Now you're ready to run your test script!

    % mysql -u root --disable-pager --batch --raw --skip-column-names --unbuffered --database test --execute 'source test.sql'
    1..1
    ok 1 - My test passed, w00t!

Using MyTAP
===========

The purpose of MyTAP is to provide a wide range of testing utilities that
output TAP. TAP, or the "Test Anything Protocol", is an emerging standard for
representing the output from unit tests. It owes its success to its format as
a simple text-based interface that allows for practical machine parsing and
high legibility for humans. TAP started life as part of the test harness for
Perl but now has implementations in C/C++, Python, PHP, JavaScript, Perl,
PostgreSQL, and now MySQL.

I love it when a plan comes together
------------------------------------

Before anything else, you need a testing plan. This basically declares how
many tests your script is going to run to protect against premature failure.

The preferred way to do this is to declare a plan by calling the `plan()`
function:

    SELECT tap.plan( 42 );

There are rare cases when you will not know beforehand how many tests your
script is going to run. In this case, you can declare that you have no plan.
(Try to avoid using this as it weakens your test.)

    CALL tap.no_plan();

Often, though, you'll be able to calculate the number of tests, like so:

    SELECT plan( COUNT(*) )
      FROM foo;

At the end of your script, you should always tell MyTAP that the tests have
completed, so that it can output any diagnostics about failures or a
discrepancy between the planned number of tests and the number actually run:

    CALL tap.finish();

Test names
----------

By convention, each test is assigned a number in order. This is largely done
automatically for you. However, it's often very useful to assign a name to
each test. Would you rather see this?

      ok 4
      not ok 5
      ok 6

Or this?

      ok 4 - basic multi-variable
      not ok 5 - simple exponential
      ok 6 - force == mass * acceleration

The latter gives you some idea of what failed. It also makes it easier to find
the test in your script, simply search for "simple exponential".

Many test functions take a name argument. It's optional, but highly suggested
that you use it.

I'm ok, you're not ok
---------------------

The basic purpose of MyTAP--and of any TAP-emitting test framework, for that
matter--is to print out either "ok #" or "not ok #", depending on whether a
given test succeeded or failed. Everything else is just gravy.

All of the following functions return "ok" or "not ok" depending on whether
the test succeeded or failed.

### `ok( boolean, description )` ###
### `ok( boolean )` ###

    SELECT tap.ok( @this = @that, @description );

This function simply evaluates any expression (`@this = @that` is just a
simple example) and uses that to determine if the test succeeded or failed. A
true expression passes, a false one fails. Very simple.

For example:

    SELECT tap.ok( 9 ^ 2 = 81,    'simple exponential' );
    SELECT tap.ok( 9 < 10,        'simple comparison' );
    SELECT tap.ok( 'foo' ~ '^f',  'simple regex' );
    SELECT tap.ok( active = true, name ||  widget active' )
      FROM widgets;

(Mnemonic:  "This is ok.")

The `@description` is a very short description of the test that will be printed
out. It makes it very easy to find a test in your script when it fails and
gives others an idea of your intentions. The description is optional, but we
*very* strongly encourage its use.

Should an `ok()` fail, it will produce some diagnostics:

    not ok 18 - sufficient mucus
    #     Failed test 18: "sufficient mucus"

Furthermore, should the boolean test result argument be passed as a `NULL`,
`ok()` will assume a test failure and attach an additional diagnostic:

    not ok 18 - sufficient mucus
    #     Failed test 18: "sufficient mucus"
    #     (test result was NULL)

### `eq( anyelement, anyelement, description )` ###
### `eq( anyelement, anyelement )` ###
### `isnt_eq( anyelement, anyelement, description )` ###
### `isnt_eq( anyelement, anyelement )` ###

    SELECT tap.eq(   @this, @that, @description );
    SELECT tap.isnt_eq( @this, @that, @description );

Similar to `ok()`, `eq()` and `isnt_eq()` compare their two arguments with
`=` AND `<>`, respectively, and use the result of that to determine if the
test succeeded or failed. So these:

    -- Is the ultimate answer 42?
    SELECT tap.eq( ultimate_answer(), 42, 'Meaning of Life' );

    -- foo() doesn't return empty
    SELECT tap.isnt_eq( foo(), '', 'Got some foo' );

are similar to these:

    SELECT tap.ok(   ultimate_answer() =  42, 'Meaning of Life' );
    SELECT tap.isnt( foo()             <> '', 'Got some foo'    );

(Mnemonic: "This is that." "This isn't that.")

*Note:* `NULL`s are not treated as unknowns by `eq()` or `isnt_eq()`. That
is, if `@this` and `@that` are both `NULL`, the test will pass, and if only
one of them is `NULL`, the test will fail.

So why use these test functions? They produce better diagnostics on failure.
`ok()` cannot know what you are testing for (beyond the description), but
`eq()` and `isnt_eq()` know what the test was and why it failed. For
example this test:

    SELECT tap.eq( 'waffle', 'yarblokos', 'Is foo the same as bar?' );

Will produce something like this:

    # Failed test 17:  "Is foo the same as bar?"
    #         have: waffle
    #         want: yarblokos

So you can figure out what went wrong without re-running the test.

You are encouraged to use `eq()` and `isnt_eq()` over `ok()` where
possible.

### `pass( description )` ###
### `pass()` ###
### `fail( description )` ###
### `fail()` ###

    SELECT tap.pass( @description );
    SELECT tap.fail( @description );

Sometimes you just want to say that the tests have passed. Usually the case is
you've got some complicated condition that is difficult to wedge into an
`ok()`. In this case, you can simply use `pass()` (to declare the test ok) or
`fail()` (for not ok). They are synonyms for `ok(1, @description)` and
`ok(0, @description)`.

Use these functions very, very, very sparingly.

The Schema Things
=================

Need to make sure that your database is designed just the way you think it
should be? Use these test functions and rest easy.

A note on comparisons: MyTAP uses a simple equivalence test (`=`) to compare
identifier names.

### `has_table( database, table, description )` ###

    SELECT has_table(DATABASE(), 'sometable', 'I got sometable');

This function tests whether or not a table exists in a database. The first
argument is a database name, the second is a table name, and the third is the
test description. If you want to test for a table in the current database, use
the `DATABASE()` function to specify the current databasen name. If you omit
the test description, it will be set to "Table ':database'.':table' should
exist".

No Test for the Wicked
======================

There is more to MyTAP. Oh *so* much more! You can output your own
[diagnostics](#Diagnostics). You can write [conditional
tests](#Conditional+Tests) based on the output of [utility
functions](#Utility+Functions). You can [batch up tests in
functions](#Tap+That+Batch). Read on to learn all about it.

Diagnostics
-----------

If you pick the right test function, you'll usually get a good idea of what
went wrong when it failed. But sometimes it doesn't work out that way. So here
we have ways for you to write your own diagnostic messages which are safer
than just `\echo` or `SELECT foo`.

### `diag( text )` ###

Returns a diagnostic message which is guaranteed not to interfere with
test output. Handy for this sort of thing:

    -- Output a diagnostic message if the collation is not en_US.UTF-8.
    SELECT tap.diag(concat(
         'These tests expect CHARACTER_SET_DATABASE to be en_US.UTF-8,\n',
         'but yours is set to ', VARIABLE_VALUE, '.\n',
         'As a result, some tests may fail. YMMV.'
    ))
      FROM information_schema.global_variables
     WHERE VARIABLE_NAME = 'CHARACTER_SET_DATABASE'
       AND VARIABLE_VALUE <> 'utf-8'

Which would produce:

    # These tests expect CHARACTER_SET_DATABASE to be en_US.UTF-8,
    # but yours is set to latin1.
    # As a result, some tests may fail. YMMV. 

Utility Functions
-----------------

Along with the usual array of testing, planning, and diagnostic functions,
pTAP provides a few extra functions to make the work of testing more pleasant.

### `mytap_version()` ###

    SELECT mytap_version();

Returns the version of MyTAP installed in the server. The value is `NUMERIC`,
and thus suitable for comparing to a decimal value.

Compose Yourself
================

So, you've been using MyTAP for a while, and now you want to write your own
test functions. Go ahead; I don't mind. In fact, I encourage it. How? Why,
by providing a function you can use to test your tests, of course!

But first, a brief primer on writing your own test functions. There isn't much
to it, really. Just write your function to do whatever comparison you want. As
long as you have a boolean value indicating whether or not the test passed,
you're golden. Just then use `ok()` to ensure that everything is tracked
appropriately by a test script.

For example, say that you wanted to create a function to ensure that two text
values always compare case-insensitively. Sure you could do this with
`eq()` and the `LOWER()` function, but if you're doing this all the time,
you might want to simplify things. Here's how to go about it:

    DROP FUNCTION IF EXITS lc_is
    DELIMITER //
    CREATE FUNCTION lc_is (have TEXT, want TEXT, descr TEXT)
    RETURNS TEXT
    BEGIN
        IF LOWER(have) = LOWER(want) THEN
            RETURN ok(1, descr);
        END IF;
        RETURN concat(ok( 0, descr ), '\n', diag(concat(
               '    Have: ', have,
             '\n    Want: ', want
        )));
    END //

    DELIMITER ;


Yep, that's it. The key is to always use MyTAP's `ok()` function to guarantee
that the output is properly formatted, uses the next number in the sequence,
and the results are properly recorded in the database for summarization at
the end of the test script. You can also provide diagnostics as appropriate;
just append them to the output of `ok()` as we've done here.

Of course, you don't have to directly use `ok()`; you can also use another
MyTAP function that ultimately calls `ok()`. IOW, while the above example
is instructive, this version is easier on the eyes:

    CREATE FUNCTION lc_is ( have TEXT, want TEXT, descr TEXT )
    RETURNS TEXT
    BEGIN
         RETURN eq( LOWER(have), LOWER(want), descr);
    END //

But either way, let MyTAP handle recording the test results and formatting the
output.

Testing Test Functions
----------------------

Now you've written your test function. So how do you test it? Why, with this
handy-dandy test function!

### `check_test( test_output, is_ok, name, want_description, want_diag, match_diag )` ###

    SELECT check_test(
        lc_eq('This', 'THAT', 'not eq'),
        0,
        'lc_eq fail',
        'not eq',
        '    Want: this\n    Have: that'
    );

    SELECT check_test(
        lc_eq('This', 'THIS', 'eq'),
        1
    );

This function runs anywhere between one and three tests against a test
function. For the impatient, the arguments are:

* `@test_output` - The output from your test. Usually it's just returned by a
  call to the test function itself. Required.
* `@is_ok` - Boolean indicating whether or not the test is expected to pass.
  Required.
* `@name` - A brief name for your test, to make it easier to find failures in
  your test script. Optional.
* `@want_description` - Expected test description to be output by the test.
  Optional. Use an empty string to test that no description is output.
* `@want_diag` - Expected diagnostic message output during the execution of
  a test. Must always follow whatever is output by the call to `ok()`.
  Optional. Use an empty string to test that no description is output.
* `@match_diag` - Use `matches()` to compare the diagnostics rather than
  `@eq()`. Useful for those situations where you're not sure what will be
  in the output, but you can match it with a regular expression.

Now, on with the detailed documentation. At its simplest, you just pass in the
output of your test function (and it must be one and **only one** test
function's output, or you'll screw up the count, so don't do that!) and a
boolean value indicating whether or not you expect the test to have passed.
That looks something like the second example above.

All other arguments are optional, but I recommend that you *always* include a
short test name to make it easier to track down failures in your test script.
`check_test()` uses this name to construct descriptions of all of the tests it
runs. For example, without a short name, the above example will yield output
like so:

    not ok 14 - Test should pass

Yeah, but which test? So give it a very succinct name and you'll know what
test. If you have a lot of these, it won't be much help. So give each call
to `check_test()` a name:

    SELECT check_test(
        lc_eq('This', 'THIS', 'eq'),
        true,
        'Simple lc_eq test',
    );

Then you'll get output more like this:

    not ok 14 - Simple lc_test should pass

Which will make it much easier to find the failing test in your test script.

The optional fourth argument is the description you expect to be output. This
is especially important if your test function generates a description when
none is passed to it. You want to make sure that your function generates the
test description you think it should! This will cause a second test to be run
on your test function. So for something like this:

    SELECT check_test(
        lc_eq( ''this'', ''THIS'' ),
        true,
        'lc_eq() test',
        'this is THIS'
    );

The output then would look something like this, assuming that the `lc_eq()`
function generated the proper description (the above example does not):

    ok 42 - lc_eq() test should pass
    ok 43 - lc_eq() test should have the proper description

See how there are two tests run for a single call to `check_test()`? Be sure
to adjust your plan accordingly. Also note how the test name was used in the
descriptions for both tests.

If the test had failed, it would output a nice diagnostics. Internally it just
uses `eq()` to compare the strings:

    # Failed test 43:  "lc_eq() test should have the proper description"
    #         have: 'this is this'
    #         want: 'this is THIS'

The fifth argument, `@want_diag`, which is also optional, compares the
diagnostics generated during the test to an expected string. Such diagnostics
**must** follow whatever is output by the call to `ok()` in your test. Your
test function should not call `diag()` until after it calls `ok()` or things
will get truly funky.

Assuming you've followed that rule in your `lc_eq()` test function, see what
happens when a `lc_eq()` fails. Write your test to test the diagnostics like
so:

    SELECT * FROM check_test(
        lc_eq( ''this'', ''THat'' ),
        false,
        'lc_eq() failing test',
        'this is THat',
        '    Want: this\n    Have: THat
    );

This of course triggers a third test to run. The output will look like so:

    ok 44 - lc_eq() failing test should fail
    ok 45 - lc_eq() failing test should have the proper description
    ok 46 - lc_eq() failing test should have the proper diagnostics

And of course, it the diagnostic test fails, it will output diagnostics just
like a description failure would, something like this:

    # Failed test 46:  "lc_eq() failing test should have the proper diagnostics"
    #         have:     Have: this
    #     Want: that
    #         want:     Have: this
    #     Want: THat

If you pass in the optional sixth argument, `@match_diag`, the `@want_diag`
argument will be compared to the actual diagnostic output using `matches()`
instead of `eq()`. This allows you to use a regular expression in the
`@want_diag` argument to match the output, for those situations where some
part of the output might vary, such as time-based diagnostics.

I realize that all of this can be a bit confusing, given the various haves and
wants, but it gets the job done. Of course, if your diagnostics use something
other than indented "have" and "want", such failures will be easier to read.
But either way, *do* test your diagnostics!

To Do
-----
* Port lot of other assertion functions from [pgTAP](http://pgtap.org/).

Public Repository
-----------------

The source code for MyTAP is available on
[GitHub](http://github.com/theory/mytap/). Please feel free to fork and
contribute!

Author
------

[David E. Wheeler](http://justatheory.com/)

Credits
-------

* Michael Schwern and chromatic for Test::More.
* Adrian Howard for Test::Exception.

Copyright and License
---------------------

Copyright (c) 2010 David E. Wheeler. Some rights reserved.

Permission to use, copy, modify, and distribute this software and its
documentation for any purpose, without fee, and without a written agreement is
hereby granted, provided that the above copyright notice and this paragraph
and the following two paragraphs appear in all copies.

IN NO EVENT SHALL KINETICODE BE LIABLE TO ANY PARTY FOR DIRECT, INDIRECT,
SPECIAL, INCIDENTAL, OR CONSEQUENTIAL DAMAGES, INCLUDING LOST PROFITS, ARISING
OUT OF THE USE OF THIS SOFTWARE AND ITS DOCUMENTATION, EVEN IF KINETICODE HAS
BEEN ADVISED OF THE POSSIBILITY OF SUCH DAMAGE.

KINETICODE SPECIFICALLY DISCLAIMS ANY WARRANTIES, INCLUDING, BUT NOT LIMITED
TO, THE IMPLIED WARRANTIES OF MERCHANTABILITY AND FITNESS FOR A PARTICULAR
PURPOSE. THE SOFTWARE PROVIDED HEREUNDER IS ON AN "AS IS" BASIS, AND
KINETICODE HAS NO OBLIGATIONS TO PROVIDE MAINTENANCE, SUPPORT, UPDATES,
ENHANCEMENTS, OR MODIFICATIONS.