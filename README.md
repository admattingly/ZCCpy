# ZCCpy (early beta)
## A Python Productivity Tool for calling the IBM Common Cryptographic Architecture (CCA) and IBM PKCS #11 Callable Services (z/OS only) from z/OS, Linux on Z or Linux on x86
## Introduction
This tool drammatically simplifies the process of calling the IBM CCA and PKCS #11 Callable Services from Python by providing a module that understands how to call CCA and PKCS #11 verbs and only requires you to populate the required verb parameters, providing default values for the rest.  It knows the data type and length of each parameter, so integer parameters can be provided as Python integers, rather than having to convert them to 4 or 8-byte binary form.  Similarly text parameters (e.g. key labels) can be provided as a string of any length, which ZCCpy will automatically pad with spaces to the the required length (e.g. 64 characters for a key label).

## Installation (of the beta)
Upload the file, [`zccpy.py`](https://github.com/admattingly/ZCCpy/blob/main/zccpy.py), to your z/OS, Linux on Z, or Linux on x86_64 system as a text file.  Place it in the directory where you are developing your Python code, so it will be automatically visible to the Python "import" statement.
## Programming with ZCCpy
### The Basics
To use ZCCpy in your Python script, you must _import_ it.  For example:
```
import zccpy as z
```
To call a CCA verb consult the "CCA callable services" section, or to call a PKCS #11 verb consult the "PKCS #11 callable services" section of the "IBM z/OS Cryptographic Services ICSF Application Programmer's Guide" (see: https://www.ibm.com/docs/en/zos/3.1.0?topic=guide-cca-callable-services or https://www.ibm.com/docs/en/zos/3.1.0?topic=guide-pkcs-11-callable-services, or equivalent documentation for Linux) for a description of the parameters passed to each verb.  ZCCpy expects you to pass named arguments with the required inputs when you invoke the CCA function (using a lower case parameter name).  The function arguments must be named exactly as the parameters are named in the IBM documentation for the platform you are using (i.e. z/OS or Linux).  You do not need to specify arguments for optional input parameters or output-only parameters - ZCCpy automatically sets null values for these parameters and populates the returned Python dictionary with all the CCA verb's parameter values on return from calling each verb.
### Getting Help
The Python help function will provide a description of the CCA verb you are invoking.  For example:
```
import zccpy as z
help(z.csnbrngl)
```
returns:
```
Help on function csnbrngl in module zccpy:

csnbrngl(**kw)
    csnbrngl - Random Number Generate Long

    Parameter  (* denotes mandatory input parameter)     Type             Dirn
    ---------------------------------------------------- ---------------- ----
    return_code                                          Integer          Out
    reason_code                                          Integer          Out
    exit_data_length                                     Integer          Both
    exit_data                                            Binary           Both
    rule_array_count*                                    Integer          In
    rule_array*                                          String           In
    key_identifier_length                                Integer          In
    key_identifier                                       Key label/token  In
    random_number_length*                                Integer          In
    random_number                                        Binary           Out
```
### zcpack Function
Several verb parameters, particularly the _rule_array_ parameter, must be supplied as a concatenation of one or more fixed-length (typically 8-character) strings.  To make your code more legible, and to avoid errors specifying such parameters, zccpy supplies a built-in function called __zcpack__.

This function takes two arguments:
1. (Mandatory) a space-delimited string comprising the words to be concatenated as fixed-length components of the output string.
2. (Optional) the length of each fixed-length component of the output string. The default length of each component is 8 characters.

Words in the input string must be separated by a single space.

Here are some examples of __zcpack__ in action:
```
import zccpy as z
z.zcpack('')                   -> ''
z.zcpack('quick brown fox')    -> 'quick   brown   fox     '
z.zcpack('quick brown fox', 4) -> 'quicbrowfox '
#                                  |---+---|---+---|---+---|
```
