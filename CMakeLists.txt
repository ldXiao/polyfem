################################################################################
cmake_minimum_required(VERSION 3.1)
project(PolyFEM)
################################################################################

# cmake_policy(SET CMP0063 NEW)
# set(CMAKE_CXX_VISIBILITY_PRESET hidden)

# Detects whether this is a top-level project
if(CMAKE_SOURCE_DIR STREQUAL CMAKE_CURRENT_SOURCE_DIR)
	set(POLYFEM_TOPLEVEL_PROJECT ON)
else()
	set(POLYFEM_TOPLEVEL_PROJECT OFF)
endif()


if(INPUT_THIRD_PARTY_DIR)
	set(THIRD_PARTY_DIR ${CMAKE_CURRENT_SOURCE_DIR}/${INPUT_THIRD_PARTY_DIR}/)
else()
	set(THIRD_PARTY_DIR ${CMAKE_CURRENT_SOURCE_DIR}/3rdparty/)
endif()
list(APPEND CMAKE_MODULE_PATH ${CMAKE_CURRENT_SOURCE_DIR}/cmake)

if(${POLYFEM_TOPLEVEL_PROJECT})
	list(APPEND CMAKE_MODULE_PATH ${THIRD_PARTY_DIR}/Catch2/contrib)
endif()

# Color output
include(UseColors)

# Prepend function
include(PrependCurrentPath)

# Polyfem utils
include(PolyfemUtils)

# Extra warnings
include(Warnings)

# Use C++11/14
include(CXXFeatures)

# Sort projects inside the solution
set_property(GLOBAL PROPERTY USE_FOLDERS ON)

# Generate position independent code by default
set(CMAKE_POSITION_INDEPENDENT_CODE ON)

################################################################################

if(${CMAKE_SYSTEM_NAME} MATCHES "Windows")
	foreach(config ${CMAKE_CONFIGURATION_TYPES})
		string(TOUPPER ${config} config)
		string(REPLACE /MD /MT CMAKE_C_FLAGS_${config} "${CMAKE_C_FLAGS_${config}}")
		string(REPLACE /MD /MT CMAKE_CXX_FLAGS_${config} "${CMAKE_CXX_FLAGS_${config}}")
	endforeach()
endif()

################################################################################

# Polyfem options
# TODO: Other options should be renamed "POLYFEM_WITH_..." for consistency (and avoiding conflicts)
option(POLYFEM_WITH_SANITIZERS "Enable sanitizers in compilation targets"   OFF)
set(POLYFEM_SMALL_N  105 CACHE STRING "Maximum length for stack-allocated vectors (gradient + Hessian).")
set(POLYFEM_BIG_N   1000 CACHE STRING "Maximum length for stack-allocated vectors (gradient only).")

# Polyfem options for enabling/disabling optional libraries
option(ENABLE_CHOLMOD     "Enable Cholmod library" ON)
option(ENABLE_UMFPACK     "Enable UmfPack library" ON)
option(ENABLE_SUPERLU     "Enable SuperLU library" OFF)
option(ENABLE_MKL         "Enable MKL library"     OFF)
option(ENABLE_PARDISO     "Enable Pardiso library" ON)
option(ENABLE_MMG         "Enable MMG library"     OFF)
option(ENABLE_TBB         "Enable TBB"             ON)
option(ENABLE_HYPRE       "Enable hypre"           ON)
option(ENABLE_OPENCL      "Enable OpenCL"          OFF)

option(POLYFEM_LARGE_INDEX    "Build for large indices"       OFF)

option(POLYFEM_WITH_APPS      "Build the apps"          ON)
option(POLYFEM_WITH_MISC      "Build misc targes"       ON)

# Sanitizer options
option(SANITIZE_ADDRESS   "Sanitize Address"       OFF)
option(SANITIZE_MEMORY    "Sanitize Memory"        OFF)
option(SANITIZE_THREAD    "Sanitize Thread"        OFF)
option(SANITIZE_UNDEFINED "Sanitize Undefined"     OFF)

