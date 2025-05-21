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
On z/OS, to call a CCA verb, consult the [`CCA callable services`](https://www.ibm.com/docs/en/zos/3.1.0?topic=guide-cca-callable-services) section, or to call a PKCS #11 verb consult the [`PKCS #11 callable services`](https://www.ibm.com/docs/en/zos/3.1.0?topic=guide-pkcs-11-callable-services) section of the "IBM z/OS Cryptographic Services ICSF Application Programmer's Guide" for a description of the parameters passed to each verb.
On Linux, to call a CCA verb, consult the [`CCA verbs`](https://www.ibm.com/docs/en/linux-on-systems?topic=82-cca-verbs) section of the "IBM Secure Key Solution with the Common Cryptographic Architecture Application Programmer's Guide".
ZCCpy expects you to pass named arguments with the required inputs when you invoke the CCA function (using  lower case parameter names).  The function arguments must be named exactly as the parameters are named in the IBM documentation for the platform you are using (i.e. z/OS or Linux).  You do not need to specify arguments for optional input parameters or output-only parameters - ZCCpy automatically sets null values for these parameters and populates the returned Python dictionary with all the CCA verb's parameter values on return from calling each verb.

When you call a verb using zccpy, a Python dictionary object is returned, containing every parameter of that verb.  Here is an example showing the use of CSNBRNGL (Random Number Generate Long) to generate an odd-parity 16-byte random value, which is then imported into a CCA token and stored in the CKDS:
```
import zccpy as z
import sys

# invoke CSNBRNGL and set Python variables for all the verbs parameters
globals().update(z.csnbrngl(rule_array_count=1,rule_array='ODD',random_number_length=16))
print(random_number.hex())

# create a skeleton token
r = z.csnbktb(key_type='CIPHER',rule_array=z.zcpack('DES INTERNAL DOUBLE NO-KEY KEY-PART'),rule_array_count=5)
rc = r['return_code']
if rc == 0:

    # import the random_number as the key value
    r=z.csnbkpi(rule_array=z.zcpack('FIRST KEYBUF16'),rule_array_count=2,key_part=random_number,key_identifier=r['key_token'])

    # finalise the key token
    r=z.csnbkpi(rule_array=z.zcpack('COMPLETE'),rule_array_count=1,key_identifier=r['key_identifier'])
    token = r['key_identifier']
    print(token.hex())

    # compute key check value
    r=z.csnbkyt(rule_array=z.zcpack('KEY-ENCD GENERATE ENC-ZERO'),rule_array_count=3,key_identifier=token)
    # display the KCV
    if sys.platform == 'zos':
        print(r['verification_pattern'][:3].hex())
    else:
        print(r['value_2'][:3].hex())

    # create a CKDS record to store the key (label might already exist, but we will overwrite if so)
    r=z.csnbkrc(key_label='MY.TEST.CIPHER')

    # write the key token to the CKDS
    r=z.csnbkrw(key_token=token,key_label=r['key_label'])
'''
The output from running this code on Linux should look something like this:
```
(CSNBRNGL - Random Number Generate Long           ) rc=0, reason=0
7adf7fb9ef31b30b79ad0e1923911532
(CSNBKTB  - Key Token Build                       ) rc=0, reason=0
(CSNBKPI  - Key Part Import                       ) rc=0, reason=0
(CSNBKPI  - Key Part Import                       ) rc=0, reason=0
010000000000c02010dff7568cb20d42d131facf682f9bc9814a38fcf1aa4840000371000341000000037100032100000000000000000000000000005151be8c
(CSNBKYT  - Key Test                              ) rc=0, reason=0
3200bc
(CSNBKRC  - DES Key Record Create                 ) rc=8, reason=44
    A record with a matching key label already exists in key storage.
(CSNBKRW  - DES Key Record Write                  ) rc=0, reason=0
```
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
