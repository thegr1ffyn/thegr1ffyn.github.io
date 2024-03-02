---
title: BackToFlag - Reversing - AirTech 2024
date: '2024-03-01'
tags: ['backtoflag', 'reversing', 'airtech', 'ghidra', 'capture-the-flag']
draft: true
---
# BackToFlag - Reversing - AirTech 2024
### Description: 
This time doc was smart enough to save the seed value in a text file only problem is he accidently deleted config.txt which contained the key for password, i dont think there's any way to open this file OR IS THERE?

Author: CY30RG

### Solution:
We were given two files named `backtoflag` which is an `elf` file with a text file named `seed.txt` containing random numbers of `274520360673462036`.

I opened the file in Ghidra and saw the functions of `main` and `readSeedFromFile` of our need.

## Source Codes
The `main` function contains of following code.

```cpp
undefined8 main(void){
  char cVar1;
  basic_ostream *pbVar2;
  basic_string local_d8 [32];
  basic_string local_b8 [32];
  basic_string local_98 [32];
  basic_string local_78 [32];
  basic_string<char,std::char_traits<char>,std::allocator<char>> local_58 [47];
  allocator local_29;
  allocator *local_28;
  undefined4 local_20;
  undefined4 local_1c;
  
  local_28 = &local_29;
                    /* try { // try from 001026d8 to 001026dc has its CatchHandler @ 0010289c */
  std::__cxx11::basic_string<char,std::char_traits<char>,std::allocator<char>>::
  basic_string<std::allocator<char>>(local_58,"config.txt",&local_29);
                    /* try { // try from 001026e4 to 001026e8 has its CatchHandler @ 0010288b */
  local_1c = readSeedFromFile((basic_string *)local_58);
  std::__cxx11::basic_string<char,std::char_traits<char>,std::allocator<char>>::~basic_string
            (local_58);
  std::__new_allocator<char>::~__new_allocator((__new_allocator<char> *)&local_29);
  std::__cxx11::basic_string<char,std::char_traits<char>,std::allocator<char>>::basic_string();
                    /* try { // try from 00102725 to 00102753 has its CatchHandler @ 001028f3 */
  std::operator<<((basic_ostream *)std::cout,"Enter the password: ");
  std::operator>>((basic_istream *)std::cin,local_78);
  generatePassword[abi:cxx11]((int)register0x00000020 - 0x98);
  cVar1 = verifyPassword(local_78,local_98);
  if (cVar1 == '\0') {
    std::__cxx11::basic_string<char,std::char_traits<char>,std::allocator<char>>::basic_string();
                    /* try { // try from 00102795 to 001027d8 has its CatchHandler @ 001028cb */
    std::operator<<((basic_ostream *)std::cout,"Enter the key: ");
    std::operator>>((basic_istream *)std::cin,local_d8);
    local_20 = deriveSeed(local_d8);
    generateFlag[abi:cxx11]((int)register0x00000020 - 0xb8);
                    /* try { // try from 001027ed to 00102818 has its CatchHandler @ 001028b7 */
    pbVar2 = std::operator<<((basic_ostream *)std::cout,"Congratulations! Here\'s your flag: ");
    pbVar2 = std::operator<<(pbVar2,local_b8);
    std::basic_ostream<char,std::char_traits<char>>::operator<<
              ((basic_ostream<char,std::char_traits<char>> *)pbVar2,
               std::endl<char,std::char_traits<char>>);
    std::__cxx11::basic_string<char,std::char_traits<char>,std::allocator<char>>::~basic_string
              ((basic_string<char,std::char_traits<char>,std::allocator<char>> *)local_b8);
    std::__cxx11::basic_string<char,std::char_traits<char>,std::allocator<char>>::~basic_string
              ((basic_string<char,std::char_traits<char>,std::allocator<char>> *)local_d8);
  }
  else {
                    /* try { // try from 0010284d to 00102863 has its CatchHandler @ 001028df */
    pbVar2 = std::operator<<((basic_ostream *)std::cout,"Incorrect password. Access denied.");
    std::basic_ostream<char,std::char_traits<char>>::operator<<
              ((basic_ostream<char,std::char_traits<char>> *)pbVar2,
               std::endl<char,std::char_traits<char>>);
  }
  std::__cxx11::basic_string<char,std::char_traits<char>,std::allocator<char>>::~basic_string
            ((basic_string<char,std::char_traits<char>,std::allocator<char>> *)local_98);
  std::__cxx11::basic_string<char,std::char_traits<char>,std::allocator<char>>::~basic_string
            ((basic_string<char,std::char_traits<char>,std::allocator<char>> *)local_78);
  return 0;
}
```

The `readSeedFromFile` contains of the following code:

