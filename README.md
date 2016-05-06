# args

A simple, small, flexible, single-header C++11 argument parsing library, in
about 1.3K lines of code.

This is designed to somewhat replicate the behavior of Python's argparse, but
in C++, with static type checking, and hopefully a lot faster.

UTF-8 support is limited at best.  No normalization is performed, so non-ascii
characters are very best kept out of flags, and combined glyphs are probably
going to mess up help output if you use it.  Most UTF-8 necessary for
internationalization should work for most cases, though heavily combinatory UTF
alphabets may wreak havoc.

This program is MIT-licensed, so you can use the header as-is with no
restrictions. I'd appreciate attribution in a README, Man page, or something if
you are feeling generous, but all that's required is that you don't remove the
license and my name from the header of the args.hxx file in source
redistributions (ie. don't pretend that you wrote it).  I do welcome additions
and updates wherever you feel like contributing code.

The API documentation can be found at https://taywee.github.io/args

The code can be downloaded at https://github.com/Taywee/args

There are also somewhat extensive examples below.

# What does it do?

It:

* lets you handle flags, flag+arguments, and positional arguments simply and
    elegently, with the full help of static typechecking.
* allows you to use your own types in a pretty simple way.
* lets you use count flags, and lists of all argument-accepting types.
* Allows full validation of groups of required arguments, though output isn't
    pretty when something fails group validation.  User validation functions are
    accepted.  Groups are fully nestable.
* Generates pretty help for you, with some good tweakable parameters.
* Lets you customize all prefixes and most separators, allowing you to create
    an infinite number of different argument syntaxes
* Lets you parse, by default, any type that has a stream extractor operator for
  it.  If this doesn't work for your uses, you can supply a function and parse
  the string yourself if you like.

# What does it not do?

There are tons of things this library does not do!

## It does not yet:

* Let you decide not to allow separate-argument argument flags or joined ones
    (like disallowing `--foo bar`, requiring `--foo=bar`, or the inverse, or the
    same for short options).

## It will not ever:

* Allow you to create subparsers like argparse in a master parser (you can do
    this yourself with iterators and multiple parsers)
* Allow one argument flag to take a specific number of arguments
    (like `--foo first second`).  You can instead split that with a flag list
    (`--foo first --foo second`) or a custom type extraction
    (`--foo first,second`)
* Allow you to intermix multiple different prefix types (eg. `++foo` and
    `--foo` in the same parser), though shortopt and longopt prefixes can be
    different.
* Allow you to have argument flags only optionally accept arguments
* Allow you to use a positional argument list before any other positional
    arguments (the last argument list will slurp all subsequent positional
    arguments).  The logic for allowing this would be a lot more code than I'd
    like, and would make static checking much more difficult, requiring us to
    sort std::string arguments and pair them to positional arguments before
    assigning them, rather than what we currently do, which is assiging them as
    we go for better simplicity and speed.

# How do I use it?

```shell
sudo make install
```

Or, to install it somewhere special (default is `/usr/local`):

```shell
sudo make install DESTDIR=/opt/mydir
```

You can also copy the file into your source tree, if you want to be absolutely
sure you keep a stable API between projects.

## I also want man pages.

```shell
sudo make installman
```

This requires Doxygen

# How fast is it?

This should not really be a question you ask when you are looking for an
argument-parsing library, but I did run a simple benchmark against args, TCLAP,
and boost::program_options, which parses the command line `-i 7 -c a 2.7 --char
b 8.4 -c c 8.8 --char d` with a parser that parses -i as an int, -c as a list
of chars, and the positional parameters as a list of doubles (the command line
was originally much more complex, but TCLAP's limitations made me trim it down
so I could use a common command line across all libraries.  I also have to copy
in the arguments list with every run, because TCLAP permutes its argument list
as it runs (and comparison would have been unfair without comparing all about
equally), but that surprisingly didn't affect much.  Also tested is pulling the
arguments out, but that was fast compared to parsing, as would be expected.

### The run:

```shell
% g++ -obench bench.cxx -O2 -std=c++11 -lboost_program_options
% ./bench
args seconds to run: 0.895472
tclap seconds to run: 1.45001
boost::program_options seconds to run: 1.98972
%
```

### The benchmark:

```cpp
#undef NDEBUG
#include <iostream>
#include <chrono>
#include <cassert>
#include "args.hxx"
#include <tclap/CmdLine.h>
#include <boost/program_options.hpp>
namespace po = boost::program_options;
using namespace std::chrono;
inline bool doubleequals(const double a, const double b)
{
    static const double delta = 0.0001;
    const double diff = a - b;
    return diff < delta && diff > -delta;
}
int main()
{
    const std::vector<std::string> carguments({"-i", "7", "-c", "a", "2.7", "--char", "b", "8.4", "-c", "c", "8.8", "--char", "d"});
    const std::vector<std::string> pcarguments({"progname", "-i", "7", "-c", "a", "2.7", "--char", "b", "8.4", "-c", "c", "8.8", "--char", "d"});
    // args
    {
        high_resolution_clock::time_point start = high_resolution_clock::now();
        for (unsigned int x = 0; x < 100000; ++x)
        {
            std::vector<std::string> arguments(carguments);
            args::ArgumentParser parser("This is a test program.", "This goes after the options.");
            args::ArgFlag<int> integer(parser, "integer", "The integer flag", args::Matcher({'i'}, {"int"}));
            args::ArgFlagList<char> characters(parser, "characters", "The character flag", args::Matcher({'c'}, {"char"}));
            args::PosArgList<double> numbers(parser, "numbers", "The numbers position list");
            parser.ParseArgs(arguments);
            const int i = integer.value;
            const std::vector<char> c(characters.values);
            const std::vector<double> n(numbers.values);
            assert(i == 7);
            assert(c[0] == 'a');
            assert(c[1] == 'b');
            assert(c[2] == 'c');
            assert(c[3] == 'd');
            assert(doubleequals(n[0], 2.7));
            assert(doubleequals(n[1], 8.4));
            assert(doubleequals(n[2], 8.8));
        }
        high_resolution_clock::duration runtime = high_resolution_clock::now() - start;
        std::cout << "args seconds to run: " << duration_cast<duration<double>>(runtime).count() << std::endl;
    }
    // tclap
    {
        high_resolution_clock::time_point start = high_resolution_clock::now();
        for (unsigned int x = 0; x < 100000; ++x)
        {
            std::vector<std::string> arguments(pcarguments);
            TCLAP::CmdLine cmd("Command description message", ' ', "0.9");
            TCLAP::ValueArg<int> integer("i", "int", "The integer flag", false, 0, "integer", cmd);
            TCLAP::MultiArg<char> characters("c", "char", "The character flag", false, "characters", cmd);
            TCLAP::UnlabeledMultiArg<double> numbers("numbers", "The numbers position list", false, "foo", cmd, false);
            cmd.parse(arguments);
            const int i = integer.getValue();
            const std::vector<char> c(characters.getValue());
            const std::vector<double> n(numbers.getValue());
            assert(i == 7);
            assert(c[0] == 'a');
            assert(c[1] == 'b');
            assert(c[2] == 'c');
            assert(c[3] == 'd');
            assert(doubleequals(n[0], 2.7));
            assert(doubleequals(n[1], 8.4));
            assert(doubleequals(n[2], 8.8));
        }
        high_resolution_clock::duration runtime = high_resolution_clock::now() - start;
        std::cout << "tclap seconds to run: " << duration_cast<duration<double>>(runtime).count() << std::endl;
    }
    // boost::program_options
    {
        high_resolution_clock::time_point start = high_resolution_clock::now();
        for (unsigned int x = 0; x < 100000; ++x)
        {
            std::vector<std::string> arguments(carguments);
            po::options_description desc("This is a test program.");
            desc.add_options()
                ("int,i", po::value<int>(), "The integer flag")
                ("char,c", po::value<std::vector<char>>(), "The character flag")
                ("numbers", po::value<std::vector<double>>(), "The numbers flag");
            po::positional_options_description p;
            p.add("numbers", -1);
            po::variables_map vm;
            po::store(po::command_line_parser(carguments).options(desc).positional(p).run(), vm);
            const int i = vm["int"].as<int>();
            const std::vector<char> c(vm["char"].as<std::vector<char>>());
            const std::vector<double> n(vm["numbers"].as<std::vector<double>>());
            assert(i == 7);
            assert(c[0] == 'a');
            assert(c[1] == 'b');
            assert(c[2] == 'c');
            assert(c[3] == 'd');
            assert(doubleequals(n[0], 2.7));
            assert(doubleequals(n[1], 8.4));
            assert(doubleequals(n[2], 8.8));
        }
        high_resolution_clock::duration runtime = high_resolution_clock::now() - start;
        std::cout << "boost::program_options seconds to run: " << duration_cast<duration<double>>(runtime).count() << std::endl;
    }
    return 0;
}
```

So, on top of being more flexible, smaller, and easier to read, it is faster
than the most common alternatives.

# Is it developed with regression tests?

Yes.  tests.cxx in the git repository has a set of standard tests (which are
still relatively small in number, but I would welcome some expansion here), and
thanks to GitLab's CI, these tests run with every single push:

```shell
% make runtests
g++ test.cxx -o test.o -I. -std=c++11 -O2 -c -MMD
g++ -o test test.o -std=c++11 -O2
./test
===============================================================================
All tests passed (74 assertions in 15 test cases)

%
```

The testing library used is [Catch](https://github.com/philsquared/Catch).

# Examples

All the code examples here will be complete code examples, with some output.

## Simple example:

```cpp
#include <iostream>
#include <args.hxx>
int main(int argc, char **argv)
{
    args::ArgumentParser parser("This is a test program.", "This goes after the options.");
    args::HelpFlag help(parser, "help", "Display this help menu", args::Matcher({'h'}, {"help"}));
    try
    {
        parser.ParseCLI(argc, argv);
    }
    catch (args::Help)
    {
        std::cout << parser;
        return 0;
    }
    catch (args::ParseError e)
    {
        std::cerr << e.what() << std::endl;
        std::cerr << parser;
        return 1;
    }
    return 0;
}
```

```shell
 % ./test
 % ./test -h
  ./test {OPTIONS} 

    This is a test program. 

  OPTIONS:

      -h, --help         Display this help menu 

    This goes after the options. 
 % 
```

## Boolean flags, special group types, different matcher construction:

```cpp
#include <iostream>
#include <args.hxx>
int main(int argc, char **argv)
{
    args::ArgumentParser parser("This is a test program.", "This goes after the options.");
    args::Group group(parser, "This group is all exclusive:", args::Group::Validators::Xor);
    args::Flag foo(group, "foo", "The foo flag", args::Matcher({'f'}, {"foo"}));
    args::Flag bar(group, "bar", "The bar flag", args::Matcher({'b'}));
    args::Flag baz(group, "baz", "The baz flag", args::Matcher({"baz"}));
    try
    {
        parser.ParseCLI(argc, argv);
    }
    catch (args::Help)
    {
        std::cout << parser;
        return 0;
    }
    catch (args::ParseError e)
    {
        std::cerr << e.what() << std::endl;
        std::cerr << parser;
        return 1;
    }
    catch (args::ValidationError e)
    {
        std::cerr << e.what() << std::endl;
        std::cerr << parser;
        return 1;
    }
    if (foo) { std::cout << "foo" << std::endl; }
    if (bar) { std::cout << "bar" << std::endl; }
    if (baz) { std::cout << "baz" << std::endl; }
    return 0;
}
```

```shell
 % ./test   
Group validation failed somewhere!
  ./test {OPTIONS} 

    This is a test program. 

  OPTIONS:

                         This group is all exclusive:
        -f, --foo          The foo flag 
        -b                 The bar flag 
        --baz              The baz flag 

    This goes after the options. 
 % ./test -f
foo
 % ./test --foo
foo
 % ./test --foo -f
foo
 % ./test -b      
bar
 % ./test --baz
baz
 % ./test --baz -f
Group validation failed somewhere!
  ./test {OPTIONS} 

    This is a test program. 
...
 % ./test --baz -fb
Group validation failed somewhere!
  ./test {OPTIONS} 
...
 % 
```

## Argument flags, Positional arguments, lists

```cpp
#include <iostream>
#include <args.hxx>
int main(int argc, char **argv)
{
    args::ArgumentParser parser("This is a test program.", "This goes after the options.");
    args::HelpFlag help(parser, "help", "Display this help menu", args::Matcher({'h'}, {"help"}));
    args::ArgFlag<int> integer(parser, "integer", "The integer flag", args::Matcher({'i'}));
    args::ArgFlagList<char> characters(parser, "characters", "The character flag", args::Matcher({'c'}));
    args::PosArg<std::string> foo(parser, "foo", "The foo position");
    args::PosArgList<double> numbers(parser, "numbers", "The numbers position list");
    try
    {
        parser.ParseCLI(argc, argv);
    }
    catch (args::Help)
    {
        std::cout << parser;
        return 0;
    }
    catch (args::ParseError e)
    {
        std::cerr << e.what() << std::endl;
        std::cerr << parser;
        return 1;
    }
    catch (args::ValidationError e)
    {
        std::cerr << e.what() << std::endl;
        std::cerr << parser;
        return 1;
    }
    if (integer) { std::cout << "i: " << integer.value << std::endl; }
    if (characters) { for (const auto ch: characters.values) { std::cout << "c: " << ch << std::endl; } }
    if (foo) { std::cout << "f: " << foo.value << std::endl; }
    if (numbers) { for (const auto nm: numbers.values) { std::cout << "n: " << nm << std::endl; } }
    return 0;
}
```

```shell
% ./test -h
  ./test {OPTIONS} [foo] [numbers...] 

    This is a test program. 

  OPTIONS:

      -h, --help         Display this help menu 
      -i integer         The integer flag 
      -c characters      The character flag 
      foo                The foo position 
      numbers            The numbers position list 
      "--" can be used to terminate flag options and force all following
      arguments to be treated as positional options 

    This goes after the options. 
 % ./test -i 5
i: 5
 % ./test -i 5.2
Argument 'integer' received invalid value type '5.2'
  ./test {OPTIONS} [foo] [numbers...] 
 % ./test -c 1 -c 2 -c 3
c: 1
c: 2
c: 3
 % 
 % ./test 1 2 3 4 5 6 7 8 9
f: 1
n: 2
n: 3
n: 4
n: 5
n: 6
n: 7
n: 8
n: 9
 % ./test 1 2 3 4 5 6 7 8 9 a
Argument 'numbers' received invalid value type 'a'
  ./test {OPTIONS} [foo] [numbers...] 

    This is a test program. 
...
```

# Custom type parsers (here we use std::tuple)

```cpp
#include <iostream>
#include <tuple>
#include <args.hxx>
std::istream& operator>>(std::istream& is, std::tuple<int, int>& ints)
{
    is >> std::get<0>(ints);
    is.get();
    is >> std::get<1>(ints);
    return is;
}

void DoublesReader(const std::string &name, const std::string &value, std::tuple<double, double> &destination)
{
    size_t commapos = 0;
    std::get<0>(destination) = std::stod(value, &commapos);
    std::get<1>(destination) = std::stod(std::string(value, commapos + 1));
}

int main(int argc, char **argv)
{
    args::ArgumentParser parser("This is a test program.");
    args::PosArg<std::tuple<int, int>> ints(parser, "INTS", "This takes a pair of integers.");
    args::PosArg<std::tuple<double, double>, DoublesReader> doubles(parser, "DOUBLES", "This takes a pair of doubles.");
    try
    {
        parser.ParseCLI(argc, argv);
    }
    catch (args::Help)
    {
        std::cout << parser;
        return 0;
    }
    catch (args::ParseError e)
    {
        std::cerr << e.what() << std::endl;
        std::cerr << parser;
        return 1;
    }
    if (ints)
    {
        std::cout << "ints found: " << std::get<0>(ints.value) << " and " << std::get<1>(ints.value) << std::endl;
    }
    if (doubles)
    {
        std::cout << "doubles found: " << std::get<0>(doubles.value) << " and " << std::get<1>(doubles.value) << std::endl;
    }
    return 0;
}
```

```shell
 % ./test -h
Argument could not be matched: 'h'
  ./test [INTS] [DOUBLES] 

    This is a test program. 

  OPTIONS:

      INTS               This takes a pair of integers. 
      DOUBLES            This takes a pair of doubles. 

 % ./test 5
ints found: 5 and 0
 % ./test 5,8
ints found: 5 and 8
 % ./test 5,8 2.4,8
ints found: 5 and 8
doubles found: 2.4 and 8
 % ./test 5,8 2.4, 
terminate called after throwing an instance of 'std::invalid_argument'
  what():  stod
zsh: abort      ./test 5,8 2.4,
 % ./test 5,8 2.4 
terminate called after throwing an instance of 'std::out_of_range'
  what():  basic_string::basic_string: __pos (which is 4) > this->size() (which is 3)
zsh: abort      ./test 5,8 2.4
 % ./test 5,8 2.4-7
ints found: 5 and 8
doubles found: 2.4 and 7
 % ./test 5,8 2.4,-7
ints found: 5 and 8
doubles found: 2.4 and -7
```

As you can see, with your own types, validation can get a little weird.  Make
sure to check and throw a parsing error (or whatever error you want to catch)
if you can't fully deduce your type.  The built-in validator will only throw if
there are unextracted characters left in the stream.

## Long descriptions and proper wrapping and listing

```cpp
#include <iostream>
#include <args.hxx>
int main(int argc, char **argv)
{
    args::ArgumentParser parser("This is a test program with a really long description that is probably going to have to be wrapped across multiple different lines.  This is a test to see how the line wrapping works", "This goes after the options.  This epilog is also long enough that it will have to be properly wrapped to display correctly on the screen");
    args::HelpFlag help(parser, "HELP", "Show this help menu.", args::Matcher({'h'}, {"help"}));
    args::ArgFlag<std::string> foo(parser, "FOO", "The foo flag.", args::Matcher({'a', 'b', 'c'}, {"a", "b", "c", "the-foo-flag"}));
    args::ArgFlag<std::string> bar(parser, "BAR", "The bar flag.  This one has a lot of options, and will need wrapping in the description, along with its long flag list.", args::Matcher({'d', 'e', 'f'}, {"d", "e", "f"}));
    args::ArgFlag<std::string> baz(parser, "FOO", "The baz flag.  This one has a lot of options, and will need wrapping in the description, even with its short flag list.", args::Matcher({"baz"}));
    args::PosArg<std::string> pos1(parser, "POS1", "The pos1 argument.");
    args::PosArgList<std::string> poslist1(parser, "POSLIST1", "The poslist1 argument.");
    args::PosArg<std::string> pos2(parser, "POS2", "The pos2 argument.");
    args::PosArgList<std::string> poslist2(parser, "POSLIST2", "The poslist2 argument.");
    args::PosArg<std::string> pos3(parser, "POS3", "The pos3 argument.");
    args::PosArgList<std::string> poslist3(parser, "POSLIST3", "The poslist3 argument.");
    args::PosArg<std::string> pos4(parser, "POS4", "The pos4 argument.");
    args::PosArgList<std::string> poslist4(parser, "POSLIST4", "The poslist4 argument.");
    args::PosArg<std::string> pos5(parser, "POS5", "The pos5 argument.");
    args::PosArgList<std::string> poslist5(parser, "POSLIST5", "The poslist5 argument.");
    args::PosArg<std::string> pos6(parser, "POS6", "The pos6 argument.");
    args::PosArgList<std::string> poslist6(parser, "POSLIST6", "The poslist6 argument.");
    args::PosArg<std::string> pos7(parser, "POS7", "The pos7 argument.");
    args::PosArgList<std::string> poslist7(parser, "POSLIST7", "The poslist7 argument.");
    args::PosArg<std::string> pos8(parser, "POS8", "The pos8 argument.");
    args::PosArgList<std::string> poslist8(parser, "POSLIST8", "The poslist8 argument.");
    args::PosArg<std::string> pos9(parser, "POS9", "The pos9 argument.");
    args::PosArgList<std::string> poslist9(parser, "POSLIST9", "The poslist9 argument.");
    try
    {
        parser.ParseCLI(argc, argv);
    }
    catch (args::Help)
    {
        std::cout << parser;
        return 0;
    }
    catch (args::ParseError e)
    {
        std::cerr << e.what() << std::endl;
        std::cerr << parser;
        return 1;
    }
    catch (args::ValidationError e)
    {
        std::cerr << e.what() << std::endl;
        std::cerr << parser;
        return 1;
    }
    return 0;
}
```

```shell
 % ./test -h
  ./test {OPTIONS} [POS1] [POSLIST1...] [POS2] [POSLIST2...] [POS3]
      [POSLIST3...] [POS4] [POSLIST4...] [POS5] [POSLIST5...] [POS6]
      [POSLIST6...] [POS7] [POSLIST7...] [POS8] [POSLIST8...] [POS9]
      [POSLIST9...] 

    This is a test program with a really long description that is probably going
    to have to be wrapped across multiple different lines. This is a test to see
    how the line wrapping works 

  OPTIONS:

      -h, --help         Show this help menu. 
      -a FOO, -b FOO, -c FOO, --a FOO, --b FOO, --c FOO, --the-foo-flag FOO
                         The foo flag. 
      -d BAR, -e BAR, -f BAR, --d BAR, --e BAR, --f BAR
                         The bar flag. This one has a lot of options, and will
                         need wrapping in the description, along with its long
                         flag list. 
      --baz FOO          The baz flag. This one has a lot of options, and will
                         need wrapping in the description, even with its short
                         flag list. 
      POS1               The pos1 argument. 
      POSLIST1           The poslist1 argument. 
      POS2               The pos2 argument. 
      POSLIST2           The poslist2 argument. 
      POS3               The pos3 argument. 
      POSLIST3           The poslist3 argument. 
      POS4               The pos4 argument. 
      POSLIST4           The poslist4 argument. 
      POS5               The pos5 argument. 
      POSLIST5           The poslist5 argument. 
      POS6               The pos6 argument. 
      POSLIST6           The poslist6 argument. 
      POS7               The pos7 argument. 
      POSLIST7           The poslist7 argument. 
      POS8               The pos8 argument. 
      POSLIST8           The poslist8 argument. 
      POS9               The pos9 argument. 
      POSLIST9           The poslist9 argument. 
      "--" can be used to terminate flag options and force all following
      arguments to be treated as positional options 

    This goes after the options. This epilog is also long enough that it will
    have to be properly wrapped to display correctly on the screen 
 %
```

## Customizing parser prefixes

### dd-style

```cpp
#include <iostream>
#include <args.hxx>
int main(int argc, char **argv)
{
    args::ArgumentParser parser("This command likes to break your disks");
    parser.LongPrefix("");
    parser.LongSeparator("=");
    args::HelpFlag help(parser, "HELP", "Show this help menu.", args::Matcher({"help"}));
    args::ArgFlag<long> bs(parser, "BYTES", "Block size", args::Matcher({"bs"}), 512);
    args::ArgFlag<long> skip(parser, "BYTES", "Bytes to skip", args::Matcher({"skip"}), 0);
    args::ArgFlag<std::string> input(parser, "BLOCK SIZE", "Block size", args::Matcher({"if"}));
    args::ArgFlag<std::string> output(parser, "BLOCK SIZE", "Block size", args::Matcher({"of"}));
    try
    {
        parser.ParseCLI(argc, argv);
    }
    catch (args::Help)
    {
        std::cout << parser;
        return 0;
    }
    catch (args::ParseError e)
    {
        std::cerr << e.what() << std::endl;
        std::cerr << parser;
        return 1;
    }
    catch (args::ValidationError e)
    {
        std::cerr << e.what() << std::endl;
        std::cerr << parser;
        return 1;
    }
    std::cout << "bs = " << bs.value << std::endl;
    std::cout << "skip = " << skip.value << std::endl;
    if (input) { std::cout << "if = " << input.value << std::endl; }
    if (output) { std::cout << "of = " << output.value << std::endl; }
    return 0;
}
```

```shell
 % ./test help
  ./test {OPTIONS} 

    This command likes to break your disks 

  OPTIONS:

      help               Show this help menu. 
      bs BYTES           Block size 
      skip BYTES         Bytes to skip 
      if BLOCK_SIZE      Block size 
      of BLOCK_SIZE      Block size 

 % ./test bs=1024 skip=7 if=/tmp/input
bs = 1024
skip = 7
if = /tmp/input
```

### Windows style

The code is the same as above, but the two lines are replaced out:

```cpp
parser.LongPrefix("/");
parser.LongSeparator(":");
```

```shell
 % ./test /help     
  ./test {OPTIONS} 

    This command likes to break your disks 

  OPTIONS:

      /help              Show this help menu. 
      /bs BYTES          Block size 
      /skip BYTES        Bytes to skip 
      /if BLOCK_SIZE     Block size 
      /of BLOCK_SIZE     Block size 

 % ./test /bs:72 /skip:87 /if:/tmp/test.txt
bs = 72
skip = 87
if = /tmp/test.txt
 % 
```

## Group nesting help menu text

```cpp
#include <iostream>
#include <args.hxx>
int main(int argc, char **argv)
{
    args::ArgumentParser parser("This is a test program.", "This goes after the options.");
    args::Group xorgroup(parser, "this group provides xor validation:", args::Group::Validators::Xor);
    args::Flag a(xorgroup, "a", "test flag", args::Matcher({'a'}));
    args::Flag b(xorgroup, "b", "test flag", args::Matcher({'b'}));
    args::Flag c(xorgroup, "c", "test flag", args::Matcher({'c'}));
    args::Group nxor(xorgroup, "this group provides all-or-none (nxor) validation:", args::Group::Validators::AllOrNone);
    args::Flag d(nxor, "d", "test flag", args::Matcher({'d'}));
    args::Flag e(nxor, "e", "test flag", args::Matcher({'e'}));
    args::Flag f(nxor, "f", "test flag", args::Matcher({'f'}));
    args::Group nxor2(nxor, "this group provides all-or-none (nxor2) validation:", args::Group::Validators::AllOrNone);
    args::Flag i(nxor2, "i", "test flag", args::Matcher({'i'}));
    args::Flag j(nxor2, "j", "test flag", args::Matcher({'j'}));
    args::Flag k(nxor2, "k", "test flag", args::Matcher({'k'}));
    args::Group nxor3(nxor, "this group provides all-or-none (nxor3) validation:", args::Group::Validators::AllOrNone);
    args::Flag l(nxor3, "l", "test flag", args::Matcher({'l'}));
    args::Flag m(nxor3, "m", "test flag", args::Matcher({'m'}));
    args::Flag n(nxor3, "n", "test flag", args::Matcher({'n'}));
    args::Group atleastone(xorgroup, "this group provides at-least-one validation:", args::Group::Validators::AtLeastOne);
    args::Flag g(atleastone, "g", "test flag", args::Matcher({'g'}));
    args::Flag o(atleastone, "o", "test flag", args::Matcher({'o'}));
    args::HelpFlag help(parser, "help", "Show this help menu", args::Matcher({'h'}, {"help"}));
    try
    {
        parser.ParseCLI(argc, argv);
    }
    catch (args::Help)
    {
        std::cout << parser;
        return 0;
    }
    catch (args::ParseError e)
    {
        std::cerr << e.what() << std::endl;
        parser.Help(std::cerr);
        return 1;
    }
    catch (args::ValidationError e)
    {
        std::cerr << e.what() << std::endl;
        parser.Help(std::cerr);
        return 1;
    }
    return 0;
}
```

```shell
 % /tmp/test -h
  /tmp/test {OPTIONS} 

    This is a test program. 

  OPTIONS:

                         this group provides xor validation: 
        -a                 test flag 
        -b                 test flag 
        -c                 test flag 
                           this group provides all-or-none (nxor) validation: 
          -d                 test flag 
          -e                 test flag 
          -f                 test flag 
                             this group provides all-or-none (nxor2) validation:
            -i                 test flag 
            -j                 test flag 
            -k                 test flag 
                             this group provides all-or-none (nxor3) validation:
            -l                 test flag 
            -m                 test flag 
            -n                 test flag 
                           this group provides at-least-one validation: 
          -g                 test flag 
          -o                 test flag 
      -h, --help         Show this help menu 

    This goes after the options. 
 %                                                                                
```