# Options for libigl modules
option(LIBIGL_USE_STATIC_LIBRARY          "Use libigl as static library" OFF)
option(LIBIGL_WITH_ANTTWEAKBAR            "Use AntTweakBar"              OFF)
option(LIBIGL_WITH_CGAL                   "Use CGAL"                     OFF)
option(LIBIGL_WITH_COMISO                 "Use CoMiso"                   OFF)
option(LIBIGL_WITH_CORK                   "Use Cork"                     OFF)
option(LIBIGL_WITH_EMBREE                 "Use Embree"                   OFF)
option(LIBIGL_WITH_LIM                    "Use LIM"                      OFF)
option(LIBIGL_WITH_MATLAB                 "Use Matlab"                   OFF)
option(LIBIGL_WITH_MOSEK                  "Use MOSEK"                    OFF)
option(LIBIGL_WITH_OPENGL                 "Use OpenGL"                   ON)
option(LIBIGL_WITH_OPENGL_GLFW            "Use GLFW"                     ON)
option(LIBIGL_WITH_OPENGL_GLFW_IMGUI      "Use ImGui"                    ON)
option(LIBIGL_WITH_PNG                    "Use PNG"                      OFF)
option(LIBIGL_WITH_PYTHON                 "Use Python"                   OFF)
option(LIBIGL_WITH_TETGEN                 "Use Tetgen"                   ON)
option(LIBIGL_WITH_TRIANGLE               "Use Triangle"                 ON)
option(LIBIGL_WITH_VIEWER                 "Use OpenGL viewer"            ON)
option(LIBIGL_WITH_XML                    "Use XML"                      OFF)

################################################################################

# Uncomment the following lines to force options which are already cached
# set(PREFER_LOCAL_LIBS   ON  CACHE BOOL "" FORCE)
# set(ENABLE_MKL          OFF CACHE BOOL "" FORCE)
# set(ENABLE_PARDISO      OFF CACHE BOOL "" FORCE)
# set(ENABLE_MMG          ON  CACHE BOOL "" FORCE)
# set(ENABLE_TBB          ON  CACHE BOOL "" FORCE)

# set(POLYFEM_WITH_SANITIZERS   OFF  CACHE BOOL "" FORCE)
# set(SANITIZE_ADDRESS          ON  CACHE BOOL "" FORCE)
# set(SANITIZE_MEMORY           ON  CACHE BOOL "" FORCE)
# set(SANITIZE_THREAD           OFF CACHE BOOL "" FORCE)
# set(SANITIZE_UNDEFINED        ON  CACHE BOOL "" FORCE)

# set(POLYFEM_SMALL_N 105 CACHE STRING "" FORCE)

# Somehow Pardiso 6 conflicts with libmetis that these guys depends on... Grr
if(ENABLE_PARDISO)
	set(ENABLE_CHOLMOD      OFF CACHE BOOL "" FORCE)
	set(ENABLE_UMFPACK      OFF CACHE BOOL "" FORCE)
	set(ENABLE_SUPERLU      OFF CACHE BOOL "" FORCE)
endif()

################################################################################
# Dependencies
################################################################################

# Sanitizers
if(POLYFEM_WITH_SANITIZERS)
	list(APPEND CMAKE_MODULE_PATH ${THIRD_PARTY_DIR}/sanitizers-cmake/cmake)
endif()

# Setup dependencies
include(PolyfemDependencies)

################################################################################
# Polyfem library
################################################################################

# add_library() can only be called without any source since CMake 3.11 ...
add_library(polyfem src/State.cpp)

if(${POLYFEM_LARGE_INDEX})
	target_compile_definitions(polyfem PUBLIC -DPOLYFEM_LARGE_INDEX)
endif()

# Must be call in the same "CMakeLists.txt" where add_library(polyfem ...) is called
polyfem_autogen(polyfem eigs.py auto_eigs)
polyfem_autogen(polyfem p_bases.py auto_p_bases)
polyfem_autogen(polyfem q_bases.py auto_q_bases)
polyfem_autogen(polyfem elasticity_rhs.py auto_elasticity_rhs)

# Public include directory for polyfem
target_include_directories(polyfem PUBLIC ${PROJECT_BINARY_DIR}/include)

