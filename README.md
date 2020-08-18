---
title: "cmdfun"
output: github_document
---

<!-- badges: start -->
[![Lifecycle: experimental](https://img.shields.io/badge/lifecycle-experimental-orange.svg)](https://www.tidyverse.org/lifecycle/#experimental)
[![Project Status: Active – The project has reached a stable, usable state and is being actively developed.](https://www.repostatus.org/badges/latest/active.svg)](https://www.repostatus.org/#active)
[![R build status](https://github.com/snystrom/cmdr/workflows/R-CMD-check/badge.svg)](https://github.com/snystrom/cmdr/actions)
[![Codecov test coverage](https://codecov.io/gh/snystrom/cmdr/branch/master/graph/badge.svg)](https://codecov.io/gh/snystrom/cmdr?branch=master)
<!-- badges: end -->

## A simple framework for building shell interfaces

The purpose of `cmdfun` is to significantly reduce the overhead involved in
wrapping shell programs in R. The tools are intended to be intuitive and
lightweight enough to use for data scientists trying to get things done quickly,
but robust and full-fledged enough for developers to extend them to more
advanced use cases.

## Vocabulary to describe operations
Briefly, `cmdfun` captures R function arguments (**args**) and converts them to
a vector of commandline **flags**. 

The `cmdfun` framework provides three
mechanisms for capturing function arguments:

 - `cmd_args_dots()` captures all arguments passed to `...`
 - `cmd_args_named()` captures all keyword arguments defined by the user
 - `cmd_args_all()` captures both named + dot arguments
 
`cmd_args_to_flags` converts the captured arguments to a parsed list of flag/value
pairs. This output can be useful for additonal handling of special flag
assignments from within R.

`cmd_list_crystallize` converts the output of `cmd_args_to_flags` to a vector of
commandline-style flags. This output can be directly fed to `system2` or
`processx`.

Together, they can be used to build user-friendly R interfaces to shell programs
without having to manually implement all commandline flags in R functions.

## Install

```r
remotes::install_github("snystrom/cmdfun")
```

## Examples

```r
library(magrittr)
library(cmdfun)
```

variables defined in `...` are converted to character vector of flags appropriate for shell commands.

```r
myFunction <- function(...){

  flags <- cmd_args_dots() %>%
    cmd_args_to_flags()
  
  return(flags)
}
```


```r
(flagsList <- myFunction(flag = "var", bool_flag = TRUE))
```

```
## $flag
## [1] "var"
## 
## $bool_flag
## [1] ""
```

This list can be passed to `cmd_list_crystallize` to generate a vector suitable for `system2` to build shell commands. 


```r
flagsList %>% 
  cmd_list_crystallize()
```

```
## [1] "-flag"      "var"        "-bool_flag"
```


```r
shellCut <- function(text, ...){

  flags <- cmd_args_dots() %>%
    cmd_args_to_flags() %>% 
    cmd_list_crystallize()

	system2("cut", flags, stdout = T, input = text)

}
```


```r
shellCut("hello_world", f = 2, d = "_") 
```

```
## [1] "world"
```

Multiple values can be passed to arguments using vectors

```r
shellCut("hello_world_hello", f = c(1,3), d = "_") 
```

```
## [1] "hello_hello"
```

### Boolean flags are passed as bool operators


```r
shell_ls <- function(dir = ".", ...){
  flags <- cmd_args_dots() %>% 
    cmd_args_to_flags() %>% 
    cmd_list_crystallize()
  
  system2("ls", c(dir, flags), stdout = T)
}
```


```r
shell_ls("R")
```

```
## [1] "cmd_args.R"       "macros.R"         "parse_help.R"     "utils.R"         
## [5] "utils_internal.R"
```


```r
shell_ls("R", l = T)
```

```
## [1] "total 48"                                                                  
## [2] "-rw-r--r-- 1 snystrom its_employee_psx  6293 Aug 18 15:45 cmd_args.R"      
## [3] "-rw-r--r-- 1 snystrom its_employee_psx 10310 Aug 18 15:53 macros.R"        
## [4] "-rw-r--r-- 1 snystrom its_employee_psx  5825 Aug 18 15:54 parse_help.R"    
## [5] "-rw-r--r-- 1 snystrom its_employee_psx  9517 Aug 18 15:46 utils.R"         
## [6] "-rw-r--r-- 1 snystrom its_employee_psx  7433 Aug 18 15:55 utils_internal.R"
```

### Named vectors can be used to provide user-friendly aliases for single-letter flags

```r
shell_ls_alias <- function(dir = ".", ...){
  
  argsDict <- c("long" = "l")
  
  flags <- cmd_args_dots() %>% 
    cmd_args_to_flags(argsDict) %>% 
    cmd_list_crystallize()
  
  system2("ls", c(dir, flags), stdout = T)
}
```


```r
shell_ls_alias("R", long = T)
```

```
## [1] "total 48"                                                                  
## [2] "-rw-r--r-- 1 snystrom its_employee_psx  6293 Aug 18 15:45 cmd_args.R"      
## [3] "-rw-r--r-- 1 snystrom its_employee_psx 10310 Aug 18 15:53 macros.R"        
## [4] "-rw-r--r-- 1 snystrom its_employee_psx  5825 Aug 18 15:54 parse_help.R"    
## [5] "-rw-r--r-- 1 snystrom its_employee_psx  9517 Aug 18 15:46 utils.R"         
## [6] "-rw-r--r-- 1 snystrom its_employee_psx  7433 Aug 18 15:55 utils_internal.R"
```


```r
shellCut_alias <- function(text, ...){

  argsDict <- c("sep" = "d",
                "fields" = "f")
    
	flags <- cmd_args_dots() %>%
		cmd_args_to_flags(argsDict) %>% 
	  cmd_list_crystallize()

	system2("cut", flags, stdout = T, input = text)
}
```


```r
shellCut_alias("hello_world", fields = 2, sep = "_") 
```

```
## [1] "world"
```

## Abstraction of command path handling

A common pattern when designing shell interfaces is to ask the user to give an
absolute path to the target shell utility. It is common to pass this information
from the user to R by using either R environment variables defined in `.Renviron`,
using options (set with `option()`, and got with `getOption()`), having the user
explicitly pass the path in the function call, or failing this, using a default
install path.

`cmd_path_handle()` is a macro which returns a function that returns a valid
path to the target by heirarchically searching a series of possible locations.

For example, to build an interface to the "MEME" suite, which is by default
installed to `~/meme/bin`, one could build the following:


```r
handle_meme_path <- cmd_path_handle(default_path = "~/meme/bin")
```

```
## Error in cmd_list_drop_named(., "utils"): could not find function "cmd_list_drop_named"
```

```r
handle_meme_path()
```

```
## Error in handle_meme_path(): could not find function "handle_meme_path"
```

To only search the R environment variable "MEME_PATH", one could build:

```r
handle_meme_path <- cmd_path_handle(environment_var = "MEME_PATH")
```

```
## Error in cmd_list_drop_named(., "utils"): could not find function "cmd_list_drop_named"
```



```r
# Without environment variable defined
handle_meme_path()
```

```
## Error in handle_meme_path(): could not find function "handle_meme_path"
```


```r
# With environment varialbe defined
Sys.setenv("MEME_PATH" = "~/meme/bin")
handle_meme_path()
```

```
## Error in handle_meme_path(): could not find function "handle_meme_path"
```

Multiple arguments can be used, and they will be searched from most-specific, to most-general. 

```r
handle_meme_path <- cmd_path_handle(environment_var = "MEME_PATH",
                                       default_path = "~/meme/bin")
```

```
## Error in cmd_list_drop_named(., "utils"): could not find function "cmd_list_drop_named"
```

For example, if "MEME_PATH" is invalid on my machine, the handler will return
the default path as long as the default is also valid on my machine.

```r
Sys.setenv("MEME_PATH" = "bad/path")
handle_meme_path()
```

```
## Error in handle_meme_path(): could not find function "handle_meme_path"
```

### Support for tool utilities
Some software, like the MEME suite is distributed as several binaries located in
a common directory. To allow interface builders to officially support specific binaries, 
each binary can be defined as a "utility" within the build path.

Here, I will include two tools from the MEME suite, AME, and DREME (distributed
as binaries named "ame", and "dreme").

```r
handle_meme_path <- cmd_path_handle(environment_var = "MEME_PATH",
                                       default_path = "~/meme/bin",
                                       utils = c("dreme", "ame"))
```

```
## Error in cmd_list_drop_named(., "utils"): could not find function "cmd_list_drop_named"
```

handler functions have two optional arguments: `path` and `util`. `path` acts as
an override to the defaults provided when building the handler. User-provided
path variables will always be used instead of provided defaults. This is to
catch problems from the user and not cause unexpected user-level behavior.

```r
handle_meme_path("bad/path")
```

```
## Error in handle_meme_path("bad/path"): could not find function "handle_meme_path"
```

`util` specifies which utility path to return (if any). The path handler will
throw an error if the utility is not found in any of the specified locations.

```r
handle_meme_path(util = "dreme")
```

```
## Error in handle_meme_path(util = "dreme"): could not find function "handle_meme_path"
```

List all utility paths (but don't check if they're valid). Useful for writing user-facing install checking functions.

```r
handle_meme_path(util = TRUE)
```

```
## Error in handle_meme_path(util = TRUE): could not find function "handle_meme_path"
```
`cmdfun` provides a preexisting utility for this, however. The `cmd_install_check`
function can be lightly wrapped by package builders to verify and print a
user-friendly series of checks for a valid tool install. it takes as input the
output of `build_package_handler` and an optional user-override `path`.

Here I build a function for checking a users `meme` install.

```r
check_meme_install <- function(path = NULL){
  cmd_install_check(handle_meme_path, path = path)
}
```



```r
# searches default meme search locations
check_meme_install()
```

```
## Error in cmd_install_check(handle_meme_path, path = path): object 'handle_meme_path' not found
```



```r
# uses user override
check_meme_install('bad/path')
```

```
## Error in cmd_install_check(handle_meme_path, path = path): object 'handle_meme_path' not found
```

If you want to write your own install checker instead of using the
`cmd_install_check` function, `cmdfun` also provides the `cmd_ui_file_exists` function
for printing pretty status messages.

```r
cmd_ui_file_exists("bad/file")
```

```
## x bad/file
```

```r
cmd_ui_file_exists("~/meme/bin")
```

```
## ✓ ~/meme/bin
```

### Internal install validators
`cmdfun` also provides a macro `cmd_install_is_valid()` to construct
functions returning boolean values testing for an install path. These are useful
in function logic, or package development for setting conditional examples or
function hooks that depend on a command install. `cmd_install_is_valid()`
takes a path handler function as input, so any `options`, `.Renviron`, or
default install location logic propagates to these funtions as well.


```r
meme_installed <- cmd_install_is_valid(handle_meme_path)
meme_installed()
```

```
## [1] FALSE
```

This also works on utils defined during path hanlder construction.

```r
ame_installed <- cmd_install_is_valid(handle_meme_path, util = "ame")
ame_installed()
```

```
## [1] FALSE
```


## Bringing it all together
Using a `cmd_args_` family function to get and convert function arguments to
commandline flags. The path handler returns the correct `command` call which can
be passed to `system2` or `processx` along with the flags generated from `cmd_args_to_flags`.

This makes for a robust shell wrapper without excess overhead.

In the `runDreme` function below, the user can pass any valid `dreme` argument
using the rules for command args defined above to `...`. Allowing `meme_path` as
a function argument and passing it to `handle_meme_path` allows the user to
override the default search path which is: `MEME_PATH` environment variable,
followed by the `~/meme/bin` default install.


```r
handle_meme_path <- cmd_path_handle(environment_var = "MEME_PATH",
                                       default_path = "~/meme/bin",
                                       utils = c("dreme", "ame"))
```

```
## Error in cmd_list_drop_named(., "utils"): could not find function "cmd_list_drop_named"
```

```r
runDreme <- function(..., meme_path = NULL){
  flags <- cmd_args_dots() %>% 
    cmd_args_to_flags() %>% 
    cmd_list_crystallize()
  
  command <- handle_meme_path(path = meme_path, util = "dreme")
  
  system2(command, flags)
}
```

Commands can now run through `runDreme` by passing flags as function arguments.

```r
runDreme(h = T)
```

```
## Error in handle_meme_path(path = meme_path, util = "dreme"): could not find function "handle_meme_path"
```

If users have issues with the install, they can run `check_meme_install()` to verify the tools are being detected by R.

## Restrict argument matching
each `cmd_args_` family function accepts a character vector of names to `keep` or
`drop` arguments which will restrict command argument matches to values in
`keep` (or ignore those in `drop`). As of now, `keep` and `drop` are mutually
exclusive.

This can be useful to allow only some function arguments to be captured as
flags, while others can be used for function logic.


```r
myFunction <- function(arg1, arg2, print = T){
  flags <- cmd_args_named(keep = c("arg1", "arg2")) %>% 
    cmd_args_to_flags() %>% 
    cmd_list_crystallize()
  
  ifelse(print, print("printing"), print("nothing"))
  
  return(flags)
}

myFunction(arg1 = "blah", arg2 = "blah")
```

```
## [1] "printing"
```

```
## [1] "-arg1" "blah"  "-arg2" "blah"
```



```r
myFunction(arg1 = "blah", arg2 = "blah", F)
```

```
## [1] "nothing"
```

```
## [1] "-arg1" "blah"  "-arg2" "blah"
```

## Manupulating flag list objects
For the most part, the [purrr](https://purrr.tidyverse.org/) library is the most
useful toolkit for operations on list objects.

`cmdfun` provides additional helper functions to handle common manipulations.

`cmd_list_drop` operates on flag lists to drop all entries corresponding to a
certain name, specific name/value pairs, or by index position. Can be useful for
ignoring setting certain flags if the user set them to a specific value.
Conversely, `cmd_list_keep` functions identically but for keeping entries.

```r
myFunction <- function(arg1, arg2){
  flags <- cmd_args_named() %>% 
    cmd_args_to_flags() %>% 
    cmd_list_drop(c("arg2" = "baz")) %>% 
    cmd_list_crystallize()
  
  return(flags)
}

myFunction(arg1 = "foo", arg2 = "bar")
```

```
## [1] "-arg1" "foo"   "-arg2" "bar"
```

```r
myFunction(arg1 = "foo", arg2 = "baz")
```

```
## [1] "-arg1" "foo"
```

## Expecting output files
Sometimes a commandline function returns multiple output files you want to check for after the run.

`cmd_files_exist` accepts a vector or list of files & checks that they exist.

`cmdfun` additionally provides a few convenience functions for generating lists
of expected files. `cmd_output_expect` generates combinations of extension/prefix
file names. The output can be passed to `cmd_files_exist` which will error if
a file isn't found on the filesystem.

```r
cmd_output_expect(ext = c("txt", "xml"), prefix = "outFile")
```

```
## $txt
## [1] "./outFile.txt"
## 
## $xml
## [1] "./outFile.xml"
```


```r
cmd_output_expect(ext = "txt", prefix = c("outFile", "outFile2", "outFile3"))
```

```
## $outFile
## [1] "./outFile.txt"
## 
## $outFile2
## [1] "./outFile2.txt"
## 
## $outFile3
## [1] "./outFile3.txt"
```

## Error checking user input

When using `cmdfun` to write lazy shell wrappers, the user can easily mistype a
commandline flag since there is not text completion. Some programs behave
unexpectedly when flags are typed incorrectly, and for this reason return
uninformative error messages. `cmdfun` has built-in methods to automatically
populate a list of valid flags from a command's help-text.

Alternatively, package builders could pass a vector of allowed flag names to
check against if they didn't want to parse help text. The goal is maximum flexibility.

The following example demonstrates how to parse help text (in this case from
`tar`) into a vector of allowed flags. This vector is compared to the user-input
flags (`user_input_flags` below), and tries to identify misspelled function
arguments.

Here, the user has accidentally used the argument `delte` instead of `delete`.
`cmdfun` tries to be helpful and identify the misspelling for the user.

```r
user_input_flags <- c("delte")

system2("tar", "--help", stdout = T) %>% 
  cmd_help_parse_flags() %>% 
  gsub(",", "", .) %>% 
  # Compares User-input flags to parsed commandline flags
  # returns flags that match based on edit distance
  cmd_help_flags_similar(user_input_flags) %>% 
  # Prints error message suggesting the most similar flag name
  cmd_help_flags_suggest()
```

```
## Error: Invalid flags. Did you mean:
## "delete" instead of: "delte"
```


## Unsafe operations

**WARNING:** It's still possible to do unsafe operations as follows, so please be careful how you build system calls.


```r
shellCut_unsafe <- function(text, ...){

  flags <- cmd_args_dots() %>%
    cmd_args_to_flags() %>% 
    cmd_list_crystallize()

	system2("echo", c(text , "|", "cut", flags), stdout = T)

}

shellCut_unsafe("hello_world", f = 2, d = "_ && echo test")
```

```
## [1] "world" "test"
```

**NOTE** even if when setting `stdout = TRUE` the second command doesn't appear
in the output, it will still have run. 

A more extreme example of what can happen is here, where `~/deleteme.txt` will be removed silently.

I promise I'll get around to sanitizing user input eventually.

```r
shellCut("hello_world", f = 2, d = "_ && rm ~/deleteme.txt")
```
