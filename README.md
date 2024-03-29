# https-github.com-zenorocha-clipboard.js
Clipboardnodejs

                ================================
                 A Subversion Testing Framework
                ================================


The three goals of Subversion's automated test-suite:

      1.  It must be easy to run.
      2.  It must be easy to understand the results.
      3.  It must be easy to add new tests.



Definition of an SVN "test program"
-----------------------------------

A Subversion test program is any executable that contains a number of
sub-tests it can run.  It has a standard interface:

1.  If run with a numeric argument N, the program runs sub-test N.

2.  If run with the argument `--list', it will list the names of all sub-tests.

3.  If run with no arguments, the program runs *all* sub-tests.

4.  The program returns either 0 (success) or 1 (if any sub-test failed).

5.  Upon finishing a test, the program reports the results in a format
    which is both machine-readable (for the benefit of automatic
    regression tracking scripts), and human-readable (for the sake of
    painstaking grovelling by hand in the dead of night):

      (PASS | FAIL): (argv[0]) (argv[1]): (description)

For example,

  [sussman@newton:~] ./frobtest 2
  PASS: frobtest 2: frobnicating fragile data
  [sussman@newton:~] 

Note that no particular programming language is required to write a
set of tests;  they just needs to export this user interface.



How to write new C tests
------------------------

The C test framework tests library APIs, both internal and external.

All test programs use a standard `main' function.  You write .c files
that contain only test functions --- you should not define your own
`main' function.

Instead, your code should define an externally visible array
`test_funcs', like this:

    /* The test table.  */
    struct svn_test_descriptor_t test_funcs[] =
    {
      SVN_TEST_NULL,
      SVN_TEST_PASS(test_a),
      SVN_TEST_PASS(test_b),
      SVN_TEST_PASS(test_c),
      SVN_TEST_NULL
    };

In this example, `test_a', `test_b', and `test_c' are the names of
test functions.  The first and last elements of the array must be
SVN_TEST_NULL.  The first SVN_TEST_NULL is there to leave room for
Buddha.  The standard `main' function searches for the final
SVN_TEST_NULL to determine the size of the array.

Instead of SVN_TEST_PASS, you can use SVN_TEST_XFAIL to declare that a
test is expected to fail. The status of such tests is then no longer
marked as PASS or FAIL, but rather as XFAIL (eXpected FAILure) or
XPASS (uneXpected PASS).

The purpose of XFAIL tests is to confirm that a known bug still
exists. When you see such a test uneXpectedly PASS, you've probably
fixed the bug it tests for, even if that wasn't your intention. :-)
XFAIL is not to be used as a way of testing a deliberately invalid
operation that is expected to fail when Subversion is working
correctly, nor as a place-holder for a test that is not yet written.

Each test function conforms to the svn_test_driver_t prototype:

        svn_error_t *f (const char **MSG,
                        svn_boolean_t MSG_ONLY
                        apr_pool_t *POOL);

When called, a test function should first set *MSG to a brief (as in,
half-line) description of the test.  Then, if MSG_ONLY is TRUE, the
test should immediately return SVN_NO_ERROR.  Else it should perform a
test.  If the test passes, the function should return SVN_NO_ERROR;
otherwise, it should return an error object, built using the functions
in svn_error.h.

Once you've got a .c file with a bunch of tests and a `test_funcs'
array, you should link it against the `libsvn_tests_main.la' libtool
library, in this directory, `subversion/tests'.  That library provides
a `main' function which will check the command-line arguments, pick
the appropriate tests to run from your `test_funcs' array, and print
the results in the standard way.


How to write new Python tests
-----------------------------

The python test framework exercises the command-line client as a
"black box".

To write python tests, please look at the README file inside the
cmdline/ subdirectory.


When to write new tests
-----------------------

In the world of CVS development, people have noticed that the same
bugs tend to recur over and over.  Thus the CVS community has adopted
a hard-and-fast rule that whenever somebody fixes a bug, a *new* test
is added to the suite to specifically check for it.  It's a common
case that in the process of fixing a bug, several old bugs are
accidentally resurrected... and then quickly revealed by the test
suite.

This same rule applies to Subversion development: ** If you fix a
bug, write a test for it. **


When to file a related issue
----------------------------

