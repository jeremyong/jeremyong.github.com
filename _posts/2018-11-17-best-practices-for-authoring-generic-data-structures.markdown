---
layout: post
title: Best Practices for Authoring Generic Data Structures
categories: C++ graphics
---

This is a collection of ideas I've developed over the years that have resulted in higher
quality and more ergonomic code. In this article, I'm going to say the caveat once (right now)
that you should always code and architect for your particular workflow, and these ideas may or may not
apply. Henceforth, I'm going to be prescriptive about what I think a good set of patterns for,
and do my best to provide the rationale. I'm not going to talk about actual data structures
themselves, but instead about design principles and coding practices that I think apply to
all data structures as it relates to C++. In the code examples, pretend I did all the `constexpr`,
`[[nodiscard]]`, `noexcept`, and any other aspects of the attribute and modifier zoo properly
(omitted for brevity).

## Moving away from the "standard" way

For the sake of example throughout this article, we'll talk about authoring the most primitive
of useful data structures (the quadratically resizing array I'll just call `vector` here).
If I asked most people to sketch the implementation, they'd probably come up with something that
rhymes with the following:

```c++
template <typename T>
class vector
{
public:
    // A bunch of constructors handling various types of initialization, moves, and copies
    // A destructor that invokes `delete` on `data_` if non-null
    // An iterator type, and `begin`, `end`, `rbegin`, `rend`, and their const equivalents
    // Methods for mutating the `vector` like `push_back`, `emplace_back`, `clear`, etc.
    // Operators and accessors for reading and writing to constituent data

private:
    void grow(); // Invoked when the size_ is about to grow beyond the capacity_

    T* data_ = nullptr;
    size_t size_ = 0;
    size_t capacity_ = 0;
};
```

This is not a bad place to start; after all, most standard containers you use will resemble
this style of implementation. And I would contend that often times, we can improve on this
a good deal.

## Interface weaknesses of the vanilla approach

The primary gripe I have with data structures authored as above is the template parameter `T`.
On the surface, this seems necessary. The data structure should know how `T` is constructed,
moved, and destructed, right? After all, such operations on `T` all need to happen over the
lifetime of the data structure. Furthermore, `T` comes with information about the size and
alignment requirements for allocating memory to store it.

To understand why this might be a problem, consider the issue of writing a common interface
to, say, a rendering backend. You want to provide a class that contains data structures that
hold information like shader handles, pipeline objects, buffer handles, and texture data.
This interface may be implemented numerous times for this project, to provide an OpenGL, Vulkan,
Metal, and D3D backend (possibly across multiple versions of each). Furthermore, for various
types of data, the alignment or size requirements may not even be known until runtime. Many
buffer types have extended alignment restrictions that must be queried by the GPU.

For this type of interface, using the naive `vector` class implemented above, we cannot house it
in the common interface layer. It needs to be duplicated within each backend implementation
of the renderer. Furthermore, we may need to choose a very pessimistic alignment requirement
to ensure that things work portably. All this means that we both waste memory, and also will
have a tough time sharing common code that needs to operate on these data structures. For
example, we'd love to have a common set of functions for managing the lifetimes of each resource
type, and tracking memory budgets and usage.

## An alternative design

The problem, in my opinion, is that there is too much internal coupling with this design. The
data structure simultaneously manages both the memory and data structure algorithms *and* the
lifecycle operations of the type `T`.

This is where my own code takes a sharp left turn from other generic code I've seen. I believe
that data structures should in fact be provided as two separate classes, the *storage* class,
and the *view* class. The *storage* class should only be concerned with the size and alignment
restrictions of its internals, as well as its policy for when data needs to move around or
allocate. The *view* class should be a thin type-safe adaptor that can access the data without
the need for excessive casting.

Here's an example, of what this might look like:

```c++
class vector_storage; // Forward declaration of non-templatized vector

template <typename T>
class vector_view
{
public:
    vector_view(vector_storage& v) : vector_storage_{v} {}

    // An iterator type, and `begin`, `end`, `rbegin`, `rend`, and their const equivalents
    // Methods for mutating the `vector` like `push_back`, `emplace_back`, `clear`, etc.
    // Operators and accessors for reading and writing to constituent data
private:
    vector_storage vector_storage_;
};

class vector_storage
{
public:
    vector(size_t element_size, size_t alignment);

    // template member functions for push_back, iterators, etc.

    // For example:
    template <typename T>
    void push_back(T&& value)
    {
        static_assert(sizeof(T) == element_size_, "Type size mismatch");
        static_assert(alignof(T) <= alignment_, "Alignment restriction violation");
        if (size_ == capacity_) grow<T>();
        *(begin<T>() + size_) = std::forward<T>(value);
        ++size_;
    }

private:
    template <T>
    void grow()
    {
        capacity_ *= 2;
        void* temp = data_;
        _data = std::malloc(capacity_ * element_size_);
        if constexpr (std::is_trivially_copyable<T>::value)
        {
            std::memcpy(_data, temp, size_ * element_size_);
        }
        else
        {
            // Loop through old memory and for each element_size_ block,
            // perform a cast and move to the new location
        }
    }

    void* data_;
    size_t size_;
    size_t capacity_;
    size_t element_size_;
    size_t alignment_;
};
```

Hopefully, this stub code is enough to get the gist of the idea. We now have two classes, one
templatized, one not. The one that isn't templatized is concerned only with the actual
storage algorithm, but defined templatized member functions so it can essentially transmute
itself as necessary. Now, with this interface, we can have a common class interface define
the structure directly, with common functions that operate on it. The various platform
specific implementations can create a "view" to operate on the structure in a type-safe manner.

Note that we an define another class like so:

```c++
template <typename T>
class vector : public vector_view<T>
{
public:
    vector()
        : vector_storage_(sizeof(T), alignof(T))
        , vector_view<T>(vector_storage_)
    {}

private:
    vector_storage vector_storage_;
};
```

and this completely recovers the original semantics of the vector! However, now, we have
several significant advantages. For the price of the storage space for the element size
and alignment requirement, we have an adaptor type we can use to alias memory if necessary.
We can handle runtime element sizing and alignment requirements. We can reuse more generic
code (even if different platform requirements store different types of data in the structure).
And we always have the fully type-safe version to fall back on if necessary.

## Views for days

There is another strength to this approach which I haven't touched on yet.
The type-safe `view` adaptor can be templatized on the `storage` class itself. This means
that we can have a single `view` adaptor for an entire class of data structures. For example,
all traversable, random access data structures could be wrapped by the same view template.
Similarly, we can provide a single uniform view for all associative containers. Looking
forward at C++ concepts, each view template accepts a data structure constrained to a
particular concept.

The possibilities of this are myriad. Here are some ideas for view-types you could provide:

- Want your container to be threadsafe? Keep the storage abstraction, but tack on a thread-safe view
- Have a version of the view that is fully instrumented with debugging and tracking that you enable as needed.
- Provide a view that takes a `thread_id` as a parameter and enforces that all access to the data
  structure originate from that thread.

With this approach, you can write such views *once* for a particular data structure concept and it's there forever.
That's generic code at work!

## Providing the allocator/arena

Another point worth making is that all my data structures accept an allocator of some
sort as an argument. Exposing the allocator as a template parameter really doesn't make
sense, since you want to usually allocate the memory in a specific *instance* of the
allocator. As I generally work with pretty performance sensitive code, this is a must,
and I rarely ever rely on the system malloc (although I have an allocator that passes
through to malloc occassionally for convenience in testing). This completely bypasses
the need for polymorphic allocators, which are needed to permit type comparisons between
data structures that would otherwise by the same except for the allocator type. To me,
this is a hot mess that I think is worth avoiding entirely in your own code.

## Yet more opinions

Consider making your default views and type-safe wrappers fail when passed a non-trivially-copyable
template parameter if performance is crucial. Usually, if you're writing a custom
data structure anyways, this is because you have particular needs that aren't met
by the out-of-box containers. As such, if you're storing rich objects that have
fancy moves and resource management semantics, you're likely not doing yourself a
favor. Enforcing various type traits, size/alignment restrictions, etc at the view
level at compile-time is *great* for forcing the user to design the data layout
correctly upfront.

## Conclusion

TL;DR For writing high-performance generic data structures

- Separate type-specific access and mutation code to a separate class for forwards the type to
  a storage class. The storage class has templatized member functions, but is itself non-templatized.
- Provide a single view for each broad class of data structures that behave with similar semantics.
- Provide different view types to handle differences in access patterns (thread safety, logging, etc).
- Always accept an allocator as an argument with an interface exposed in your code somewhere. Don't
  bake it into your type-signature.
- Enforce better defaults for encouraging data-oriented SoA design and hide less-performant patterns
  behind more verbose interfaces.