set(MESH_PATH "${CMAKE_CURRENT_SOURCE_DIR}/tests/")
target_compile_definitions(polyfem PUBLIC -DPOLYFEM_MESH_PATH=\"${MESH_PATH}\")

# Dependencies
target_link_libraries(polyfem
	PUBLIC
		clipper::clipper
)

# Extra warnings
target_link_libraries(polyfem PRIVATE warnings::all)

# Use C++14
target_compile_features(polyfem PUBLIC ${CXX14_FEATURES})

# No limit yay
target_compile_definitions(polyfem PUBLIC -DEIGEN_STACK_ALLOCATION_LIMIT=0)

# 8MB
# target_compile_definitions(polyfem PUBLIC -DEIGEN_STACK_ALLOCATION_LIMIT=8388608)

# Max stack-size small vectors (for gradient and Hessian)
target_compile_definitions(polyfem PUBLIC -DPOLYFEM_SMALL_N=${POLYFEM_SMALL_N})
target_compile_definitions(polyfem PUBLIC -DPOLYFEM_BIG_N=${POLYFEM_BIG_N})

if("$ENV{CLUSTER}" STREQUAL "PRINCE")
	target_compile_definitions(polyfem PUBLIC -DPOLYFEM_ON_HPC)
endif()

if(POLYFEM_WITH_SANITIZERS)
	add_sanitizers(polyfem)
endif()

################################################################################
# Polyfem binary
################################################################################

# Main executable
if(POLYFEM_TOPLEVEL_PROJECT)
	add_executable(${PROJECT_NAME}_bin src/main.cpp)

	source_group("gui" FILES src/main.cpp)
	source_group("gui" FILES src/viewer/UIState.cpp)
	source_group("gui" FILES src/viewer/UIState.hpp)
	source_group("gui" FILES src/viewer/UIMenu.cpp)

	target_include_directories(${PROJECT_NAME}_bin PUBLIC src/viewer)
	target_compile_features(${PROJECT_NAME}_bin PUBLIC ${CXX14_FEATURES})

	target_link_libraries(${PROJECT_NAME}_bin
		PUBLIC
			polyfem
			warnings::all
			igl::opengl_glfw_imgui
			# igl::png
			# tinyfiledialogs::tinyfiledialogs
			CLI11::CLI11
	)

	if(POLYFEM_WITH_SANITIZERS)
		add_sanitizers(${PROJECT_NAME}_bin)
	endif()

	if(NOT (${CMAKE_VERSION} VERSION_LESS "3.6.0"))
		set_property(DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR} PROPERTY VS_STARTUP_PROJECT ${PROJECT_NAME}_bin)
	endif()
endif()

################################################################################
# Required libraries
################################################################################

# Eigen
# Uncomment to use the system's version of Eigen (e.g. to use Eigen 3.3)
# find_package(Eigen3 REQUIRED)
# target_link_libraries(polyfem PUBLIC Eigen3::Eigen)

# TBB
if(ENABLE_TBB)
	polyfem_download_tbb()
	set(TBB_BUILD_STATIC ON CACHE BOOL " " FORCE)
	set(TBB_BUILD_SHARED OFF CACHE BOOL " " FORCE)
	set(TBB_BUILD_TBBMALLOC OFF CACHE BOOL " " FORCE)
	set(TBB_BUILD_TBBMALLOC_PROXY OFF CACHE BOOL " " FORCE)
	set(TBB_BUILD_TESTS OFF CACHE BOOL " " FORCE)

	add_subdirectory(${THIRD_PARTY_DIR}/tbb tbb)
	set_property(TARGET tbb_static tbb_def_files PROPERTY FOLDER "dependencies")

	target_compile_definitions(tbb_static PUBLIC -DUSE_TBB)
	target_include_directories(polyfem SYSTEM PUBLIC ${THIRD_PARTY_DIR}/tbb/include)
	target_link_libraries(polyfem PUBLIC tbb_static)
endif()

# libigl
polyfem_download_libigl()
find_package(LIBIGL REQUIRED)
target_link_libraries(polyfem PUBLIC igl::core igl::triangle igl::tetgen)


# Geogram
polyfem_download_geogram()
include(geogram)
target_link_libraries(polyfem PUBLIC geogram)

#spdlog
target_link_libraries(polyfem PUBLIC spdlog::spdlog)

# CppNumericalSolvers
polyfem_download_CppNumericalSolvers()
add_library(cppoptlib INTERFACE)
target_include_directories(cppoptlib SYSTEM INTERFACE ${THIRD_PARTY_DIR}/CppNumericalSolvers/include)
target_link_libraries(polyfem PUBLIC cppoptlib)

# Spectra
polyfem_download_spectra()
add_library(spectra INTERFACE)
target_include_directories(spectra SYSTEM INTERFACE ${THIRD_PARTY_DIR}/spectra/include)
target_link_libraries(polyfem PUBLIC spectra)

# Json
polyfem_download_json()
add_library(json INTERFACE)
target_include_directories(json SYSTEM INTERFACE ${THIRD_PARTY_DIR}/json/include)
target_link_libraries(polyfem PUBLIC json)

# TinyExpr library
if(NOT TARGET tinyexpr::tinyexpr)
	polyfem_download_tinyexpr()
    add_library(tinyexpr_tinyexpr ${THIRD_PARTY_DIR}/tinyexpr/tinyexpr.c)
    target_include_directories(tinyexpr_tinyexpr SYSTEM PUBLIC ${THIRD_PARTY_DIR}/tinyexpr)
    add_library(tinyexpr::tinyexpr ALIAS tinyexpr_tinyexpr)
