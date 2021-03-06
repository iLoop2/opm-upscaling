# -*- mode: cmake; tab-width: 2; indent-tabs-mode: t; truncate-lines: t; compile-command: "cmake -Wdev" -*-
# vim: set filetype=cmake autoindent tabstop=2 shiftwidth=2 noexpandtab softtabstop=2 nowrap:

###########################################################################
#                                                                         #
# Note: The bulk of the build system is located in the cmake/ directory.  #
#       This file only contains the specializations for this particular   #
#       project. Most likely you are interested in editing one of these   #
#       files instead:                                                    #
#                                                                         #
#       dune.module                              Name and version number  #
#       CMakeLists_files.cmake                   Path of source files     #
#       cmake/Modules/${project}-prereqs.cmake   Dependencies             #
#                                                                         #
###########################################################################

cmake_minimum_required (VERSION 2.8)

set( OPM_CMAKE_ROOT "" CACHE PATH "Root directory containing OPM related cmake modules")

include (OpmInit OPTIONAL RESULT_VARIABLE OPM_INIT)

if (OPM_INIT) 

   # We actually found the OpmInit module without using the OPM_CMAKE_ROOT variable.
   # The build needs the variable $OPM_MACROS_ROOT to be set, so here we infer it 
   # by backtracing from the location of the OpmInit module.
   get_filename_component( TMP1 ${OPM_INIT} PATH )
   get_filename_component( TMP2 ${TMP1} PATH )
   get_filename_component( OPM_MACROS_ROOT ${TMP2} PATH )

else()

   if (OPM_CMAKE_ROOT)
      list( APPEND CMAKE_MODULE_PATH "${OPM_CMAKE_ROOT}/cmake/Modules")
      include (OpmInit OPTIONAL RESULT_VARIABLE OPM_INIT)
      set( OPM_MACROS_ROOT ${OPM_CMAKE_ROOT} )
   endif()

   if (NOT OPM_INIT)
      message( "" )       
      message( " /-------------------------------------------------------------------------------\\")
      message( " |  Could not locate the opm build macros. The opm build macros                  |")
      message( " |  are in a separate repository - instructions to proceed:                      |")
      message( " |                                                                               |")
      message( " |    1. Clone the repository: git clone git@github.com:OPM/opm-cmake.git        |")
      message( " |                                                                               |")
      message( " |    2. Run cmake in the current project with -DOPM_CMAKE_ROOT=<path>/opm-cmake |")
      message( " |                                                                               |") 
      message( " \\-------------------------------------------------------------------------------/")
      message( "" )       
      message( FATAL_ERROR "Could not find OPM Macros")
   endif()

endif()

# not the same location as most of the other projects? this hook overrides
macro (dir_hook)
endmacro (dir_hook)

# project information is in dune.module. Read this file and set variables.
# we cannot generate dune.module since it is read by dunecontrol before
# the build starts, so it makes sense to keep the data there then.
include (OpmInit)

set(USE_OPENMP_DEFAULT ON)

# list of prerequisites for this particular project; this is in a
# separate file (in cmake/Modules sub-directory) because it is shared
# with the find module
include (${project}-prereqs)

# read the list of components from this file (in the project directory);
# it should set various lists with the names of the files to include
include (CMakeLists_files.cmake)

macro (config_hook)
	opm_need_version_of ("dune-common")
	opm_need_version_of ("dune-istl")
	list (APPEND ${project}_CONFIG_IMPL_VARS
		HAVE_LAPACK
		HAVE_UMFPACK
		HAVE_SUPERLU
		)
	# check unversioned CpGrid api change
	check_cxx_source_compiles("
	#include <dune/grid/CpGrid.hpp>

	int main()
	{
		Dune::CpGrid grid;
		grid.readEclipseFormat(\"foo\", 1.0e-3, false, false);
		return 0;
	}
	" HAVE_OLD_CPGRID_API)
	if (HAVE_OLD_CPGRID_API)
		add_definitions(-DHAVE_OLD_CPGRID_API)
	endif()
endmacro (config_hook)

macro (prereqs_hook)
endmacro (prereqs_hook)

macro (sources_hook)
endmacro (sources_hook)

macro (files_hook)
	find_dune_version ("dune" "istl")
	if((DUNE_ISTL_VERSION_MAJOR GREATER 2) OR
		(DUNE_ISTL_VERSION_MAJOR EQUAL 2 AND DUNE_ISTL_VERSION_MINOR GREATER 2))
		list (APPEND MAIN_SOURCE_FILES
			opm/elasticity/boundarygrid.cpp
			opm/elasticity/elasticity_preconditioners.cpp
			opm/elasticity/material.cpp
			opm/elasticity/materials.cpp
			opm/elasticity/matrixops.cpp
			opm/elasticity/meshcolorizer.cpp
			opm/elasticity/mpc.cpp
			)
		list(APPEND PROGRAM_SOURCE_FILES examples/upscale_elasticity.cpp)
		list (APPEND EXAMPLE_SOURCE_FILES examples/upscale_elasticity.cpp)
		list (APPEND TEST_DATA_FILES
			tests/input_data/reference_solutions/upscale_elasticity_mpc_EightCells.txt
			tests/input_data/reference_solutions/upscale_elasticity_mortar_EightCells.txt
		)
		list (APPEND PUBLIC_HEADER_FILES
			opm/elasticity/applier.hpp
			opm/elasticity/asmhandler.hpp
			opm/elasticity/asmhandler_impl.hpp
			opm/elasticity/boundarygrid.hh
			opm/elasticity/elasticity.hpp
			opm/elasticity/elasticity_impl.hpp
			opm/elasticity/elasticity_preconditioners.hpp
			opm/elasticity/elasticity_upscale.hpp
			opm/elasticity/elasticity_upscale_impl.hpp
			opm/elasticity/logutils.hpp
			opm/elasticity/material.hh
			opm/elasticity/materials.hh
			opm/elasticity/matrixops.hpp
			opm/elasticity/mortar_evaluator.hpp
			opm/elasticity/mortar_schur.hpp
			opm/elasticity/mortar_schur_precond.hpp
			opm/elasticity/mortar_utils.hpp
			opm/elasticity/mpc.hh
			opm/elasticity/shapefunctions.hpp
			opm/elasticity/uzawa_solver.hpp
			)
	endif()
endmacro (files_hook)

macro (fortran_hook)
	#  interface requires a variable to inspect, a value won't suffice
	set (UNCONDITIONALLY TRUE)
	set (${project}_FORTRAN_IF UNCONDITIONALLY)
endmacro (fortran_hook)

macro (tests_hook)
endmacro (tests_hook)

# all setup common to the OPM library modules is done here
include (OpmLibMain)

add_custom_target(check-commits
									COMMAND ${CMAKE_COMMAND}
													-DPROJECT_SOURCE_DIR=${PROJECT_SOURCE_DIR}
													-DCMAKE_BINARY_DIR=${CMAKE_BINARY_DIR}
													-P ${PROJECT_SOURCE_DIR}/cmake/Scripts/CheckCommits.cmake)

# setup extra tests (using helper binaries)
include (${CMAKE_CURRENT_SOURCE_DIR}/TestUpscalingBinaries.cmake)
