---
layout: post
title:  "Achieving generic bliss with reflection in modern C++"
date:   2021-07-19 17:04:01
categories: post
---

Reflection is often presented as a feature that makes software harder to understand.
In this article, I will introduce ways to approximate some level of static reflection in pure C++, thanks to C++17 and C++20 features, show how that tool can considerably simplify a class of programs and libraries, and more generally enable ontologies to be specified and implemented in code.
<!--more-->

# What we aim to solve

Imagine that you are writing a neat algorithm. You've worked on it for a few years ; it produces great results and is now ready to be shared to the world. 
Let's say that this algorithm is a noise generator.

You'd like that noise generator to easily work in a breadth of environments: in 2D bitmap manipulation programs (Krita, GIMP, ...), in audio and multimedia software (PureData, Audacity, [ossia score](https://ossia.io)...), 3D voxel editors, etc.

Your algorithm's implementation more-or-less looks like this:

```C++
// [[pre: alpha >= -1. && alpha <= 1. ]]
// [[pre: 11 <= beta && beta <= 247 ]]
// [[post ret: ret >= 0. && ret <= 1.]]
auto noise(
    float input, 
    float alpha, 
    int beta)
{
  // set of complex operations involving the inputs to the algorithm
  return std::fmod(std::pow(input, alpha), float(beta)) / beta;
}
```

You are proud of your neat results, prepare conference papers, etc... but ! Now is the time to implement your noise algorithm in a set of software in order to have it used widely and become the next industry standard in procedural noise generation.

If you are used to working in C#, Java, Python, or any other language more recent than 1983, the solution may at this point seem trivial. Sadly, in C++, this has been unordinately hard to implement until now, especially when one aims for as close as possible to a zero-runtime cost-abstraction. 

On the other hand, if you implement your algorithm in C#, Java, or Python, having it useable from any other runtime environment is a massive challenge, as two VMs, often with their own garbage collection mechanism, etc... now have to cooperate. Thus, for something really universal, a language than can compile to native binaries, with minimal dependencies, is the easiest way to get a large reach. In particular, most media host environments are written in a native language and expect plug-ins conforming to operating system DLLs. There aren't that many suitable candidates with a high enough capacity for abstraction: C++, Rust, D without GC. Since most of the media host provide C or C++ APIs, C++ is the natural, minimal-friction choice. 

This blogpost is a plea to C++ implementors, to show how much better and easier true reflection as available in other languages, in particular with attribute reflection and user-defined attributes, would make one's life, and what kind of abstracting power "reflective programming" holds over existing generic programming techniques in C++: macro-based metaprogramming, template-based metaprogramming (with e.g. CRTP being commonly used for that).

### The problem domain

The software in which we want to embed our algorithm should be able to display UI widgets adapted for the control of `alpha` and `beta`, whose bounds you have so painstakingly and thoroughly defined. Likewise, the UI widgets should adapt to the type of the parameter ; a spinbox may make more sense for `beta`, and a slider, knob, or any kind of continuous control for `alpha`.

Maybe you'd also like to serve your algorithm over the network, or through an IPC protocol like D-Bus. Again, you'd have to specify the data format being used.

If for instance you were using the OSC protocol, to make your algorithm controllable over the network, messages may look like:

```
/noise/input ,f 0.123
/noise/alpha ,f 13.5
/noise/beta ,i 17
```

Maybe you'd also like to serialize your algorithm's inputs, in order to have a preset system, or just to exchange with another runtime system expecting a serialized version of your data. In JSON ? YAML ? Binary ? Network-byte-order binary ? GLSL `std140` ? So many possibilities !

### Hell on earth 

For *every* protocol, host environment, plug-in system, etc etc. that you want to provide your algorithm to, you will have to write some amount of binding code, also often called *glue code*.

How does that binding code may look, you ask ? 

Let's look at some examples from around the world:

