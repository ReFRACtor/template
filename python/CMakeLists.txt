# set the name of the python package
set(python_package_name "template_python")

# new policy is to pass swig_module_name as -module. we don't actually need
# this, but no reason not to turn it on. without this, you get a warning
# message from cmaker version 3.14 and later
if(policy cmp0086)
  cmake_policy(set cmp0086 new)
endif()
if(policy cmp0078)
  cmake_policy(set cmp0078 old)
endif()

# find python. note for conda on the mac you need to find the interpreter
# before the libs (i.e., the order is important). otherwise this finds
# the system python for libraries but uses conda for the interpreter, which
# of course causes problems. this goes away if you do this in the other order.
find_package(pythoninterp 3.0 required)
# find python
find_package(pythonlibs 3.0 required)
include_directories(${python_include_path})

# find numpy
find_package(numpy required)
include_directories(${numpy_include_dirs})

# find swig
find_package(swig 3.0 required)
include(${swig_use_file})
set(cmake_swig_flags "")

# output swig files to a directory named for the package
set(package_outdir ${cmake_current_binary_dir}/${python_package_name})
set(cmake_swig_outdir ${package_outdir})

# add the c++ library directory to the list of includes
include_directories(${thirdparty_includes} ${cmake_current_source_dir} ${cmake_source_dir}/lib)

# which interface files should be used for creating modules
set(module_interface_files 
    template.i
)

# use c++ mode for swig interface files, turn on all warnings
set_source_files_properties(${module_interface_files} properties cplusplus on swig_flags "-wall") 

foreach(swig_file ${module_interface_files})
    get_filename_component(module_name ${swig_file} name_we)

    # figure out which other interface files are included, and hence depended on by the module interface file
    # regenerate swig wrappers and recompile if those change
    execute_process(command ${cmake_current_source_dir}/interface_deps.sh ${cmake_current_source_dir}/${swig_file}
        output_variable dep_interface_files output_strip_trailing_whitespace result_variable interface_dep_res error_variable interface_dep_err)

    if(${interface_dep_res})
        message(fatal_error "error running interface_deps.py:\n${interface_dep_err}\n${dep_interface_files}")
    endif()

    # make module recompile if any of the interface files change
    set(swig_module_${module_name}_extra_deps ${dep_interface_files})

    if(cmake_version version_greater 3.7)
        swig_add_library(${module_name} type shared language python sources ${swig_file})
    else()
        swig_add_module(${module_name} python ${swig_file})
    endif()
    if(apple)
      # compiling with python causes a problem on the mac using conda.
      # see https://github.com/pytorch/pytorch/commit/73f6715f4725a0723d8171d3131e09ac7abf0666
      # the symbols get resolved when loaded into python, so this doesn't cause
      # a problem.
      swig_link_libraries(${module_name} ${thirdparty_libraries} template)
      set_target_properties(${swig_module_${module_name}_real_name} properties link_flags "-undefined dynamic_lookup")
    else()
      swig_link_libraries(${module_name} ${python_libraries} ${thirdparty_libraries} template)
    endif(apple)
    target_compile_definitions(${swig_module_${module_name}_real_name} private swig_type_table=refractor)

    list(append swig_libs ${swig_module_${module_name}_real_name})
endforeach()

set_target_properties(${swig_libs} properties library_output_directory ${package_outdir})

# ensure module has .so extension (required by python) on all platforms including os x (where dylib would otherwise be the case)
set_target_properties(${swig_libs} properties suffix ".so")

# make a custom target to allow doing: make python
add_custom_target(python depends ${swig_libs})

# add python files to install target
file(glob install_interface_files ${cmake_current_source_dir}/*.i)
install(files ${install_interface_files} destination include)

execute_process(command ${python_executable} -c "import distutils.sysconfig; print(distutils.sysconfig.get_python_lib(true, false, ''))" output_variable python_install_dir output_strip_trailing_whitespace)

# install python source files from the python/ source directory
install(directory "${cmake_current_source_dir}/" destination "${python_install_dir}/${python_package_name}"
        files_matching
        pattern "*.py")

# install swig created files
install(directory ${cmake_swig_outdir} destination ${python_install_dir}
        files_matching
        pattern "*.${cmake_shared_library_suffix}"
        pattern "*.so"
        pattern "*.py")

# expose python install path to parent scope
set(python_install_dir ${python_install_dir} parent_scope)
      
