set(LLVM_OPTIONAL_SOURCES
  ocl-runtime-wrappers.cpp
  mlir-ocl-runner.cpp
  )
set(LLVM_LINK_COMPONENTS
  Core
  Support
)

if(MLIR_OCL_RUNNER_ENABLED)
  if (NOT ("NVPTX" IN_LIST LLVM_TARGETS_TO_BUILD))
    message(SEND_ERROR
      "Building the mlir ocl runner requires the NVPTX backend")
  endif()

  # We need the libpocl.so OR libOpenCL.so library.
  find_library(OCL_RUNTIME_LIBRARY OpenCL)
  if (NOT DEFINED OCL_RUNTIME_LIBRARY) 
    message(SEND_ERROR
      "Building the mlir ocl runner requires a working OpenCL/POCL library install")
  endif()

  add_llvm_library(ocl-runtime-wrappers SHARED
    ocl-runtime-wrappers.cpp
  )
  target_include_directories(ocl-runtime-wrappers
    PRIVATE ${CMAKE_OCL_TOOLKIT_INCLUDE_DIRECTORIES}
    LLVMSupport
  )
  target_link_libraries(ocl-runtime-wrappers
    PUBLIC
    LLVMSupport
    ${OCL_RUNTIME_LIBRARY}
  )

  get_property(dialect_libs GLOBAL PROPERTY MLIR_DIALECT_LIBS)
  get_property(conversion_libs GLOBAL PROPERTY MLIR_CONVERSION_LIBS)
  set(LIBS
    ${dialect_libs}
    ${conversion_libs}
    MLIRJitRunner
    MLIRAnalysis
    MLIREDSC
    MLIRExecutionEngine
    MLIRIR
    MLIRParser
    MLIRSupport
    MLIRTargetLLVMIR
    MLIRTargetNVVMIR
    MLIRTransforms
    MLIRTranslation
    ${OCL_RUNTIME_LIBRARY}
  )

  # Manually expand the target library, since our MLIR libraries
  # aren't plugged into the LLVM dependency tracking. If we don't
  # do this then we can't insert the CodeGen library after ourselves
  llvm_expand_pseudo_components(TARGET_LIBS AllTargetsCodeGens)
  # Prepend LLVM in front of every target, this is how the library
  # are named with CMake
  SET(targets_to_link)
  FOREACH(t ${TARGET_LIBS})
    LIST(APPEND targets_to_link "LLVM${t}")
  ENDFOREACH(t)

  add_llvm_tool(mlir-ocl-runner
    mlir-ocl-runner.cpp

    DEPENDS
    ocl-runtime-wrappers
    )
  target_include_directories(mlir-ocl-runner
    PRIVATE ${CMAKE_OCL_TOOLKIT_INCLUDE_DIRECTORIES}
  )
  llvm_update_compile_flags(mlir-ocl-runner)
  target_link_libraries(mlir-ocl-runner PRIVATE ${LIBS} ${targets_to_link})

endif()
