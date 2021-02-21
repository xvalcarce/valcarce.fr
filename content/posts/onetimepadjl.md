+++
title = "Julia - One-Time Pad (XOR)"
date = "2020-02-06"
type = ["posts","post"]
[ author ]
  name = "Xavier Valcarce"
+++

## Disclaimer

This is a pet project to help me understand the relatively new programming language [Julia](https://julialang.org). I don’t guarantee that the code in this blog post is optimal, nor respect all Julia conventions.

## Algorithm objective

An implementation of the one-time pad encryption algorithm, based on the XOR gate. The code should be able to encrypt any given file by generating a key of this file’s size and XORing the file with the key, bit-by-bit. We want the code to also be able to do the reverse; given a key and an encrypted file, run the XOR to get the original file.

## Code

### Part 1: Arguments and help

``` julia
HELP = "xore.jl usage:
    xore.jl [enc|dec] [filename [keyfile]]
    Options:
        enc : encrypt file. Generated file will be the filename.enc and generated key filename.key
        dec : decrypt file. [keyfile] argument required
        "

# Test arguments
@assert (length(ARGS)==2 || length(ARGS)==3) HELP
@assert (ARGS[1]=="enc" || ARGS[1]=="dec") HELP
@assert isfile(ARGS[2]) "Can not find provided file"
if ARGS[1]=="dec"
    @assert length(ARGS)==3 "Keyfile argument missing"
    @assert isfile(ARGS[3]) "Can not find provided keyfile"
end

# Filling variable
filename = ARGS[2]
extension = findlast(isequal('.'),ARGS[2])
if extension!=nothing
    file = filename[1:extension-1]
else
    file = filename
end
```

This the meta part of our program. Our program takes 2 or 3 arguments. The first one, either enc or dec set the mode of our program (decryption or encryption mode). The second one is the file to encrypt/decrypt. Finally, if the decryption mode is trigge, we need the file containing the key .
Some checks are made using assert to ensure that provided arguments, and arguments number, are correct.

### Part 2: I/O

Let’s now study how one can read a given file and store it into a variable. 

``` julia
# Read file to byte arrays
f = open(filename, "r")
data = UInt8[]
readbytes!(f,data,Inf)
close(f)
```

This open the file with the open function — argument r is for read only. We then create an array of type UInt8. Finally, we read the file by byte, and store them in the previously declared array. The Inf argument is to read to entire file — this way we don’t need to provide the length (number of byte) of our file.

### Part 3: Encryption

Here is the encryption section in which we generate the key, encrypt the file and save both the encrypted version of the file and the key.

``` julia
if ARGS[1]=="enc"
    # Genearting key of the file's size
    key = rand(UInt8,size(data)[1])

    # Generating encrypted file
    enc = UInt8[]
    for (i,d) in enumerate(data)
        push!(enc,xor(d,key[i]))
    end

    # Saving everything
    @assert !isfile(file*".scrt") "enc.scrt file already exist in current directory"
    enc_file = open(file*".scrt", "w")
    write(enc_file, enc)
    close(enc_file)

    @assert !isfile(file*".key") "enc.key file already exist in current directory"
    key_file = open(file*".key", "w")
    write(key_file, key)
    close(key_file)
```

The key is generated using the rand function. The key consists of an UInt8 array of the file’s size (number of byte).
We then proceed to the encryption by XOR-ing the file and the key, storing everything in the newly created enc array. The XOR bit-wise operation is done by the built-in xor function.
Finally, we save the encrypted file as well as the key using the name of the file (without the extension). A small check is made for the existence of such file, in order not to erase unintentionally potential files.

### Part 3: Decryption

``` julia
else
    # Read the keyfile
    k = open(ARGS[3], "r")
    key = UInt8[]
    readbytes!(k,key,Inf)
    close(k)

    # Decryption
    dec = UInt8[]
    for (i,d) in enumerate(data)
        push!(dec,xor(d,key[i]))
    end

    #Saving decrypted file
    dec_file = open(file*".clear", "w")
    write(dec_file, dec)
    close(dec_file)
end
```

This is the decryption mode. It works in a similar manner as the encryption process, no further explanation are required to understand this part of the code.