By definition, if you write a new test which is set to XFail, then it
assumed that the test is for a known bug.  In these cases it is
recommended that you associate an issue in the issue tracker with the
XFailing test.  This ensures that the issue tracker is the authoritative
list of known bugs -- see http://subversion.tigris.org/issue-tracker.html.
You may need to create a new issue if one doesn't already exist.

For C tests simply add a comment noting any associated issue:

    /* This is for issue #3234. */
    static svn_error_t *
    test_copy_crash(const svn_test_opts_t *opts,
                    apr_pool_t *pool)
    {
      apr_array_header_t *sources;
      svn_opt_revision_t rev;
      .
      .

For Python tests use the @Issue() decorator (a summary comment of the
issue never hurts either):

    #---------------------------------------------------------------------
    # Test for issue #3657 'dav update report handler in skelta mode can
    # cause spurious conflicts'.
    @Issue(3657)
    @XFail()
    def dav_skelta_mode_causes_spurious_conflicts(sbox):
      "dav skelta mode can cause spurious conflicts"
      .
      .

Of course it isn't *always* necessary to create an associated issue.
If a the fix for an new XFailing test is imminent, you are probably
better off simply fixing the bug and moving on.  Use common sense, but
when in doubt associate a new issue.


What not to test
----------------

Regression tests are for testing interface promises.  This might
include semi-private interfaces (such as the non-public .h files
inside module subdirs), but does not include implementation details
behind the interfaces.  For example, this is a good way to test
svn_fs_txn_name:

      /* Test that svn_fs_txn_name fulfills its promise. */
      char *txn_name = NULL;
      SVN_ERR = svn_fs_txn_name (&txn_name, txn, pool);
      if (txn_name == NULL)
        return fail();

But this is not:

      /* Test that the txn got id "0", since it's the first txn. */
      char *txn_name = NULL;
      SVN_ERR = svn_fs_txn_name (&txn_name, txn, pool);
      if (txn_name && (strcmp (txn_name, "0") != 0))
        return fail();

During development, it may sometimes be very convenient to
*temporarily* test implementation details via the regular test suite.
It's okay to do that, but please remove the test when you're done and
make sure it's clearly marked in the meantime.  Since implementation
details are not interface promises, they might legitimately change --
and when they change, that test will break.  At which point whoever
encountered the problem will look into the test suite and find the
temporary test you forgot to remove.  As long as it's marked like
this...

      /* Temporary test for debugging only: Test that the txn got id
       * "0", since it's the first txn. 
       * NOTE: If the test suite is failing because of this test, then
       * just remove the test.  It was written to help me debug an
       * implementation detail that might have changed by now, so its
       * failure does not necessarily mean there's anything wrong with
       * Subversion. */
      char *txn_name = NULL;
      SVN_ERR = svn_fs_txn_name (&txn_name, txn, pool);
      if (txn_name && (strcmp (txn_name, "0") != 0))
        return fail();

...then they won't have wasted much time.


What's here
-----------

   * svn_test_main.c
     [shared library "libsvn_tests_main"]
     A standardized main() function to drive tests.  Link this into
     your automated test-programs.

   * svn_test_editor.c
     [shared library "libsvn_tests_editor"]
     An editor for testing drivers of svn_delta_edit_fns_t.  This
     editor's functions simply print information to stdout.

   * cmdline/
     A collection of python scripts to test the command-line client.


`make check`
------------

The file `build.conf' (at the top level of the tree) defines a
[test-scripts] section.  These are a list of scripts that will be run
whenever someone types `make check`.

Each script is expected to output sub-test information as described in
the first section of this document;  the `make check` rule scans for
FAIL codes, and logs all the sub-test output into a top-level file
called `tests.log'.

If you write a new C executable that contains subtests, be sure to add
a build "target" under the TESTING TARGETS section of build.conf.

If you write a new python-script, be sure to add to the [test-scripts]
section.


Testing Over DAV
----------------

Please see subversion/tests/cmdline/README for how to run the
command-line client test suite against a remote repository.

Conclusion
----------

Our test suite...


  1.  ...must be easy to run.

      * run `make check`

  2.  ...must be easy to understand the results.

      * test programs output standardized messages
      * all messages are logged
      * `make check` only displays errors (not successes!)

  3.  ...must be easy to add new tests.

      * add your own sub-test to an existing test program, or
      * add a new test program using template C or python code.
