---
title: "The Gaudi framework"
teaching: 60
exercises: 60
questions:
- "Why is Gaudi used?"
objectives:
- "Learn about the Gaudi framework."
keypoints:
- "Work with Gaudi."
---

## Goals of the design

Gaudi is being improved with the following goals in mind:

* Improve scaling by making the interface simpler and more uniform. This allows simpler code, thereby allowing more complex code to be written.
* Improve memory usage, to be cache friendly and avoid pointers to pointers. Use C++11 move where possible.
* Thread-safe code, with no state or local threading. This allows Gaudi to be multithreaded in the future.

In order to reach those goals, the following choices should be made when designing Gaudi algorithms:

* Proper `const` usage: Since C++11, `const` and `mutable` are defined in terms of data races (see [this video](https://channel9.msdn.com/posts/C-and-Beyond-2012-Herb-Sutter-You-dont-know-blank-and-blank)).
* No global state; otherwise it would be impossible to multithread.
* Tools: should be private or public.
* Ban `beginEvent`/`endEvent`, since there might be multiple events.
* No explicit `new`/`delete` usage (C++11 goal).
* Remove `get/put<MyData>("")` for safer constructs.

## Gaudi properties

One of the standout features of new Gaudi is in the new properties; they can be directly defined in the class definition instead of the implementation and used everywhere in the class, instead of requiring special calls in the constructor and custom management of the data. So the following now creates a property:

```cpp
Gaudi::Property<int> m_some_int{this, "SomeInt", 0, "Description of some int"};
```

This defines a member variable of type `Gaudi::Property<int>`, and calls the constructor on initialization with several parameters. The first is `this`, a pointer to the current class. The second is the property name. The third is the default value. You can also optionally give a description.

> ## Notes on the new syntax
> 
> The usage of `this` is a common feature of the new interfaces, giving Gaudi components access to the algorithms that contain them. The location of `this` is not consistent across the components, however, as you will see with `AnyDataHandle`.
> 
{: .callout}


## AnyDataHandle

The requirement to inherit from `DataObject` has been lifted with `AnyDataHandle`. This provides a wrapper than can be put into the TES, and can be retrieved from it as well. Assuming you have some object called `Anything`, you can wrap it in `AnyDataHandleWrapper` and then use the traditional get/put syntax:

```cpp
auto vec = new AnyDataWrapper<Anything>(Anything{1,2,3,4});
put(vec, "/Event/Anything");
```

```cpp
auto base = getIfExists<AnyDataWrapperBase>("/Event/Anything");
const auto i = dynamic_cast<const AnyDataWrapper<Anything>*>(base);
const Anything anything = i->getData();
```

This, however, is significantly convoluted. A much cleaner way to add an `AnyDataHandle` is to construct it as a member of your class, just like a `Gaudi::Property`, and then use the `.get` and `.put` methods.

```cpp
AnyDataHandle<Anything> m_anything{"/Event/Anything", Gaudi::DataHandle::Writer, this};
m_anything.put(Anything{1,2,3,4});
```

```cpp
AnyDataHandle<Anything> m_anything{"/Event/Anything", Gaudi::DataHandle::Reader, this};
const Anything* p_anything = m_anything.get();
```


> ## Notes on the new syntax
>
> Adding AnyDataHandle as a member variable breaks compatibility with the `using` statement, so you will need to explicitly define the constructor.
{: .callout}

This works by replacing inheritance with *type erasure*.

## Gaudi::Functional

Rather than provide a series of specialised tools, the new `Gaudi::`Functional` provides a general building block that is well defined and multithreading friendly. This standardizes the common pattern of getting data our of the TES, working on it, and putting it back in (in a different location). This reduces the amount of code, makes it more uniform, and encourages 'best practice' procedures in coding in the event store. The "annoying details", or scaffolding, of the design is left to the
framework.

A functional, in general, looks like this example of a transform algorithm to sum two bits of data:

```cpp
class MySum
 : public TransformAlgorithm<OutputData(const Input1&, const Input2&)> {

     MySum(const std::string& name, ISvcLocator* pSvc)
      : TransformAlgorithm(
           name,
           pSvc, {
              KeyValue("Input1Loc", "Data1"),
              KeyValue("Input2Loc", "Data2") },
           KeyValue("OutputLoc", "Output/Data")) { }

    OutputData operator()(const Input1& in1, const Input2& in2) const override {
        return in1 + in2;
    }
```

This example highlights several features of the new design. This starts by inheriting from a templated class, where the template defines the input and output expected. Different functionals may have no input and/or output, and might have multiple inputs or outputs. The constructor takes KeyValue objects that define the data inputs and outputs, with multiple data elements input as a initializer list. The `operator()` method is overridden, and is `const` to ensure thread safety
and proper design. This takes the inputs and returns an output.




The functionals available in the framework are named by the data they work on (with examples):

* One input, no output: `Consumer`
    - `Rec/RecAlgs`: `EventTimeMonitor`, `ProcStatusAbortMoni`, `TimingTuple`
* No input, one or more output: `Producer`
    - Should be used for File IO, constant data
* True/False only as output: `FilterPredicate`
    - `Phys/LoKiHlt`: `HDRFilter`, `L0Filter`, `ODINFilter`
    - `Phys/LoKiGen`: `MCFilter`
    - `Hlt/HltDAQ`: `HltRoutingBitsFilter`
    - `Rec/LumiAlgs`: `FilterFillingScheme`, `FilterOnLumiSummary`
* One or more input, one output: `Transformer`
* One or more input, more than one output: `MultiTransformer`
* Identical inputs, one output: `MergingTransformer`
    - `Calo/CaloPIDs`: `InCaloAcceptanceAlg`
    - `Tr/TrackUtils`: `TrackListMerger`
* One input, identical outputs: `SplittingTransformer`
    - `Hlt/HltDAQ`: `HltRawBankDecoderBase`
* Converting a scalar transformation to a vector one: `ScalarTransformer`
    - `Calo/CaloReco`: `CaloElectronAlg`, `CaloSinglePhotonAlg`
    - This is used for vector to vector transformations where the same algorithm is applied on each vector element with 1 to 1 mapping.

> A note on stateless algorithms
> 
> A stateless algorithm (one that does not store state between events) provides several important benefits:
>
> * Thread safety
> * Better scalability
> * Leaner code
>
> The downside is that a lot of code needs to be migrated.
{: .callout}

### Consumer

This is a class that takes in TES data. It looks like:

```cpp
class EventTimeMonitor : public Gaudi::Functional::Consumer<
                void(const LHCb::ODIN&),
                Gaudi::Functional::Traits::BaseClass_t<GaudiHistoAlg>
                > {
    
    EventTimeMonitor(const std::string& name, ISvcLocator* pSvcLocator)
        : Consumer(name, pSvcLocator, KeyValue{"Input"}, LHCb::ODINLocation::Default } ) 
    { ... }

    void EventTimeMonitor::operator()(const LHCb::ODIN& odin) const override
    { ... }
```

The class is created inheriting the templated class `Consumer`, with the signature of the `operator()` function as it's first argument, and the `Traits`, which allows the selection of a base class to use (default is `GaudiAlgorithm`.

The constructor should delegate the Consumer constructor, with one additional argument, the input that it will use. This is done by constructing a `KeyValue` with property `"input"` set (in this case, to the default location).

The work is then done in the `operator()` method.

## Producer

This is the inverse of the consumer. Produces `Out1` to `OutN` given nothing:

```cpp
template <typename... Out, typename Traits_>
class Producer<std::tuple<Out...>(), Traits_> {
      virtual std::tuple<Out...> operator()() const = 0;
}
```

The tuple that is produced can be replaced with a single type if only one output is produced. The pass is done by value, due to move improvements in modern C++. 

## FilterPredicate

This blocks algorithms behind it, returns "filterPassed".

## Transformers

Split or merge containers.

# Conversion from the old framework to the new

* Try to convert everything to a `Gaudi::Functional` algorithm. Split into smaller basic algorithms and build a `SuperAlgorithm` from the smaller ones if needed.
* At least try to migrate to data handles + tool handles.
* Make sure the event loop code is `const`; do not cache event dependent data. Interface code, especially for tools, should be `const`.
* Try to migrate away from `beginEvent`, `endEvent` incidents.
* Please add as many tests as possible!


