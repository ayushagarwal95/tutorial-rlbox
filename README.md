# Tutorial for using RLBox

This is a tutorial to get started when using RLBox for sandboxing a library in your application. 

RLBox is a framework available in C++ to sandbox libraries aiming to provide software fault isolation without significant performance overhead.

RLBox has two types of sandbox - No-Op and WASM. No-op does not actually perform any software fault isolation, and is provided to allow for an easier integration into a production application. 

Documentation for the framework is available here: [RLBox Documentation](https://plsyssec.github.io/rlbox_sandboxing_api/sphinx/). 

RLBox has been used in Firefox to sandbox a few libraries, the source code for them is available at the following links:
* libgraphite: [Source](https://searchfox.org/mozilla-central/source/gfx/thebes/gfxFontEntry.cpp)
* libogg: [Source](https://searchfox.org/mozilla-central/source/dom/media/ogg/OggDemuxer.cpp#95)
* hunspell: [Source](https://searchfox.org/mozilla-central/source/extensions/spellcheck/hunspell/glue/RLBoxHunspell.cpp)

## Before Starting

* RLBox does **NOT** allow passing pointers to application memory into the sandbox. This is very important as these are the most common cause of build errors when sandboxing using RLBox.
* Any return value from a sandboxed function will be marked `tainted` and **CANNOT** be used without verifying.
* RLBox does **NOT** support functions with variable arguments. Make sure that you write wrapper functions in the library to convert functions with variable arguments into one with fixed arguments.
* After every change, it is best to do a re-build and a quick test if things work as expected.
* RLBox only supports the C ABI - so the library to be sandboxed must have a C interface for the application.
* Currently, RLBox based WASM sandboxing is only available on 64-bit based MacOS and Linux.

## Steps

### 1. Creation and Deletion of Sandbox
* Identify sandbox usage in your application - places where you need to create sandboxes, and then where you could destroy/delete them.
* Start by using noop sandbox - first add functions to create and destroy sandbox like below:

  ```C++
    /* The macro below should be defined before including
    any header file from RLBox */
    #define RLBOX_SINGLE_THREADED_INVOCATIONS

    //Include these header files
    #include "rlbox_types.hpp"
    #include "rlbox_config.h"

	//No-op Sandbox
	#define RLBOX_USE_STATIC_CALLS() rlbox_noop_sandbox_lookup_symbol
	#include "rlbox_noop_sandbox.hpp"
	using rlbox_sandbox_type = rlbox::rlbox_noop_sandbox;

	using rlbox_sandbox_lib = rlbox::rlbox_sandbox<rlbox_sandbox_type>;

	rlbox_sandbox_lib* CreateSandbox() {
		  rlbox_sandbox_lib* sandbox = new rlbox_sandbox_lib();
		  sandbox->create_sandbox();
		  return sandbox;
	}

	void DeleteSandbox (rlbox_sandbox_lib *sandbox) {
		sandbox->destroy_sandbox();
		delete sandbox;
	}
  ```
* Add calls to the `CreateSandbox()` and `DeleteSandbox()` at the places identified.
* Build your application and check for any build errors introduced. We would also suggest to quickly run the application and check if everything is working. It should be as no changes have been made to the functional part of the code.  

### 2. Sandbox Function Calls
* Once done with the above, start sandboxing function calls of the library. Remember that, RLBox does not allow any pointers from the application memory to be passed into the sandbox memory.  This is probably the most common cause of build errors when sandboxing a library in an application.  
Note: This requires looking at the API of the library being sandboxed for in and out parameters, return values.
The below example is from sandboxing libopus in Firefox.  

  ```C++
  /* original code snippet
  Here, r is an integer which is being passed as a pointer to the API
  mMappingTable.Elements() is expected to be a pointer by the API
  so - first step we allocate memory for them in the sandbox memory
  */
  mOpusDecoder = opus_multistream_decoder_create(
        mOpusParser->mRate, mOpusParser->mChannels, mOpusParser->mStreams,
        mOpusParser->mCoupledStreams, mMappingTable.Elements(), &r);
  ```

  This gets changed to:

  ```C++
  /* modified snippet */
  auto t_r = mSandbox->malloc_in_sandbox<int>(1);
  auto sandboxedMappingTable = mSandbox->malloc_in_sandbox<uint8_t>(mMappingTable.Length());
  //copy original mapping table into sandbox using rlbox::memcpy
  rlbox::memcpy(*mSandbox, sandboxedMappingTable,
          mMappingTable.Elements(), mMappingTable.Length());
  /* we also modify the type of mOpusDecoder as it is a pointer returned that is
  needed in the future calls to the opus library when decoding media, hence it needs to
  be tainted. */
  mOpusDecoder = mSandbox->invoke_sandbox_function(opus_multistream_decoder_create,
      mOpusParser->mRate, mOpusParser->mChannels, mOpusParser->mStreams,
      mOpusParser->mCoupledStreams, sandboxedMappingTable, t_r);
  /* we need to free the memory allocated in sandbox once we know it is not needed */
  mSandbox->free_in_sandbox(sandboxedMappingTable);
  /* Looking at the opus_multistream_decoder_create API documentation, t_r is the
  return code. To be able to check the value we need to either verify it (next step
  covers that) or RLBox provides an API which could be used for the timebeing */
  int r = *(t_r.unverified_safe_pointer_because(1, "migrating to RLBox");
  ```
  * `malloc_in_sandbox`:  allows you to allocate memory within the sandbox, returns a tainted pointer.
  * `free_in_sandbox`: allows you to free memory allocated within the sandbox.
  * `unverified_safe_pointer_because`: removes the taint from the value and expects as parameters the size being copied, along with a string as to why the tainted value is being used without verification. One should not be using this freely, as it defeats the purpose of sandboxing - it should only be used when necessary or with a valid reason which could be audited during code reviews.

  RLBox provides 2 sets of APIs one could use to incrementally migrate to sandboxed APIs. Both sets should be used only when migrating, and the developer should work towards removing as many instance of these APIs when the migration/integration to RLBox is done.

  * `unverified_safe_*`: can be used to "untaint" any `tainted` value coming from the sandbox and use it in the application without writing verifiers for them.
  * `UNSAFE_*`: can be used to cast a pointer/variable in the application memory to its' `tainted` equivalent and pass it as a parameter to a sandboxed function.  

  There is another difference among the above 2 set of APIs, WASM sandoxing can be enabled with `unverified_safe_*` being present in the application, however, with `UNSAFE_*` present anywhere in the application, WASM sandboxing cannot be enabled. It will throw a build error.

* We **strongly** recommend building and testing for every function you sandbox. The testing maybe a quick one that tests the core functionality of the library you are sandboxing. Eg: In our case (libopus), we built Firefox and launched a YouTube video since it uses libopus for audio.

* Once all the functions are sandboxed, you should run the full test suite you have related to the library being sandboxed and look for any unexpected behaviour.


### 3. Writing Verifiers

* Once all functions are sandboxed, you could start writing the verifiers for each return value and out parameter from the library APIs.
* For writing these verifiers, you need to go through the API documentation of the library to make note of the expected values.
* Verifiers are written using the following set of APIs, they expect as parameter a verifier function (a promise) which can run checks on the tainted value and copy over a suitable value into the application memory. Some of the common APIs you would need for verifying are:
  * `copy_and_verify()`
  * `copy_and_verify_buffer()`
  * `copy_and_verify_string()`
  * `copy_and_verify_range()`

  An example is given below from our experience with libopus on Firefox:
  ```C++
  /* Consider the following sandboxed function - it returns the number of frames in the passed data */
  auto frames_number = mSandbox->invoke_sandbox_function(
      opus_packet_get_nb_frames, t_aSampleData, aSample->Size());
  /* The maximum number of frames in an opus packet is 63*2880 - so we check if
  the number returned is positive and within the valid range. The API could
  also return OPUS_BAD_ARG or OPUS_INVALID_PACKET in case of an error - so
  those checks are also present. If the values do not match either values,
  we return OPUS_INVALID_PACKET to the application */
  int verifiedFramesNumber = frames_number.copy_and_verify(
                            [](int frames_number) {
      if (frames_number > 0 && frames_number <= (63*2880))
        return frames_number;
      else if (frames_number == OPUS_BAD_ARG ||
              frames_number == OPUS_INVALID_PACKET)
        return frames_number;
      else
        return OPUS_INVALID_PACKET;
  });
  ```

* Once the verifiers are written, make sure to have built your application and tested it. It would be better to check for any instances of the `UNSAFE_*` in the application before proceeding to the next step.


### 4. Enabling WASM Sandbox
There are 2 things that you must do to be able to enable the WASM based sandboxing.
  #### 4.1 Building your library via WASM toolchain

  * C code can be compiled into WASM using a recent version of `clang`
  * A good guide of compiling C code to WASM is available [here](https://depth-first.com/articles/2019/10/16/compiling-c-to-webassembly-and-running-it-without-emscripten/).
  * One additional requirement from RLBox in this case is that it has to be compiled along with the file [lucet_sandbox_wrapper.c](https://github.com/PLSysSec/rlbox_lucet_sandbox/blob/master/c_src/lucet_sandbox_wrapper.c)
  * Once a WASM module is generated, it needs to be compiled into the native architecture using lucet - a WASM compiler.
  * RLBox needs the path to the binary generated by the Lucet compiler for WASM based sandboxing.
  
  Now, we see an example which follows the above steps to generate a module that could be sandboxed based on WASM in RLBox.

  Assuming that the library is a single source file `foo.c` and the setup steps from the above mentioned guide on compiling C into WASM. The following command should compile the library into a WASM module.
  
  [//]: # (Markdown inline comment - did not include the -fPIC and -shared flags here as the wrapper file has a main defined so I assumed it will not be needed but I am not sure.)

  ```
  clang \
  --target wasm32-unknown-wasi \
  --sysroot /tmp/wasi-libc \
  -Wl,--export-all \
  -o libfoo \
  copy.c lucet_sandbox_wrapper.c
  ```

This generates the WASM module `libfoo`, next we use lucet to compile this to a shared library.

```
lucetc                                \
--bindings lucet-wasi/bindings.json   \
    --guard-size "4GiB"               \
    --min-reserved-size "4GiB"        \
    --max-reserved-size "4GiB"        \
    libfoo                            \
    -o libfoowasm.so
```

This generates the shared library which RLBox could load and provide us with in-process sandboxing based on WASM.

  #### 4.2 Change sandbox type to WASM from noop

Once the library binary is ready, a few changes are needed to enable RLBox's WASM sandboxing mode. The steps are described below.

The description is written in a way such that a compile time flag determines if the sandboxing would be WASM based or not sandboxing is needed. It is not necessary to do it this way. The initial snippet from Section 1 is to be modified as follows:

```C++
/* The macro below should be defined before including
 any header file from RLBox */
#define RLBOX_SINGLE_THREADED_INVOCATIONS

//Include these header files
#include "rlbox_types.hpp"
#include "rlbox_config.h"

#ifdef SANDOXING_DISABLE
//No-op Sandbox
#define RLBOX_USE_STATIC_CALLS()rlbox_noop_sandbox_lookup_symbol
#include "rlbox_noop_sandbox.hpp"
using rlbox_sandbox_type = rlbox::rlbox_noop_sandbox;

#else
//WASM sandbox
#include "rlbox_lucet_sandbox.hpp"
using rlbox_sandbox_type = rlbox::rlbox_lucet_sandbox;
#endif 
using rlbox_sandbox_lib = rlbox::rlbox_sandbox<rlbox_sandbox_type>;

rlbox_sandbox_lib* CreateSandbox() {
	rlbox_sandbox_lib* sandbox = new rlbox_sandbox_lib();
  #ifdef SANDBOXING_DISABLE
  //No-op sandbox, no further config needed
	sandbox->create_sandbox();
  #else
  //WASM sandbox
  const char path[] = "/path/to/libfoowasm.so";
  //Set this to true if the application loads the library 
  //the lucet sandbox create call, for eg: via dlopen()
  bool external_load_true = false; 
  //Set this to true only if you need std I/O from the
  //library
  bool allow_stdio = false;
  sandbox->create_sandbox(path, external_load_true, allow_stdio);
  #endif

	return sandbox;
}

void DeleteSandbox (rlbox_sandbox_lib *sandbox) {
	sandbox->destroy_sandbox();
	delete sandbox;
}
```
With these changes in place, WASM sandboxing mode has been enabled in the application for the specific library. 