```cpp
uint readSeedFromFile(basic_string *param_1){
  char cVar1;
  basic_ostream *pbVar2;
  long *plVar3;
  uint local_22c;
  basic_string local_228 [536];
  
  std::basic_ifstream<char,std::char_traits<char>>::basic_ifstream(local_228,(_Ios_Openmode)param_1)
  ;
  cVar1 = std::basic_ifstream<char,std::char_traits<char>>::is_open();
  if (cVar1 != '\x01') {
                    /* try { // try from 00102394 to 0010243b has its CatchHandler @ 0010245f */
    pbVar2 = std::operator<<((basic_ostream *)std::cerr,"Error: Unable to open file: ");
    pbVar2 = std::operator<<(pbVar2,param_1);
    std::basic_ostream<char,std::char_traits<char>>::operator<<
              ((basic_ostream<char,std::char_traits<char>> *)pbVar2,
               std::endl<char,std::char_traits<char>>);
                    /* WARNING: Subroutine does not return */
    exit(1);
  }
  plVar3 = (long *)std::basic_istream<char,std::char_traits<char>>::operator>>
                             ((basic_istream<char,std::char_traits<char>> *)local_228,&local_22c);
  cVar1 = std::basic_ios<char,std::char_traits<char>>::operator!
                    ((basic_ios<char,std::char_traits<char>> *)
                     ((long)plVar3 + *(long *)(*plVar3 + -0x18)));
  if (cVar1 != '\0') {
    pbVar2 = std::operator<<((basic_ostream *)std::cerr,"Error: Unable to read seed from file: ");
    pbVar2 = std::operator<<(pbVar2,param_1);
    std::basic_ostream<char,std::char_traits<char>>::operator<<
              ((basic_ostream<char,std::char_traits<char>> *)pbVar2,
               std::endl<char,std::char_traits<char>>);
                    /* WARNING: Subroutine does not return */
    exit(1);
  }
  std::basic_ifstream<char,std::char_traits<char>>::~basic_ifstream
            ((basic_ifstream<char,std::char_traits<char>> *)local_228);
  return local_22c;
}
```

Running the elf file, we get the following output:
```
└─$ ./backtoflag
Error: Unable to read seed from file: config.txt
```

We need to bypass two conditions here. In the `readSeedfromFile` function, there is a local variable named `local_22c` which is a `uint` that can hold 32 bytes of data at max. Its an unsigned integer used to store the seed value read from the file.

### File Opening:

The function attempts to open the file specified by `param_1` using `std::basic_ifstream`. If the file cannot be opened (`is_open()` returns `false`), it prints an error message to `std::cerr` and exits the program with an error code of 1.

### Reading Seed Value:

If the file is successfully opened, the function attempts to read a seed value from the file using `std::basic_istream::operator>>`. It stores the result in `local_22c`.

### Bypassing First Condition
We can simply touch a file named `config.txt` and add random numbers uptil 10 in the same directory.
```
└─$ cat config.txt
1234567
```
Now when we run the file, we get new output where it asks for a password.

```
└─$ ./backtoflag
Enter the password: 111
Incorrect password. Access denied.
```

### Bypassing Second Condition
In the `main` function, we have a loophole where we can bypass the password condition at the following line

```cpp
  if (cVar1 != '\0') {
    pbVar2 = std::operator<<((basic_ostream *)std::cerr,"Error: Unable to read seed from file: ");
```

Ghidra also provides us with a functionality to edit Assembly code to tweak the application to our need. The above mentioned code comes up to the following.

```x86asm
        001023fa 74  4a           JZ         LAB_00102446
```
Here we can replace the JZ instruction with a JNZ instruction. This will cause the jump to be taken if the zero flag is not set, effectively bypassing the original condition of checking the password.

Editing this and then saving this as a new elf file in Ghidra using `Export Program`.

Executing the file and entering random numbers in when it asks for password bypasses the password check condition.
```
└─$ ./backtoflagnew
Enter the password: 123
Enter the key: 
```

Now it asks us for a key which has already been given to us in the form of `seed.txt` above, entering it gives us the flag.
```
└─$ ./backtoflagnew
Enter the password: 123
Enter the key: 274520360673462036
Congratulations! Here's your flag: AT24{JKYZCRLCYJ}
```

We can summarize our learnings as follows:
The vulnerability in the provided scenario lies in the inadequate input validation and error handling in the `readSeedFromFile` function, as well as the possibility to modify the control flow of the program to bypass password authentication.

## Summary

### Vulnerabilities:

1. **File Opening Vulnerability**:
   - The `readSeedFromFile` function attempts to open the file specified by the `param_1` parameter without adequate validation of the input. If an attacker provides a malicious or unexpected file path, it could lead to unintended file operations, potentially allowing the attacker to read sensitive data or perform unauthorized actions.

2. **Control Flow Vulnerability**:
   - In the `main` function, the program checks if the seed value can be successfully read from the file. If successful, it proceeds to prompt the user for a password. However, an attacker could bypass this password prompt by modifying the control flow of the program, such as replacing the `JZ` (jump if zero) instruction with a `JNZ` (jump if not zero) instruction. This allows the attacker to skip the password check entirely and proceed to obtain the flag without authentication.

### Exploitation:

1. **Bypassing File Opening Check**:
   - By creating a file named `config.txt` with arbitrary contents (e.g., random numbers), an attacker can bypass the file opening check in the `readSeedFromFile` function. This allows the attacker to proceed to the next stage of the program without encountering the expected error condition.

2. **Control Flow Modification**:
   - By editing the assembly code of the program, an attacker can replace the `JZ` instruction with a `JNZ` instruction, effectively bypassing the password authentication check in the `main` function. This allows the attacker to directly obtain the flag without providing the correct password.
