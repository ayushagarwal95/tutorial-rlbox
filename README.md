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
	define RLBOX_USE_STATIC_CALLS() rlbox_noop_sandbox_lookup_symbol
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

### Sandbox Function Calls
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
  * `UNSAFE_*`: can be used to cast a pointer/variable in the application memory to its' `tainted` equivalent and pass it as a parameter to a sandboxed function. There is another difference, WASM sandoxing can be enabled with `unverified_safe_*` being present in the application, however, with `UNSAFE_*` present anywhere in the application, WASM sandboxing cannot be enabled. It will throw a build error.