endif()
target_link_libraries(polyfem PUBLIC tinyexpr::tinyexpr)

# Catch2
if(${POLYFEM_TOPLEVEL_PROJECT})
	polyfem_download_catch2()
	add_library(catch INTERFACE)
	target_include_directories(catch SYSTEM INTERFACE ${THIRD_PARTY_DIR}/Catch2/single_include/catch2)
endif()

# Hypre
if(ENABLE_HYPRE)
	polyfem_download_hypre()
	include(hypre)
	target_link_libraries(polyfem PUBLIC HYPRE)
	target_compile_definitions(polyfem PUBLIC -DPOLYFEM_WITH_HYPRE)
endif()

################################################################################
# Optional libraries
################################################################################

# MMG wrapper
if(ENABLE_MMG)
	include(ExternalProject)
	set(MMG_DIR "${THIRD_PARTY_DIR}/mmg")
	ExternalProject_Add(MMG_Project
		PREFIX "${MMG_DIR}"
		GIT_REPOSITORY https://github.com/MmgTools/mmg.git
		GIT_TAG "v5.3.8"

		UPDATE_COMMAND ""
		PATCH_COMMAND ""

		CMAKE_ARGS
			-DCMAKE_PREFIX_PATH=${MMG_DIR}/bin
			-DBUILD_SHARED_LIBS=OFF
			"-DCMAKE_INSTALL_PREFIX=${MMG_DIR}/"

		TEST_COMMAND "")

	set(MMG_PATH "${THIRD_PARTY_DIR}/mmg/bin/mmg3d_O3")
	target_compile_definitions(polyfem PUBLIC -DPOLYFEM_MMG_PATH=\"${MMG_PATH}\")
	target_compile_definitions(polyfem PUBLIC -DPOLYFEM_WITH_MMG)

	# Boost
	find_package(Boost 1.61 QUIET REQUIRED COMPONENTS filesystem system)
	target_include_directories(polyfem PUBLIC ${Boost_INCLUDE_DIRS})
	target_link_libraries(polyfem PUBLIC ${Boost_LIBRARIES})
endif()

# Cholmod solver
if(ENABLE_CHOLMOD)
	find_package(Cholmod)
	if(CHOLMOD_FOUND)
		target_include_directories(polyfem PUBLIC ${CHOLMOD_INCLUDES})
		target_link_libraries(polyfem PUBLIC ${CHOLMOD_LIBRARIES})
		target_compile_definitions(polyfem PUBLIC -DPOLYFEM_WITH_CHOLMOD)
	endif()
endif()

# MKL library
if(ENABLE_MKL)
	find_package(MKL)
	if(MKL_FOUND)
		target_include_directories(polyfem PUBLIC ${MKL_INCLUDE_DIR})
		target_link_libraries(polyfem PUBLIC ${MKL_LIBRARIES})
		target_compile_definitions(polyfem PUBLIC -DPOLYFEM_WITH_MKL)
	endif()
endif()

# Pardiso solver
if(ENABLE_PARDISO AND NOT (ENABLE_MKL AND MKL_FOUND))
	find_package(Pardiso)
	if(PARDISO_FOUND)
		find_package(LAPACK)
		if(LAPACK_FOUND)
			target_link_libraries(polyfem PUBLIC ${LAPACK_LIBRARIES})
		else()
			message(FATAL_ERROR "unable to find lapack")
		endif()

		# find_package(OpenMP)
		# if( OpenMP_CXX_FOUND )
		# 	target_link_libraries(polyfem PUBLIC ${OpenMP_CXX_LIBRARIES})
		# 	# target_compile_definitions(polyfem PUBLIC ${OpenMP_CXX_FLAGS})
		# else()
		# 	message(FATAL_ERROR "unable to find omp")
		# endif()

		target_link_libraries(polyfem PUBLIC ${PARDISO_LIBRARIES})
		target_compile_definitions(polyfem PUBLIC -DPOLYFEM_WITH_PARDISO)
	else()
		message(WARNING "Pardiso not found, solver will not be available.")
	endif()
endif()

# UmfPack solver
if(ENABLE_UMFPACK)
	find_package(Umfpack)
	if(UMFPACK_FOUND)
		target_include_directories(polyfem PUBLIC ${UMFPACK_INCLUDES})
		target_link_libraries(polyfem PUBLIC ${UMFPACK_LIBRARIES})
		target_compile_definitions(polyfem PUBLIC -DPOLYFEM_WITH_UMFPACK)
	endif()
endif()

# SuperLU solver
if(ENABLE_SUPERLU AND NOT (ENABLE_MKL AND MKL_FOUND))
	find_package(SuperLU)
	if(SUPERLU_FOUND)
		target_include_directories(polyfem PUBLIC ${SUPERLU_INCLUDES})
		target_link_libraries(polyfem PUBLIC ${SUPERLU_LIBRARIES})
		target_compile_definitions(polyfem PUBLIC ${SUPERLU_DEFINES})
		target_compile_definitions(polyfem PUBLIC -DPOLYFEM_WITH_SUPERLU)
	endif()
endif()


# if(ENABLE_OPENCL)
# 	find_package(OpenCL)
# 	if(OpenCL_FOUND)
# 		polyfem_download_rbf()
# 		target_include_directories(polyfem PUBLIC ${OpenCL_INCLUDE_DIRS})
# 		target_link_libraries(polyfem PUBLIC OpenCL::OpenCL)
# 		target_compile_definitions(polyfem PUBLIC "-DPOLYFEM_OPENCL -DRBF_PUM_LIB -DRBF_SCRIPT_ROOT=\"${THIRD_PARTY_DIR}rbf/\"")

# 		target_sources(polyfem PRIVATE ${THIRD_PARTY_DIR}/rbf/cl.hpp ${THIRD_PARTY_DIR}/rbf/rbf_interpolate.hpp ${THIRD_PARTY_DIR}/rbf/rbf_interpolate.cpp)
# 		target_include_directories(polyfem PUBLIC ${THIRD_PARTY_DIR}/rbf)

# 		add_executable(rbf_test ${THIRD_PARTY_DIR}/rbf/cl.hpp ${THIRD_PARTY_DIR}/rbf/rbf_interpolate.hpp ${THIRD_PARTY_DIR}/rbf/rbf_interpolate.cpp)
# 		target_compile_definitions(rbf_test PUBLIC "-DRBF_SCRIPT_ROOT=\"${THIRD_PARTY_DIR}rbf/\"")
# 		target_include_directories(rbf_test PUBLIC ${OpenCL_INCLUDE_DIRS})
# 		target_link_libraries(rbf_test PUBLIC OpenCL::OpenCL)
# 	else()
# 		MESSAGE(WANING "OpenCL not found!")
# 	endif()
# endif()

################################################################################
# Clutter management
################################################################################

# set_target_properties(igl_triangle PROPERTIES FOLDER 3rdparty/igl)

# if(TARGET igl_png)
# 	set_target_properties(igl_png PROPERTIES FOLDER 3rdparty/igl)
# endif()
# if(TARGET igl_opengl)
# 	set_target_properties(igl_opengl PROPERTIES FOLDER 3rdparty/igl)
# 	set_target_properties(igl_opengl_glfw_imgui PROPERTIES FOLDER 3rdparty/igl)
# 	set_target_properties(igl_opengl_glfw PROPERTIES FOLDER 3rdparty/igl)
# endif()

# set_target_properties(igl PROPERTIES FOLDER 3rdparty/igl)
# set_target_properties(igl_tetgen PROPERTIES FOLDER 3rdparty/igl)

if(TARGET igl_stb_image)
	set_target_properties(igl_stb_image PROPERTIES FOLDER 3rdparty/igl)
endif()

# if(TARGET igl_opengl)
# 	set_target_properties(glad PROPERTIES FOLDER 3rdparty/other)
# endif()
# # set_target_properties(tinyfiledialogs_tinyfiledialogs PROPERTIES FOLDER 3rdparty/other)
# if(TARGET igl::opengl_glfw_imgui)
# 	set_target_properties(imgui PROPERTIES FOLDER 3rdparty/other)
# endif()
set_target_properties(tetgen PROPERTIES FOLDER 3rdparty/other)
set_target_properties(triangle PROPERTIES FOLDER 3rdparty/other)
# set_target_properties(uninstall_geo PROPERTIES FOLDER 3rdparty/other)
set_target_properties(geogram PROPERTIES FOLDER 3rdparty/other)
set_target_properties(clipper_clipper PROPERTIES FOLDER 3rdparty/other)

################################################################################
# Subdirectories
################################################################################

# Other sources
add_subdirectory(src)

# Compile extras only if this is a top-level project
if(POLYFEM_TOPLEVEL_PROJECT)
	# Auxiliary binaries
	if(POLYFEM_WITH_APPS)
		add_subdirectory(apps)
	endif()

	if(POLYFEM_WITH_MISC)
		add_subdirectory(misc)
	endif()

	# Unit tests
	include(CTest)
	enable_testing()
	add_subdirectory(tests)
endif()
