# Writing tests for swupd-client
<br/>

When writing tests for swupd-client, the swupd test library (testlib.bash) should
be used. This will facilitate the creation of test environments and test objects,
and will also ensure that validations are performed in a consistent manner accross
tests. The swupd-client uses BATS as the test framework of choice, to discover and
run tests.
<br/>
To use the test library you just need to source it in your shell
```bash
$ source testlib.bash
```
 or load it in your test script.  
```bash
load "testlib"
```
<br/>

## Quickstart

1. Source testlib.bash from your terminal, this will load all the test functions
from the library into your current shell process:
```bash
$ source testlib.bash
```

2. If necessary, create a directory to group the new test script with other test
scripts of the same theme.
```bash
$ mkdir <some_test_theme>
$ cd <some_test_theme>
```

3. Create the new test script.  
```bash
$ generate_test <test_name>
```

A new test script will be generated in the current directory.  
```bash
#!/usr/bin/env bats

load "../testlib"

global_setup() {

	# global setup

}

test_setup() {

	# create_test_environment "$TEST_NAME"
	# create_bundle -n <bundle_name> -f <file_1>,<file_2>,<file_N> "$TEST_NAME"

}

test_teardown() {

	# destroy_test_environment "$TEST_NAME"
}

global_teardown() {

	# global cleanup

}

@test "<test description>" {

	run sudo sh -c "$SWUPD <swupd_command> $SWUPD_OPTS <command_options>"
	# <validations>

}
```

4. The global_setup() function contains commands that are run only once per test
script, it runs before any test in the script is executed.