* [Making a fade algorithm in PureData](https://github.com/pure-data/externals-howto/blob/master/example4/xfade%7E.c) : a class is constructed at run-time, with custom `t_object`, `t_float`, `t_inlet` etc... types, some of which requiring calls to various runtime allocating functions. Lots of not-very-safe-looking casts (but it's C, there's not a lot of choice).
* [Noise generator for Max/MSP's Jitter, using OpenCV](https://github.com/Cycling74/cv.jit/blob/master/source/projects/cv.jit.noise/cv.jit.noise.cpp). Same as PureData, with macros sprinkled on top. Wanna get a floating point value input by the user ? Lo and behold

```C++
void cv_jit_noise_set_stddev(t_cv_jit_noise *x, t_symbol *s, short argc, t_atom *argv)
{
	if (x && argc > 0) {
		for (short i = 0; i < x->m_dims; i++) {
			short j = i < argc ? i : argc - 1;
			x->m_stddev[i] = atom_getfloat(argv + j);
            ...
        }
    }
}
```

What happens if `argv + j` isn't a float but a string ? Let's leave that for future generations to discover !

* [Audio filter suitable for use as a VST](https://github.com/SpotlightKid/faustfilters/blob/master/plugins/oberheim/Oberheim.cpp). Notice how the parameters to the algorithms are handled in `switch/case 0,1,2...` ; thankfully this is all generated code from the Faust programming language. What happens if at some point a parameter is removed ? Better have good unit tests to catch all the implicit uses of each parameter...
* [OpenFX image filter](https://openfx.readthedocs.io/en/master/Guide/ofxExample3_Gain.html#gainexample): here's how one says that the algorithm has a bounded input widget (e.g. a slider going from 0 to 10 with a default value of 1):

```C++
gPropertySuite->propSetString(paramProps, kOfxParamPropDoubleType, 0, kOfxParamDoubleTypeScale);
gPropertySuite->propSetDouble(paramProps, kOfxParamPropDefault, 0, 1.0);
gPropertySuite->propSetDouble(paramProps, kOfxParamPropDisplayMin, 0, 0.0);
gPropertySuite->propSetDouble(paramProps, kOfxParamPropDisplayMax, 0, 10.0);
```

Hopefully you don't forget all the incantations's updates when you decide that this control would indeed be better as an integer !
* Things like [iPlug](https://github.com/iPlug2/iPlug2/blob/master/Examples/IPlugEffect/IPlugEffect.cpp#L8) are a bit more sane, but we still have to triplicate our parameter creation / access: in an enum in the hpp, in the constructor and finally in `ProcessBlock` where we get the actual value. This is still a whole lot of work versus **JUST ACCESSING A FLOAT IN A STRUCT !!11!1!!**
* A Krita [plug-in for noise generation](https://github.com/KDE/krita/blob/master/plugins/generators/simplexnoise/simplexnoisegenerator.cpp#L62) -- here Qt's QObject run-time property system is used to declare and use the algorithm controls. That also means inheriting from Qt's QObject, which has a non-negligible memory cost.  
* Wanna receive messages through OSC ? [Make the exceptions rain !](https://github.com/RossBencina/oscpack/blob/master/examples/SimpleReceive.cpp).
* Wanna expose your algorithm to another language, such as Python ? [Get ready for some py::<>'y boilerplate](https://pybind11.readthedocs.io/en/stable/advanced/classes.html). 

As such, one can see that:
- There is no current generic way for writing an audio processor in PureData, and have it work in, say, Audacity, Ardour or LMMS as a VST plug-in, expose it through the network... Writing a PureData external ties you to PureData, and so does writing a Krita plug-in.
It's the well-known ["quadratic glue"](https://www.oreilly.com/radar/thinking-about-glue/) problem: there are N algorithms and M "host systems", thus NxM glue code to write. 

- All the approaches are riddled with unsafety, since the run-time environments force the inputs & outputs to the algorithm to be declared in a dynamic way ; thus, if you make an error in your call sequence, you rely on the runtime system you are using to notice this error and notify you (e.g; if you are lucky you'll get an error message on stdout ; but most likely a crash).

- All the approaches require duplicating the actual parameters of your algorithm, e;g. our `alpha`, `beta`, once as actual C++ variables, once as facades to the runtime object system you are interacting with. 

Of course, the above list is not an indictment on the code quality of those various projects: they simply all do as well as they can considering the limitations of the language.

We will show how reflection allows to improve on that, and in particular get down to N+M pieces of code to write instead of NxM.

### Problem statement
Basically: there's a ton of environments which define ad-hoc protocols or object systems. Can we find a way to make a C++ definition which:

- Does not depend on any pre-existing code: doesn't inherit from a class, doesn't call arbitrary run-time functions, etc. The *definition* of the algorithm shall be writable without having to include *anything*, even standard headers (and discounting of course whatever third-party library is required for the algorithm itself).

- Does not use anything other than structures of trivial, standard-layout types. No tuples, no templates, no magic, just `structs` containing `float`, `int` and not much more. This is because we want to be able to give the *simplest possible expression* of a problem. C++ is often sold as a language which aims to leave no room for a lower-level language. The technique in this post is about leaving no room for a simpler implementation of an algorithm, while maintaining the ability to control its inputs and outputs. Ideally, that would lead to a collection of such algorithms not depending on any framework, except optional concept definitions for a given problem domain. Of course, once *this* works, a specific community could choose to define its core concepts and ontologies through a set of standard-library-like-types, e.g. `string_view`, `array` or `span`-like types.

- Does not duplicate parameter creation: defining a parameter should be as simple as adding a member to a structure. The parameter's value should not be of a complicated, custom library type; just using `int` or `float` should work. At no point one should have to write the name of a variable twice, e.g. with a macro system such as Boost.Fusion with `BOOST_FUSION_ADAPT_STRUCT`, or with pybind-like templates: remember, we do not want our code to have any dependency !

- Allows to specify metadata on parameters: as one could see, it is necessary to be able to define bounds, textual descriptions, etc... for the inputs to the algorithm. For instance, the algorithm author may want to define a help text for each of the parameters, describing how each control will affect the result.

- Allows that definition to be *automatically* used to generate binding code to any of the environments, protocols, runtime systems mentioned above, with for only tool a C++20 compiler.

# Massaging the problem

Sadly, C++ does not offer true reflection on any entity: from the generic function `noise` defined above, it would be fairly hard to extract its parameter list, and reconstruct what we need to perform the above. Likewise, due to the lack of user-defined attributes, one wouldn't be able to tag the input / output parameters, to give them a name, bounds, etc.

We will however show that with very simple transformations, we can reach our goals ! 

### First transformation: function to class

This transformation is commonplace in C++: classes / structs are in general more convenient to use than straight function pointers. They are easier to pass as arguments, work better with the type system as template arguments, etc.

Let's apply it: 
```C++
// noise.hpp
#pragma once
struct noise
{
  float alpha;
  int beta;

  /* constexpr_in_some_future_standard */
  float operator()(float input) const
  {
    return std::fmod(std::pow(input, alpha), float(beta)) / beta;
  }
};
```

Thankfully, the actual implementation does not change ; we merely put some arguments as struct members instead. If the algorithm is complex with many settings and toggles, it is likely that this was already the case in your implementation.

What if, dear reader, I told you that, as of C++20, this is pretty much enough for achieving three of our four goals ?

### Mapping our class to a run-time API automagically

Assume the following imaginary run-time API for doing some level of processing, in cross-platform C89:

```C
typedef void* lib_type_t;

enum lib_argument_types {
    kFloat, kInt
};

lib_type_t lib_define_type(const char* name);
void lib_add_float(lib_type_t handle, const char* name, float* ptr, float min, float max);
void lib_add_int(lib_type_t handle, const char* name, int *ptr, int min, int max);

// Vararg is a list of lib_argument_types members, defining the arguments of the function.
void lib_add_method(lib_type_t handle, const char* name, void* ptr, ...); // mhhh VARARGS
```

To register our process to thar imaginary API, one may write the following, which would then be compiled as a .dll / .so / .dylib and be loaded by our runtime system through `dlopen` and friends:

```C++
noise algo;
void process(float* out, const float* in)
{ *out = algo(*in); }

lib_type_t main()
{
  auto r = lib_define_type("noise");
  lib_add_float(r, "alpha", &algo.alpha, 0., 1.); // oops
  lib_add_int(r, "beta", &algo.beta, 11, 247);
  lib_add_method(r, "process", reinterpret_cast<void*>(&process), kFloat, kFloat);
  return r;
}
```

What we want, is simply to use C++ to generate all the code above automatically.
That means, most importantly, to call the relevant `lib_add_*` function for each parameter with the correct arguments.

### Enumerating members

This is trivial, thanks to a library, Boost.PFR, which technically works from C++14 and up.
Note that the library is under the Boost umbrella but does not have any dependencies and can be used stand-alone.
The technique is basically a band-aid until we get true reflection: it counts the fields by checking whether the type T is constructible by N arguments of a magic type convertible to anything, and then uses destructuring to generate tuples of the matching count.

In a nutshell:
```C++
auto tie_as_tuple(auto&& t, size<1>) {
  auto&& [_1] = t;  
  return std::tie(_1);
}
auto tie_as_tuple(auto&& t, size<2>) {
  auto&& [_1, _2] = t;  
  return std::tie(_1, _2);
}
// etc... computer-generated
```

It opens a wealth of possibilities: iterating on every member, performing operations on them, etc. ; the only restriction being: the type must be an aggregate. Thankfully, that is not a very hard restriction to follow, especially if we want to write declarative code, which lends itself pretty well to using aggregates.

Let's for instance write a function that takes our struct and generates the `lib_add_float` / `lib_add_int` calls:

```C++
struct bind_to_lib {
  lib_type_t handle;

  void register_parameters(noise& algo)
  {
    struct {
        bind_to_lib& self;
        void operator()(float& f) const noexcept {
          lib_add_float(self.handle, "???", &f, ???, ???); 
        }
        void operator()(int& i) const noexcept {
          lib_add_int(self.handle, "???", &i, ???, ???); 
        }
    } visitor;
    boost::pfr::for_each(algo, visitor);
  }
};
```

This gets us 90% there: if our C API was just `lib_add_float(lib_type_t, float*);` that blog stop would stop right there ! 

But, as it stands, our API also expects some additional metadata: a pretty name to show to the user, mins and maxs...

### Second transformation: ad-hoc types for parameters

This transformation is mechanical, but complexifies our code a little bit.
We will change each of our parameters, into an anonymous structure containing the parameter:

```C++
float alpha;
```
becomes
```c++
struct {
  float value;
} alpha;
```

And at this point, it becomes easy to add metadata that will not have a per-instance cost, unlike a lot of runtime systems (for instance QObject properties used in Krita plug-ins):

```c++
struct {
  constexpr auto name() { return "α"sv; }
  float value;
} alpha;
```

The code sadly uglifies a little bit:

```C++
struct noise
{
  struct {
    constexpr auto name() { return "α"; }
    constexpr auto min() { return -1.f; }
    constexpr auto max() { return 1.f; }
    float value;
  } alpha;
  struct {
    constexpr auto name() { return "β"; }
    constexpr auto min() { return 11; }
    constexpr auto max() { return 247; }
    int value;
  } beta;

  float operator()(float input) const
  {
    return std::fmod(std::pow(input, alpha.value), float(beta.value)) / beta.value;
  }
};
```

There isn't a lot of wiggle room to improve. It is not possible to have static member variables in anonymous structs ; if one is willing to duplicate the name of the struct, it's possible to get things down to:

```C++
struct alpha {
  static constexpr auto name = "α";
  static constexpr auto min  = -1.f;
  static constexpr auto max  =  1.f;
  float value;
} alpha;
```

#### How user-defined attributes would help
Now, if we were in, say, C#, what we'd most likely write instead would instead just be: 
```C#
[Name("α")]
[Range(min = -1.f, max = 1.f)]
float alpha;
```

Simpler, isn't it ?
How neat would it be if we had [the same thing](https://manu343726.github.io/2019-07-14-reflections-on-user-defined-attributes/) in C++ ! There is some work towards that in Clang and the [lock3/meta metaclasses clang fork](https://github.com/lock3/meta/issues/215).

Although in practice some methods may be needed: for instance, multiple APIs require the user to provide a method which will from an input value, render a string to show to the user.

### Updating our binding code
We now have to go back and work on the binding function implementation: the main issue is that where we were using the actual type of the values, `boost::pfr::for_each` will give us references to anonymous types (or, even if not anonymous, types that we shouldn't have knowledge of in our binding code).

In our case, we assume (as part of our ontology), that *parameters* have a *value*. This is a compile-time protocol.

Thankfully, a C++20 feature, concepts, makes encoding compile-time protocols in code fairly easy.
Consider a member of our earlier visitor:

```C++
void operator()(???& f) const noexcept
{
  lib_add_float(r, ???, ???, ???, ???); 
}
```

We can for instance fill it that way : 
```C++
// we are writing the binding code, here everything is allowed !
#include <concepts>
...
void operator()(auto& f) const noexcept
  requires std::same_as<decltype(f.value), float>
{
  lib_add_float(r, f.name(), &f.value, f.min(), f.max());
}
```

And that would work with our current `noise` implementation.
But what if the program author forgets to implement the `name()` method ? Mainly a not-so-terrible compile error:

```
<source>:30:23: error: no member named 'name' in 'noise::(anonymous struct at <source>:18:5)'
  lib_add_float(r, f.name(), &f.value, f.min(), f.max());
                    ~ ^
<source>:48:12: note: in instantiation of function template specialization '(anonymous struct)::operator()<noise::(anonymous struct at <source>:18:5)>' requested here
    visitor(n.alpha);
           ^
```

If our API absolutely requires a `name()`, and a `value`, concepts are very helpful: 

```C++
template<typename T, typename Value_T>
concept parameter = requires (T t) {
  { t.value } -> std::same_as<Value_T>;
  { t.min() } -> std::same_as<Value_T>;
  { t.max() } -> std::same_as<Value_T>;
  { t.name() } -> std::same_as<const char*>;
};
```
Our code becomes:

```C++
void operator()(parameter<float> auto& f) const noexcept
{
  lib_add_float(r, f.name(), &f.value, f.min(), f.max());
}
```

Forgetting to implement `name()` now results in:

```
<source>:44:5: error: no matching function for call to object of type 'struct (anonymous struct at <source>:34:5)'
    visitor(n.alpha);
    ^~~~~~~
<source>:35:14: note: candidate template ignored: constraints not satisfied [with f:auto = noise::(anonymous struct at <source>:18:5)]
        void operator()(parameter<float> auto& f) const noexcept
             ^
<source>:35:25: note: because 'parameter<noise::(anonymous struct at <source>:18:5), float>' evaluated to false
        void operator()(parameter<float> auto& f) const noexcept
                        ^
<source>:29:7: note: because 't.min()' would be invalid: no member named 'min' in 'noise::(anonymous struct at <source>:18:5)'
  { t.min() } -> std::same_as<Value_T>;
      ^
```

whether that constitutes an improvement in readability of errors in our specific case is left as an exercise to the reader.

But, what if our algorithm *doesn't* actually need bounds ? We'd still want it to work in a bounded host system, right ? The host system would just choose arbitrary bounds that make sense for e.g. an input widget.

In this case, we'd get a combinatorial explosion of concepts: we'd need an overload for a parameter with a name and no range, an overload for a parameter with a range and no name, etc etc.  

### Handling optionality
As an algorithm author, you cannot specify every possible metadata known to man. We want our algorithm to be future-proof: even if refinements can be added, we want the code we write today to still be able to integrate into tomorrow's host.

Thankfully, the age-old notion of condition can help here ; in particular compile-time conditions depending on the existence of a member.

C++20 makes that trivial: 

```
void operator()(auto& f) const noexcept
  // We still need our "requires here", or a simpler concept
  // in order to have the right overload be selected.
  requires std::same_as<decltype(f.value), float>
{
  const char* name = "Parameter";
  float min = 0.f, max = 1.f;
  if constexpr(requires { name = f.name(); })
    name = f.name();
  if constexpr(requires { min = f.min(); })
    min = f.min();
  if constexpr(requires { max = f.max(); })
    max = f.max();

  lib_add_float(r, name, &f.value, min, max);
}
```

This way, the algorithm has maximal flexibility: it can provide the bare minimal metadata for a proof-of-concept, or give as much information as possible.

### Calling our code

There's not much difference with the previous technique when we want to call our process (`operator()`) function.

What we cannot do without reflection & code generation (metaclasses) is an entirely generic transformation from one of our algorithm's processing method, which, depending on the problem domain, could have any number of inputs / outputs of various types, to arbitrary run-time data. For instance, audio processors generally have inputs and outputs in the form of an array of channels of float / double values, plus the amount of data to be processed: 

```C++
void canonical_audio_processor(float** inputs, float** outputs, int frames_to_process);
```

While image processors would instead look like:
```C++
void canonical_image_processor(unsigned char* data, int width, int height);
```
There's no practical way to enumerate all the possible sets of arguments.

Thus, the author of the binding code has the responsibility of adapting the expected ontology for algorithms to the API we are binding to.

```C++
struct bind_to_lib {
  lib_type_t handle;
  void register_process(noise& algo)
  {
    auto process = [] (float* out, const float* in) { *out = algo(*in); };
    lib_add_method(handle, "process", reinterpret_cast<void*>(&process), kFloat, kFloat);
  }
}
```

Nothing prevents multiple cases to be handled: for instance, some plug-ins may have a more efficient, array-based, implementation for their process, which hosts whose main mode of operation is through arrays instead of single values can leverage if available, and make a fallback if not:

```C++
void register_process(noise& algo)
{
  if constexpr(std::invocable<noise, float, float>)
  {
    auto process = [] (float* out, const float* in) { *out = algo(*in); };
    lib_add_method(handle, "process", reinterpret_cast<void*>(&process), kFloat, kFloat);
  }
  else if constexpr(std::invocable<noise, const float*, float*, std::size_t>)
  {
    auto process = [] (float* out, const float* in, std::size_t n) {
      for(std::size_t i = 0U; i < n; i++) 
        out[i] = algo(in[i]); 
    };
    lib_add_method(handle, "process", reinterpret_cast<void*>(&process), kFloat, kFloat);
  }
}
```

# Benefits

Data-flow graphs can be done both at compile-time and run-time.