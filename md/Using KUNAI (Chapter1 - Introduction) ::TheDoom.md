> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [k0deless.github.io](https://k0deless.github.io/kunai/kunai1/)

> UC3M Cybersecurity students webpage, where you'll find our research posts, papers and so on.

by [Fare9](https://k0deless.github.io/#Fare9) struct dexheader_t { std::uint8_t magic[8]; std::int32_t checksum; std::uint8_t signature[20]; std::uint32_t file_size; std::uint32_t header_size; std::uint32_t endian_tag; std::uint32_t link_size; std::uint32_t link_off; std::uint32_t map_off; std::uint32_t string_ids_size; std::uint32_t string_ids_off; std::uint32_t type_ids_size; std::uint32_t type_ids_off; std::uint32_t proto_ids_size; std::uint32_t proto_ids_off; std::uint32_t field_ids_size; std::uint32_t field_ids_off; std::uint32_t method_ids_size; std::uint32_t method_ids_off; std::uint32_t class_defs_size; std::uint32_t class_defs_off; std::uint32_t data_size; std::uint32_t data_off; };

These first structure contains a value that **Kunai** will check in order to know we are working with a possible **DEX** file, this is the _magic_ field, which contains the word _dex_ together with the version of the **DEX** file used (currently version _039_). Then we have values like:

*   _file_size_: size of the DEX file we are analyzing (**Kunai** uses this for applying some checks).
*   _header_size_: size of this structure.
*   _endian_tag_: indicates the type of _endiness_ used by the **DEX** file, currently **Kunai** just supports _Little-Endian_.

The next fields are key of the file format as they indicate the real offset from the beginning of the file where we can find one of the structures, and also the size of that structure.

If we return to the previous image, we can see that most of the data in the **DEX** file format is based on indexes, and these indexes point to other fields, finally, at the end of the chain of indexes we find indexes to the _string ids_, these _string ids_ contains offsets to the _string data_ which contains all the strings present in the file, these are: _class names, method names, field names, types, etc_. Then the other values present inside of each structure are constants that will have different meanings depending on the field.

With **Kunai** we will parse all these structures, and we will be able to recover them, access them, and also print them in order to analyze the **DEX** files.

Installation
------------

First of all, we will see the installation process of **Kunai** in our system, the first thing we will do is to pull the project from its repository, and then we will pull its submodules in order to compile the whole project:

```
$ https://github.com/Fare9/KUNAI-static-analyzer.git
...
$ cd KUNAI-static-analyzer
$ git submodule update --init --rebase --recursive
...


```

**Kunai** has a script for installing the dependencies, this is mainly written for _Debian_ based system, but probably you can follow the commands from the file _make.sh_, however here we will run the script:

```
$ ./make.sh dependencies
[+] Checking for package libspdlog-dev
[-] Not found zip folder, cloning from repo...
fatal: destination path './external/zip' already exists and is not an empty directory.
[sudo] password for KUNAI-User:               
Reading package lists... Done
Building dependency tree       
Reading state information... Done
The following additional packages will be installed:
  cmake-data libjsoncpp1 librhash0
...
[+] Checking for libzip.so
[!] Finished installing dependencies


```

In my case, I already had the folder of the external dependecy, but in case it’s not downloaded, the script will install it. We should have now everything for starting compilation of **Kunai**.

The script _make.sh_ allows you also to compile **Kunai** both in a _release_ mode and in a _debug_ mode, latter is thought for debugging purposes, but in our case we can directly compile in release mode, by default 4 jobs will be used for compilation of the project, feel free to modify the **MAKE_JOBS** variable in order to set as many jobs as you want:

Or if you want to compile in debug mode:

In this case _g++_ will be used, but _make.sh_ has the option to use _clang++_ as the compiler, if you are more _LLVM_ friendly, please go ahead using it:

If you prefer, you can instead make use of the _Makefile_ for doing all these steps:

```
$ # compile and specify 4 processes, do not show compiler messages
$ make -j4 --silent
$ # compile in debug mode
$ make -j4 debug --silent
$ # use clang++ for compilation
$ CXX=clang++ make -j4 --silent


```

Finally for the installation, we can use again the script _make.sh_ or the _Makefile_:

```
$ # use the script
$ ./make.sh install
$ # use the Makefile
$ make install


```

You can optionally compile the _test_ Java projects that can be used for testing the different parts of **Kunai**, for doing that you’ll need the _Java compiler_ (**javac**) and the compiler **d8** to compile the _.class_ files to _.dex_

```
$ make tests
current_dir=/opt/sources/KUNAI-static-analyzer
Compiling test-assignment-arith-logic
cd ./tests/test-assignment-arith-logic/ && javac --release 8 Main.java && d8 Main.class && mv classes.dex Main.dex
Compiling test-const_class
cd ./tests/test-const_class/ && javac --release 8 Main.java && d8 Main.class && mv classes.dex Main.dex
Compiling test-try-catch
cd ./tests/test-try-catch/ && javac --release 8 Main.java && d8 Main.class && mv classes.dex Main.dex
Compiling test-graph
cd ./tests/test-graph/ && javac --release 8 Main.java && d8 Main.class && mv classes.dex Main.dex
Compiling test-cyclomatic-complexity
cd ./tests/test-cyclomatic-complexity/ && javac --release 8 Main.java && d8 Main.class && mv classes.dex Main.dex
Compiling test-vm
cd ./tests/test-vm/ && javac --release 8 PCodeVM.java VClass.java && \
	d8 VClass.class && \
	mv classes.dex VClass.dex &&\
	d8 PCodeVM.class &&\
	mv classes.dex PCodeVM.dex
Compiling test-modexp
cd ./tests/test-modexp && javac --release 8 Main.java && d8 Main.class && mv classes.dex Main.dex


```

Now you can start working with **Kunai** and start writing your first programs using the library. For our first project we will start digging with the headers of a **DEX** file using **Kunai** and we will see how it’s possible to do that without disassembly the file or without applying further analysis.

Writing our first “analyzer”
----------------------------

Let’s start writing our first code for our first **Dalvik** analyzer using **Kunai** as a supporting library. As we have **Kunai** already installed in our system, we will not need to modify the compilation script, nor the _Makefile_, we will just need to link the library to our binary file during compilation.

Our first program it will just print the classes using the _C++_ output stream _std::cout_, **Kunai** has for most of its classes a defined behaviour for the operator **«**, but for starting to learn about the structures, we will try to avoid using it as much as possible (you can try just printing out one of the headers using _std::cout_).

For the sake of simplicity, the program will not work with **APK** files but with **DEX**.

First part of our code, first we will have to add the _include_ of the headers needed in our program, these will include one for analyzing **DEX** files with **Kunai** and one for managing of the _logging_ level, we will also start writing the _main_ function, and add some error check code:

```
#include <iostream>
#include <filesystem>

#include <KUNAI/DEX/dex.hpp>
#include <spdlog/spdlog.h>

int
main(int argc, char **argv)
{
    if (argc != 2)
    {
        std::cerr << "[-] USAGE: " << argv[0] << " <dex file>" << std::endl;
        return -1;
    }


```

The code is pretty simple and straightforward to understand, the second _include_ contains all we need to work with **DEX** files with **Kunai** and the next one will be used to set the _logging_ level from the library. After that, we have the _main_ function with the check of the arguments.

Now we can move to a most interesting part:

```
// watch only info and error messages from Kunai
spdlog::set_level(spdlog::level::info);

auto logger = KUNAI::LOGGER::logger();

logger->info("Starting the analysis of {}", argv[1]);

std::ifstream dex_file;
dex_file.open(argv[1], std::ios::binary);

auto fsize = dex_file.tellg();
dex_file.seekg(0, std::ios::end);
fsize = dex_file.tellg() - fsize;
dex_file.seekg(0);

auto dex_object = KUNAI::DEX::get_unique_dex_object(dex_file, fsize);

if (!dex_object->get_parsing_correct())
{
    logger->error("Error analyzing {}, maybe DEX file is not correct...", argv[1]);
    return -1;
}


```

Here we have the main part of the code, here we load the **DEX** file into a _dex_object_, the call to _get_unique_dex_object_ will create a _unique_ptr_ of the **DEX** object type, with this we will not have to care about releasing the memory. As I said at the beginning, **Kunai** will only take care of parsing the **DEX** file, but we should control that the parsing process was correct to alert the user, to do that we can call to the function _get_parsing_correct_, in case an error happened the function will return _false_ and we can finish the program.

Now we can finally print some information about the **DEX** file:

```
auto dex_parser = dex_object->get_parser();

logger->info("Dex version number: {}", dex_parser->get_header_version());
logger->info("Dex version string: {}", dex_parser->get_header_version_str());

auto dex_header = dex_parser->get_header();

logger->info("File size: {:d}, Checksum: 0x{:x}, Header size: {:d}", 
    dex_header->get_dex_header().file_size, 
    (std::uint32_t)dex_header->get_dex_header().checksum, 
    dex_header->get_dex_header().header_size);

logger->info("String ids size: {:d}, String ids offset: 0x{:x}", dex_header->get_dex_header().string_ids_size, dex_header->get_dex_header().string_ids_off);


```

The main object for working with the **DEX** format is the **DexParser** object, to get that object, we can call to the function _get_parser_ from the **DEX** object. The **DexParser** object allows us to get the different parts from the **DEX** structure, as a simple example, and for starting, I will only take the **DexHeader** object, this object contains a structure we can access with all the fields from the header, I copied this structure previously with the name of _dexheader_t_. Instead of using the _std::cout_ output from _C++_, in this case we are using the _logger_ from **Kunai**.

Let’s print now information about the classes, and the methods defined in the **DEX** file:

```
auto vector_class_defs = dex_parser->get_classes_def_item();

logger->info("[!] ClassDefs");

for (auto &class_def : vector_class_defs)
{
    auto class_data = class_def->get_class_idx();

    if (!class_data)
        continue;

    logger->info("[+] ClassDef:");

    logger->info("Type of object: {}, name: {}", class_data->print_type(), class_data->get_name());
    
    auto source_file = class_def->get_source_file_idx();
    if (source_file)
        logger->info("Source file of the class: {}", *source_file);
    
    // enum that can be checked!
    logger->info("Access Flag: {}", class_def->get_access_flags());

    logger->info("Implemented interfaces: {:d}", class_def->get_number_of_interfaces());

    logger->info("Number of static fields: {}, number of instance fields: {}, number of direct methods: {}, number of virtual methods: {}",
                     class_data_item->get_number_of_static_fields(),
                     class_data_item->get_number_of_instance_fields(),
                     class_data_item->get_number_of_direct_methods(),
                     class_data_item->get_number_of_virtual_methods());

    auto superclass = class_def->get_superclass_idx();

    if (!superclass)
        continue;
    
    logger->info("[Superclass] Type of object: {}, name: {}", superclass->print_type(), superclass->get_name());
}


```

As I previously said, the **DexParser** object gave us all the information, and in this case from the object we retrieve a vector of **ClassDef** objects, this is a definition of a class that contains different information, it contains an object of type **Class** that will contain the name of the class. We can find the access flag of the class, this is an enum that we can check to print correct string, or just to check for some specific classes in our program. It will contain interfaces in case it implements any. Then it contains an object of type **ClassDataItem**, this object is important as it contains the fields of the class and the methods, we will just print the number for each one. Finally we can access to the information from the _superclass_, this is the parent class of the current one, in _Java_ commonly all the classes derive from **Ljava/lang/Object;**.

No we can go with the methods from the **DEX** file:

```
auto vector_method_ids = dex_parser->get_methods_id_item();

logger->info("[!] MethodIds");

for (auto &method_id : vector_method_ids)
{
    auto prototype = method_id->get_method_prototype();
    auto method_class = method_id->get_method_class();
    auto class_name = std::string("");
    auto name = *method_id->get_method_name();
    
    if (method_class->get_type() == KUNAI::DEX::Type::CLASS)
        class_name = std::dynamic_pointer_cast<KUNAI::DEX::Class>(method_class)->get_name();
    else
        class_name = method_class->get_raw();
        
    logger->info("[+] MethodId:");
    logger->info("Method name: {}, method prototype: {}, class: {}", name, prototype->get_proto_str(), class_name);
}


```

Using the **DexParser** object, we can access to all the methods defined in the **DEX** file, from each method, we can access to the _prototype_ which shows the definition of the method (parameters and return type), the class the method belongs to, the returned object is from the **Type** class that can represent **Fundamental** types, **Array** class or **Class** type (we accessed this in previous code snippet for printing class information), here we apply a check to make a cast in case of necessary to retrieve the name, or just get raw type in other case. Finally together with the method name we print all the information to the terminal.

Finally, what we’ll do is printing all the **Strings** that are defined in the **DEX** file, just to see how classes names, method names, types and so on, are defined as strings in the file.

```
logger->info("[!] Strings");
    
for (size_t n_strings = strings->get_number_of_strings(), i = 0; i < n_strings; i++)
    logger->info("String[{}] = {}", i, *strings->get_string_from_order(i));


```

The code in this case is very simple to understand, so mostly what we do is to retrieve the number of strings defined in the **DEX** file, and finally obtain them from their order, it’s also possible to extract it by its offset in the **DEX**.

Now we’ll see the whole code of the program:

```
#include <iostream>

#include <KUNAI/DEX/dex.hpp>
#include <spdlog/spdlog.h>

int main(int argc, char **argv)
{
    if (argc != 2)
    {
        std::cerr << "[-] USAGE: " << argv[0] << " <dex file>" << std::endl;
        return -1;
    }

    // watch only info and error messages from Kunai
    spdlog::set_level(spdlog::level::info);

    auto logger = KUNAI::LOGGER::logger();

    logger->info("Starting the analysis of {}", argv[1]);

    std::ifstream dex_file;
    dex_file.open(argv[1], std::ios::binary);

    auto fsize = dex_file.tellg();
    dex_file.seekg(0, std::ios::end);
    fsize = dex_file.tellg() - fsize;
    dex_file.seekg(0);

    auto dex_object = KUNAI::DEX::get_unique_dex_object(dex_file, fsize);

    if (!dex_object->get_parsing_correct())
    {
        logger->error("Error analyzing {}, maybe DEX file is not correct...", argv[1]);
        return -1;
    }

    auto dex_parser = dex_object->get_parser();

    logger->info("Dex version number: {}", dex_parser->get_header_version());
    logger->info("Dex version string: {}", dex_parser->get_header_version_str());

    auto dex_header = dex_parser->get_header();

    logger->info("File size: {:d}, Checksum: 0x{:x}, Header size: {:d}", 
        dex_header->get_dex_header().file_size, 
        (std::uint32_t)dex_header->get_dex_header().checksum, 
        dex_header->get_dex_header().header_size);

    logger->info("String ids size: {:d}, String ids offset: 0x{:x}", dex_header->get_dex_header().string_ids_size, dex_header->get_dex_header().string_ids_off);

    auto vector_class_defs = dex_parser->get_classes_def_item();

    logger->info("[!] ClassDefs");

    for (auto &class_def : vector_class_defs)
    {
        auto class_data = class_def->get_class_idx();

        if (!class_data)
            continue;

        logger->info("[+] ClassDef:");

        logger->info("Type of object: {}, name: {}", class_data->print_type(), class_data->get_name());

        auto source_file = class_def->get_source_file_idx();
        if (source_file)
            logger->info("Source file of the class: {}", *source_file);

        // enum that can be checked!
        logger->info("Access Flag: {}", class_def->get_access_flags());

        logger->info("Implemented interfaces: {:d}", class_def->get_number_of_interfaces());

        auto class_data_item = class_def->get_class_data();

        logger->info("Number of static fields: {}, number of instance fields: {}, number of direct methods: {}, number of virtual methods: {}",
                     class_data_item->get_number_of_static_fields(),
                     class_data_item->get_number_of_instance_fields(),
                     class_data_item->get_number_of_direct_methods(),
                     class_data_item->get_number_of_virtual_methods());

        auto superclass = class_def->get_superclass_idx();

        if (!superclass)
            continue;

        logger->info("[Superclass] Type of object: {}, name: {}", superclass->print_type(), superclass->get_name());
    }

    auto vector_method_ids = dex_parser->get_methods_id_item();

    logger->info("[!] MethodIds");

    for (auto &method_id : vector_method_ids)
    {
        auto prototype = method_id->get_method_prototype();
        auto method_class = method_id->get_method_class();
        auto class_name = std::string("");
        auto name = *method_id->get_method_name();
        
        if (method_class->get_type() == KUNAI::DEX::Type::CLASS)
            class_name = std::dynamic_pointer_cast<KUNAI::DEX::Class>(method_class)->get_name();
        else
            class_name = method_class->get_raw();

        logger->info("[+] MethodId:");
        logger->info("Method name: {}, method prototype: {}, class: {}", name, prototype->get_proto_str(), class_name);
    }

    auto strings = dex_parser->get_strings();

    logger->info("[!] Strings");
    
    for (size_t n_strings = strings->get_number_of_strings(), i = 0; i < n_strings; i++)
        logger->info("String[{}] = {}", i, *strings->get_string_from_order(i));

    return 0;
}


```

And also a text code we will use for testing the result binary:

```
import java.util.*;

public class Main {
    public static int modexp(int y, int x[], int w, int n)
    {
        int R = 0, L = 0;
        int k = 0;
        int s = 1;

        while (k < w) {
            if (x[k] == 1)
                R = (s*y) % n;
            else
                R = s;
            s = R*R % n;
            L = R;
            k++;
        }

        return L;
    }

    public static void main(String[] args) throws Exception {
        int y = Integer.parseInt(args[0]);
        int x[] = {1,2,3,4,5,6,7,8,9,10};
        int w = Integer.parseInt(args[1]);
        int n = Integer.parseInt(args[2]);
        
        int ret_value = modexp(y, x, w, n);

        System.out.println("The result is: "+String.valueOf(ret_value));

        return;
    }
}


```

Let’s compile both and try our new **DEX** parser to see the results:

```
$ # compile the binary and link it to Kunai
$ g++ -std=c++17 header-dumper.cpp -o header-dumper -lkunai
$ # Compile the Java part
$ javac --release 8 Main.java && d8 Main.class && mv classes.dex Main.dex
$ ./header-dumper Main.dex 
[2022-07-25 13:20:32.361] [stderr] [info] Starting the analysis of Main.dex
[2022-07-25 13:20:32.362] [stderr] [info] DexParser start parsing dex file with a size of 1584 bytes
[2022-07-25 13:20:32.362] [stderr] [info] Starting DEX headers parsing
[2022-07-25 13:20:32.362] [stderr] [info] DexHeader parsing correct
[2022-07-25 13:20:32.363] [stderr] [info] DexStrings parsing correct
[2022-07-25 13:20:32.364] [stderr] [info] DexTypes parsing correct
[2022-07-25 13:20:32.364] [stderr] [info] DexProtos parsing correct
[2022-07-25 13:20:32.366] [stderr] [info] DexMethods parsing correct
[2022-07-25 13:20:32.367] [stderr] [info] DexClasses parsing correct
[2022-07-25 13:20:32.367] [stderr] [info] Finished DEX headers parsing
[2022-07-25 13:20:32.369] [stderr] [info] Dex version number: 37
[2022-07-25 13:20:32.369] [stderr] [info] Dex version string: DEX_VERSION_37
[2022-07-25 13:20:32.369] [stderr] [info] File size: 1584, Checksum: 0xa698de91, Header size: 112
[2022-07-25 13:20:32.370] [stderr] [info] String ids size: 32, String ids offset: 0x70
[2022-07-25 13:20:32.370] [stderr] [info] [!] ClassDefs
[2022-07-25 13:20:32.370] [stderr] [info] [+] ClassDef:
[2022-07-25 13:20:32.370] [stderr] [info] Type of object: Class, name: LMain;
[2022-07-25 13:20:32.370] [stderr] [info] Source file of the class: Main.java
[2022-07-25 13:20:32.370] [stderr] [info] Access Flag: 1
[2022-07-25 13:20:32.370] [stderr] [info] Implemented interfaces: 0
[2022-07-25 13:20:32.370] [stderr] [info] Number of static fields: 0, number of instance fields: 0, number of direct methods: 3, number of virtual methods: 0
[2022-07-25 13:20:32.370] [stderr] [info] [Superclass] Type of object: Class, name: Ljava/lang/Object;
[2022-07-25 13:20:32.370] [stderr] [info] [!] MethodIds
[2022-07-25 13:20:32.370] [stderr] [info] [+] MethodId:
[2022-07-25 13:20:32.370] [stderr] [info] Method name: <init>, method prototype: ()V, class: LMain;
[2022-07-25 13:20:32.370] [stderr] [info] [+] MethodId:
[2022-07-25 13:20:32.370] [stderr] [info] Method name: main, method prototype: ([Ljava/lang/String;)V, class: LMain;
[2022-07-25 13:20:32.370] [stderr] [info] [+] MethodId:
[2022-07-25 13:20:32.370] [stderr] [info] Method name: modexp, method prototype: (I[III)I, class: LMain;
[2022-07-25 13:20:32.370] [stderr] [info] [+] MethodId:
[2022-07-25 13:20:32.370] [stderr] [info] Method name: println, method prototype: (Ljava/lang/String;)V, class: Ljava/io/PrintStream;
[2022-07-25 13:20:32.370] [stderr] [info] [+] MethodId:
[2022-07-25 13:20:32.370] [stderr] [info] Method name: parseInt, method prototype: (Ljava/lang/String;)I, class: Ljava/lang/Integer;
[2022-07-25 13:20:32.370] [stderr] [info] [+] MethodId:
[2022-07-25 13:20:32.371] [stderr] [info] Method name: <init>, method prototype: ()V, class: Ljava/lang/Object;
[2022-07-25 13:20:32.371] [stderr] [info] [+] MethodId:
[2022-07-25 13:20:32.371] [stderr] [info] Method name: valueOf, method prototype: (I)Ljava/lang/String;, class: Ljava/lang/String;
[2022-07-25 13:20:32.371] [stderr] [info] [+] MethodId:
[2022-07-25 13:20:32.371] [stderr] [info] Method name: <init>, method prototype: ()V, class: Ljava/lang/StringBuilder;
[2022-07-25 13:20:32.371] [stderr] [info] [+] MethodId:
[2022-07-25 13:20:32.371] [stderr] [info] Method name: append, method prototype: (Ljava/lang/String;)Ljava/lang/StringBuilder;, class: Ljava/lang/StringBuilder;
[2022-07-25 13:20:32.371] [stderr] [info] [+] MethodId:
[2022-07-25 13:20:32.371] [stderr] [info] Method name: toString, method prototype: ()Ljava/lang/String;, class: Ljava/lang/StringBuilder;
[2022-07-25 13:20:32.371] [stderr] [info] [!] Strings
[2022-07-25 13:20:32.371] [stderr] [info] String[0] = <init>
[2022-07-25 13:20:32.371] [stderr] [info] String[1] = I
[2022-07-25 13:20:32.371] [stderr] [info] String[2] = IILII
[2022-07-25 13:20:32.371] [stderr] [info] String[3] = IL
[2022-07-25 13:20:32.371] [stderr] [info] String[4] = L
[2022-07-25 13:20:32.371] [stderr] [info] String[5] = LI
[2022-07-25 13:20:32.371] [stderr] [info] String[6] = LL
[2022-07-25 13:20:32.371] [stderr] [info] String[7] = LMain;
[2022-07-25 13:20:32.371] [stderr] [info] String[8] = Ldalvik/annotation/Throws;
[2022-07-25 13:20:32.371] [stderr] [info] String[9] = Ljava/io/PrintStream;
[2022-07-25 13:20:32.371] [stderr] [info] String[10] = Ljava/lang/Exception;
[2022-07-25 13:20:32.371] [stderr] [info] String[11] = Ljava/lang/Integer;
[2022-07-25 13:20:32.372] [stderr] [info] String[12] = Ljava/lang/Object;
[2022-07-25 13:20:32.372] [stderr] [info] String[13] = Ljava/lang/String;
[2022-07-25 13:20:32.372] [stderr] [info] String[14] = Ljava/lang/StringBuilder;
[2022-07-25 13:20:32.372] [stderr] [info] String[15] = Ljava/lang/System;
[2022-07-25 13:20:32.372] [stderr] [info] String[16] = Main.java
[2022-07-25 13:20:32.372] [stderr] [info] String[17] = The result is: 
[2022-07-25 13:20:32.372] [stderr] [info] String[18] = V
[2022-07-25 13:20:32.372] [stderr] [info] String[19] = VL
[2022-07-25 13:20:32.372] [stderr] [info] String[20] = [I
[2022-07-25 13:20:32.372] [stderr] [info] String[21] = [Ljava/lang/String;
[2022-07-25 13:20:32.372] [stderr] [info] String[22] = append
[2022-07-25 13:20:32.372] [stderr] [info] String[23] = main
[2022-07-25 13:20:32.372] [stderr] [info] String[24] = modexp
[2022-07-25 13:20:32.372] [stderr] [info] String[25] = out
[2022-07-25 13:20:32.372] [stderr] [info] String[26] = parseInt
[2022-07-25 13:20:32.372] [stderr] [info] String[27] = println
[2022-07-25 13:20:32.372] [stderr] [info] String[28] = toString
[2022-07-25 13:20:32.372] [stderr] [info] String[29] = value
[2022-07-25 13:20:32.372] [stderr] [info] String[30] = valueOf
[2022-07-25 13:20:32.372] [stderr] [info] String[31] = ~~D8{"backend":"dex","compilation-mode":"debug","has-checksums":false,"min-api":1,"version":"3.3.20-dev+aosp1"}


```

This would be the first post about **Kunai**, so you can get more or less the idea about how the tool is written, and how it works. In next series I will explain how to use the _disassembler_, the _analysis_ part for **DEX**, and also the _Intermediate Representation_ **MjolnIR**.

References
----------

Here I leave some references if you want to dig in the topic, I haven’t really covered much about **DEX** file, but you can find more information in next links:

*   [Kunai Source](https://github.com/Fare9/KUNAI-static-analyzer)
*   [Kunai Documentation](https://fare9.github.io/KUNAI-static-analyzer/)
*   [Dex-format from AOSP documentation](https://source.android.com/devices/tech/dalvik/dex-format)
*   [Dalvik-bytecode format](https://source.android.com/devices/tech/dalvik/dalvik-bytecode)
*   [Dalvik Instructions format](https://source.android.com/devices/tech/dalvik/instruction-formats)
*   [More on Android formats by @rh0main on Lief webpage](https://lief-project.github.io/doc/latest/tutorials/10_android_formats.html)