5. The test_setup() function contains commands that should be executed before every
test in the test script. This is usually the best place for creating test
objects that are prerequisites for all tests in the script. The test library
provides many [fixtures](#test-fixtures) to create all these prerequisites easily.
A default value for the test_setup() function is already defined in the test library,
but this is a minimal definition, only a test environment with name $TEST_NAME gets
created, so if this is all your test needs you can then just remove the test_setup()
from the test script so the pre-defined one is used, otherwise you will need to
overwrite that function with your test prerequisites.

6. The test_teardown() function contains commands that should be executed after
every test in the test script. It is the most common place for cleaning up
test resources created for and by tests.
Similarly as with the test_setup(), there is a minimal definition of the test_teardown()
function already in the test library. In this definition, the test environment $TEST_NAME
gets deleted, in most tests this is all it needs to be done as part of the cleanup, so
if this is the case you can go ahead and remove the local test_teardown definition from the
test script so the pre-defined one is used, if you have more specific cleanup needs you will
need to overwrite the function.

6. The global_teardown() function contains commands that are run only once per test
script, it runs once all tests in the scripts have finished.

7. Every test within the test file starts with the @test identifier followed
by a description of the test. Add one or more tests to the script as needed.

8. From within each test, execute the command you want to validate by using
the `run` helper.

9. Validate that the program under test performed as expected by using one
or more of the [assertions](#assertions) provided by the test library.
<br/>

### Test Principles

* Every test needs to run in its own test environment so they don't interfere with
each other.
* Every test should clean up after itself.
* Tests should be atomic and test only one thing per test.
* Tests should be independent from each other, one should not need another
one in order to work.
* Test files from the same "theme" should be grouped within the theme directory.
Tests from the same type can optionally be added to the same test file, for
example:
  * bundle-add  # A theme
    * add-single-bundle.bats
      * @test "add a bundle"
      * @test "add a bundle that has only a directory"
      * @test "add a bundle that has many files"
      * @test "add a bundle that is already installed"
      * @test "add a bundle that is invalid"
      * etc.
    * add-multiple-bundles.bats
      * @test "add multiple bundles"
      * @test "add multiple bundles, one valid, one invalid"
      * etc.
  * bundle-remove  # Another theme
  * etc.
<br/>

### Running Tests
To run a specific test script locally:  
```bash
$ bats <theme_directory>/<test_script>.bats
```  

To run all tests from a theme locally:  
```bash
$ bats <theme_directory>
```  

To run all tests locally:
```bash
$ cd swupd-client/test/functional
$ bats *
```

Alternatively, to run all tests locally you can also use make:
```bash
$ cd swupd-client/
$ make check
```

To include tests to be run in the CI system:  
TBD  
<br/>

## Test Fixtures
### Test Environment
A test environment is nothing but a directory that contains the necessary file
structure that is used to emulate the existance of the following resources:
* a target file system <test_enviroment>/target-dir
* a local state directory to avoid conflicts with the system state directory
<test_enviroment>/state
* a remote system that provides content file downloads <test_enviroment>/web-dir
* the os-core bundle, since this bundle is required in every system it gets
created by default in the content download directory, and "installed" in the
target system

To create a test environment called "my_env" for a test you would run the following
command from within the test script.
```bash
# create a test environment for version 10 (default) 
create_test_environment my_env 
```
Or
```bash
# create a test environment for version 20
create_test_environment my_env 20
```

The test library provides the following functions for handling test environments:
* create_test_environment
* destroy_test_environment
<br/>

### Test Objects
Test objects are all those elements that need to be mocked up in order to automate
a test. The test library provides many functions for the creation and manipulation
of these test objects. By far, the most useful test object that can be created by
the test library are bundles. When creating a bundle using the test library, many
other required test objects are created as a side effect, all things that are
necessary for the bundle like files, directories, manifests, packs, etc.

Use the following command to create a bundle with two files called "test-bundle" in
the "my_env" test environment.
```bash
create_bundle -n test-bundle -f /foo/bar/test-file1,/baz/test-file2 my_env
```
By creating that bundle the following objects will also be created:
* A hashed directory (to be used for the /foo, /foo/bar and /baz directories)
* Two hashed files (test-file1 and test-file2, the content for these files is
randomly generated so hashes are different)
* A hashed tracking file for the bundle
* A bundle manifest
* A Manifest of Manifests (MoM)
* A tar file for each file, directory and manifest
* A zero pack for the bundle

Another example:
```bash
create_bundle -L -n another-bundle -d /some_dir -f /baz/test-file -l /test-link my_env
```
This bundle, besides having characteristics similar to the previous bundle will
have this new characteristics:
* One directory without any file (/some_dir)
* One link (test-link), this will also generate another extra file to which the
symbolic link is pointing to
* Since the -L flag was used, the bundle will not only be created in the directory
for content download, but it will also be installed in the target file system, this
is useful for tests that need a pre-installed bundle as prerequisite

The following are some of the functions provided by the test library to create and
handle test objects:
* create_bundle: creates a bundle
* create_dir: creates a hashed directory
* create_file: creates a hashed file
* create_link: creates a hashed file and a hashed link pointing to the file
* create_tar: creates a tar of the specified object (manifest, full file, etc.)
* create_manifest: creates an empty manifest (with initial headers)
* add_to_manifest: adds the specified object (directory, file, or link) to the
manifest, and creates/updates the manifest tar
* add_dependency_to_manifest: adds another bundle as dependency in the manifest,
and creates/updates the manifest tar
* remove_from_manifest: removes an object from the manifest, and updates the
manifest tar
* sign_manifest: signs the manifest using Swupd_Root.pem
* update_hashes_in_mom: after modifying manifests included in a MoM, this function
can be run to update the manifest hashes in the MoM
* get_hash_from_manifest: retrieves the hash of an object within a manifest
<br/>

## Assertions
The following assertions are included in the test library. These should be used to
perform the test validations.

*assert_status_is*  
passes if the exit status matches the provided one, fails otherwise  
Example:  
```bash
run <some_command>
assert_status_is 0
```

*assert_status_is_not*  
passes if the exit status does not match the provided one, fails otherwise  
Example:  
```bash
run <some_command>
assert_status_is_not 0
```

*assert_dir_exists*  
passes if the provided directory exists, fails otherwise  
Example:  
```bash
run <some_command>
assert_dir_exists /some/directory
```

*assert_dir_not_exists*  
passes if the provided directory does not exist, fails otherwise  
Example:  
```bash
run <some_command>
assert_dir_not_exists /some/directory
```

*assert_file_exists*  
passes if the provided file exists, fails otherwise  
Example:  
```bash
run <some_command>
assert_file_exists /some/file
```

*assert_file_not_exists*  
passes if the provided file does not exist, fails otherwise  
Example:  
```bash
run <some_command>
assert_file_not_exists /some/file
```

*assert_in_output*  
passes if the provided text is included in the command output, fails otherwise  
Examples:  
```bash
# one line strings
run <some_command>
assert_in_output "Successfully installed 1 bundle"

# multi-line strings
run <other_command>
expected_output=$(cat <<-EOM
  Some multi-ine text that needs to be present
  in the exact order
  some more lines, bla bla
EOM
)
assert_in_output "$expected_output"

# combination of both
run <yet_another_command>
expected_output=$(cat <<-EOM
  Some multi-ine text that needs to be present
  in the exact order
  some more lines, bla bla
EOM
)
assert_in_output "some literall text"
assert_in_output "$expected_output"
assert_in_output "more text to be checked"
```

*assert_not_in_output*  
passes if the provided text is not included in the command output, fails otherwise  
Examples:  
```bash
# one line strings
run <some_command>
assert_not_in_output "Error"
```

*assert_equal*  
passes if the two values provided are equal, fails otherwise  
Example:  
```bash
run <some_command>
assert_equal "some_value" "$my_variable"
```

*assert_not_equal*  
passes if the two values provided are not equal, fails otherwise  
Example:  
```bash
run <some_command>
assert_not_equal "$variable1" "$variable2"
```