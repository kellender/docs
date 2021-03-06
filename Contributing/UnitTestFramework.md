# Seattle's Unit Test Framework--Writing Tests

(For information about running tests, visit [UnitTestFrameworkRunning Running Unit Tests with Seattle's Unit Test Framework].)

----



----



## Introduction
----

So, you are writing a piece of software whose API has been defined and maybe implemented, and now you would like to start testing its behavior. The component you are testing is a resource monitor. For the sake of simplicity, suppose there are only two methods implemented in this file.

```python

"""
File: monitor.py
"""

def add_host(host_name): ...
def list_hosts(): ...
```




## Naming Requirements
----

The first thing you need to do is create a test file. Every test file has to follow a very specific naming convention -- otherwise, it will fail to be registered by UTF. There are two components to it:

 * Module Name -- Everything that you will be writing for Seattle is a smaller component (or a module) of the entire Seattle framework. Thus, we need to give it an appropriate name.

 * Descriptor -- Is an optional attribute. It is used to distinguish between different test files writing the same module.

Putting this together, we get:

```python

ut_MODULE[_DESCRIPTOR].py

```



## Basics
----

Let's suppose that we've written a module called 'monitor.py'. We've written all of the code, and all we need are unit tests. 

Let's begin writing unit tests. You write them the same way you would any python executable. The only thing to remember is that by default your test file is not allowed to produce any output on stdout or stderr. This is how UTF is able to tell if your test file has run successfully.

The following test for the monitor module tests the adding of host names. We follow the utf naming conventions by using a module name of "monitor" and a descriptor of "testhostnames".

```python

"""
File: ut_monitor_testhostnames.py
"""

# import your module.
import monitor

# Test valid host names.
def test_valid_add():

  valid_host_names = ('google.com', '128.208.1.1', '127.0.0.1')
  
  for host_name in valid_host_names:
    monitor.add_host(host_name)

  available_hosts = monitor.list_hosts()
  if len(available_hosts) != len(valid_host_names):
    print 'Adding valid host names failed.'


# Test invalid host names.
def test_invalid_add():

  invalid_host_names = ('...', '0.0.0.0')
  for host_name in invalid_host_names:
    try: 
      monitor.add_host(host_name)
    except InvalidHostName:
      pass
    else: 
      print 'Adding invalid host did not raise an exception'

```



## Multiple Test Files
----

You might require more than one test for a module. To create multiple unit tests for one module, all you need to do is use the same naming convention we described earlier. In our case where we were testing the 'monitor' module, we used the name 'ut_monitor_testhostnames.py'. All we need to do is use the same module name. For example, you might use 'ut_monitor_secondtest.py' as your second unit test's name. Keep in mind that when choosing a descriptor for your filename, you should be as descriptive as possible (if you were testing disk usage, a good name for your test would be 'ut_monitor_diskusage.py'). 




## Repy
----

Another feature of UTF is the support for testing repy files.  

Writing repy test cases is pretty simple. Imagine you are writing a synchronization primitive (semaphore) which has four methods defined.

```python

def create_semaphore(): ...
def destroy_semaphore(handle): ...
def up(handle): ...
def down(handle): ...
```


And you decide to test it by creating an appropriate test file.

```python
"""
File: ut_semaphore_testrepy.py
"""
```
UTF has to be able to differentiate between Python and Repy. Therefore, there must be a way to instruct the testing framework that this file is suppose to be executed inside of the Repy Execution Environment. To achieve this you need to insert what is called a **pragma directive** at the beginning of the test file. Each pragma directive instructs UTF in how to behave.

Here is the general form of a pragma:
```
...

#pragma TYPE [ARGS]

...
```



For this specific purpose use ''#pragma repy'' directive. Whenever UTF encounters this directive, it knows it has to execute the test file inside of the Repy Execution Environment. This directive must be inserted on its own line, somewhere in the file (a good convention is to put it near the top).

The rest of the file is written as a regular Repy file. Just as with Python testing, by default your unit test will fail if there is anything written on standard out or standard error, so keep that in mind when writing your tests. 

```python

#pragma repy


def test_create():
def test_destroy():
def test_up():
def test_down():
```



### Repy Restrictions
----

Repy execution is always associated with a restrictions file. To use a specific restrictions file, append it to the pragma directive as an argument. As with the previous example, if no argument was provided to #pragma repy directive, UTF assumes default restrictions are to be used. Therefore,


```python

#pragma repy
```

is equivalent to:

```python

#pragma repy restrictions.default
```

For instance, if your test file is suppose to be executed using non-default restrictions (i.e. restrictions.callomit) you would be using #pragma repy restrictions.callomit (the restriction file we are using does not allow function calls).
 


## Standard Out/Error
----
Remember, by default UTF will cause a test to fail if there is anything written to standard out or standard error. However, if you need to instead distinguish between good output and bad output (for example if you're testing whether an error happens when it's supposed to), you can change the way the test evaluates using the "#pragma out" and/or "#pragma error" directives. UTF will check to see that the entire string of words following the directive is written to standard out/error. If the string was not there, the test will fail. Note that when you include one or more "#pragma out" or "#pragma error" instructions, the test stops caring about any other output, and simply checks to see if the output includes what you demanded that it include.

Example 1: This test passes because "Hello Out" appears somewhere in standard out. (Any other output is ignored.)
```python

# This is a repy file!
#pragma repy

# This must be a substring of standard out for the test to pass.
#pragma out Hello Out.

print 'Hello Out.'
print 'Some other unimportant stuff.'
```

Example 2: This test fails because even though standard out matches the pragma out direction, standard error is still expected not to contain anything.
```
# This must be a substring of standard out for the test to pass.
#pragma out Hello Out.

print 'Hello Out.'
raise Exception('Some error occurs and the test fails because we expect no errors!')
```

Example 3: This test passes even though there is an unexpected error.
```
#pragma error assertion is always true, perhaps remove parentheses?
assert(False, "Whoops.")
raise Exception('Some completely unrelated error occurs but the test still passes.")
```


If you include multiple "#pragma out" directives, then each of the strings you specify must appear somewhere in the output, in the order you specified them!

Example 5: This test fails because the specified strings don't all appear and in the same order. 

```
#pragma out Roses are red.
#pragma out Violets are blue.
#pragma out Sugar is sweet.

print 'Roses are red.'
print 'Sugar is sweet.'
print 'Violets are blue.'
```

Example 6: If no output is specified as an argument to the directive, UTF will ignore all output to standard out or standard error, depending on which pragma directive you used.

```
#pragma out

print 'This passes because standard out is completely ignored.'
```



## Setup/Shutdown Scripts
----
Some software modules require additional steps before and after the test execution. We refer to those as ''setup'' and ''shutdown'' scripts respectively.

Suppose we are testing an IO module with several functions defined.

```python

def open(file_path): ...
def read(handle) ...
def write(handle): ...
def truncate(handle): ...
def stat(handle): ...
...
```

And you decide to start testing. You notice that every test requires some sort of file setup and cleanup. Instead of doing the setup and cleanup before and after every test case, it is much easier to do this all at the beginning or all at the end of running the tests. So we introduce **setup and shutdown scripts**. 

Setup scripts will always run before any unit test, and shutdown scripts will always run after any unit test. To make a shutdown script, all you need to do is use the following naming convention **ut_<module name>_shutdown.py**. However, there are two types of setup scripts. One is for if all you need to do is change configurations (create a temporary directory, for example). In this case, name your setup file **ut_<module name>_setup.py**. If you need your setup program to run while simultaneously running your unit tests, name your setup file **ut_<module name>_subprocess.py**.   

A subprocess script will be signaled to exit by the caller closing its stdin.   You can do sys.stdin.read() to block until it is time to clean up.

The following is an example of a shutdown script:

```python

import os

# Remove the directory.
def remove_tmp():
  temporary_dir = '/tmp/io'
  os.rmdir(temporary_dir)



if __name__ == '__main__':
  remove_tmp()
```

