cmake_minimum_required( VERSION 2.4 )
project(ViZDoom)

option(BUILD_PYTHON "Build ViZDoom API Python binding." OFF)
option(BUILD_JAVA "Build ViZDoom API Java binding." OFF)
option(BUILD_STATIC "Build ViZDoom API Java binding." ON)

if(COMMAND cmake_policy)
    cmake_policy(SET CMP0003 NEW)
endif(COMMAND cmake_policy)

set(CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} "${CMAKE_SOURCE_DIR}" "${PROJECT_SOURCE_DIR}")

# System dependent settings    
if( UNIX )
	set(Boost_USE_STATIC_LIBS OFF)
	set(Boost_USE_MULTITHREADED ON)
	set(Boost_USE_STATIC_RUNTIME OFF)
endif()

if( WIN32 )
	set(Boost_USE_STATIC_LIBS ON)
	set(Boost_USE_MULTITHREADED ON)
	set(Boost_USE_STATIC_RUNTIME ON)

	macro(get_WIN32_WINNT version)
		if (WIN32 AND CMAKE_SYSTEM_VERSION)
			set(ver ${CMAKE_SYSTEM_VERSION})
			string(REPLACE "." "" ver ${ver})
			string(REGEX REPLACE "([0-9])" "0\\1" ver ${ver})

			set(${version} "0x${ver}")
		endif()
	endmacro()

	get_WIN32_WINNT(ver)
	add_definitions(-D_WIN32_WINNT=${ver})
endif()

# Compiler dependent settings     
if(CMAKE_COMPILER_IS_GNUCXX)
    add_definitions("-fPIC")
endif()  

if( MSVC )
	string(REPLACE "/MD " "/MT " CMAKE_CXX_FLAGS_RELEASE ${CMAKE_CXX_FLAGS_RELEASE})
	string(REPLACE "/MD " "/MT " CMAKE_CXX_FLAGS_MINSIZEREL ${CMAKE_CXX_FLAGS_MINSIZEREL})
	string(REPLACE "/MD " "/MT " CMAKE_CXX_FLAGS_RELWITHDEBINFO ${CMAKE_CXX_FLAGS_RELWITHDEBINFO})
	string(REPLACE "/MDd " "/MTd " CMAKE_CXX_FLAGS_DEBUG ${CMAKE_CXX_FLAGS_DEBUG})
endif()

set(VIZDOOM_OUTPUT_DIR ${CMAKE_CURRENT_SOURCE_DIR}/bin)

# Targets
add_subdirectory(vizdoom_src)

find_package(Boost COMPONENTS filesystem thread system date_time chrono regex iostreams REQUIRED)
find_package(Threads REQUIRED)

set(VIZDOOM_INCLUDE_DIR vizdoom_api_src)

include_directories(${VIZDOOM_INCLUDE_DIR} ${Boost_INCLUDE_DIR})

set(VIZDOOM_LIBS
    ${Boost_LIBRARIES}  
    ${CMAKE_THREAD_LIBS_INIT} )
	
if( UNIX)
  if (NOT (${CMAKE_SYSTEM_NAME} MATCHES "Darwin"))
    set(VIZDOOM_LIBS ${VIZDOOM_LIBS} rt)
   endif()
endif()	
    
set(VIZDOOM_API_SOURCES
    vizdoom_api_src/ViZDoomUtilities.cpp
	vizdoom_api_src/ViZDoomController.cpp
	vizdoom_api_src/ViZDoomGame.cpp)

add_library(vizdoomstatic STATIC ${VIZDOOM_API_SOURCES})
target_link_libraries (vizdoomstatic ${VIZDOOM_LIBS})

add_library(vizdoomshared SHARED ${VIZDOOM_API_SOURCES})
target_link_libraries (vizdoomshared ${VIZDOOM_LIBS})

set_target_properties(vizdoomstatic vizdoomshared
    PROPERTIES
    ARCHIVE_OUTPUT_DIRECTORY ${VIZDOOM_OUTPUT_DIR}
	ARCHIVE_OUTPUT_DIRECTORY_DEBUG ${VIZDOOM_OUTPUT_DIR}
	ARCHIVE_OUTPUT_DIRECTORY_RELEASE ${VIZDOOM_OUTPUT_DIR}
    LIBRARY_OUTPUT_DIRECTORY ${VIZDOOM_OUTPUT_DIR}
	LIBRARY_OUTPUT_DIRECTORY_DEBUG ${VIZDOOM_OUTPUT_DIR}
	LIBRARY_OUTPUT_DIRECTORY_RELEASE ${VIZDOOM_OUTPUT_DIR}
    RUNTIME_OUTPUT_DIRECTORY ${VIZDOOM_OUTPUT_DIR}
	RUNTIME_OUTPUT_DIRECTORY_DEBUG ${VIZDOOM_OUTPUT_DIR}
	RUNTIME_OUTPUT_DIRECTORY_RELEASE ${VIZDOOM_OUTPUT_DIR}
    OUTPUT_NAME vizdoom)

#Python binding	
if(BUILD_PYTHON)

    #set(Boost_USE_DEBUG_PYTHON OFF)
    
    find_package(PythonInterp 2.7 REQUIRED)
    find_package(PythonLibs 2.7 REQUIRED)
	find_package(NumPy REQUIRED)
    find_package(Boost COMPONENTS python REQUIRED)

    set(VIZDOOM_PYTHON_INCLUDE_DIR ${VIZDOOM_INCLUDE_DIR} vizdoom_api_bindings/python)

    set(VIZDOOM_PYTHON_LIBS 
        ${VIZDOOM_LIBS}
		${Boost_PYTHON_LIBRARY}
        ${PYTHON_LIBRARIES}
		${NUMPY_LIBRARIES})
        
    include_directories(${VIZDOOM_PYTHON_INCLUDE_DIR}
                        ${Boost_INCLUDE_DIR} 
                        ${PYTHON_INCLUDE_DIR}
						${NUMPY_INCLUDES})  

    set(VIZDOOM_PYTHON_SOURCES
	    vizdoom_api_bindings/python/ViZDoomPythonBinding.cpp
        vizdoom_api_bindings/python/ViZDoomGamePython.cpp)
	
	if(WIN32)	
		add_definitions(-DBOOST_PYTHON_STATIC_LIB)
	endif()

    python_add_module(vizdoompython ${VIZDOOM_PYTHON_SOURCES})
    target_link_libraries (vizdoompython ${VIZDOOM_PYTHON_LIBS} vizdoomstatic)

    set_target_properties(vizdoompython
        PROPERTIES
        ARCHIVE_OUTPUT_DIRECTORY ${VIZDOOM_OUTPUT_DIR}/python
		ARCHIVE_OUTPUT_DIRECTORY_DEBUG ${VIZDOOM_OUTPUT_DIR}/python
		ARCHIVE_OUTPUT_DIRECTORY_RELEASE ${VIZDOOM_OUTPUT_DIR}/python
		LIBRARY_OUTPUT_DIRECTORY ${VIZDOOM_OUTPUT_DIR}/python
		LIBRARY_OUTPUT_DIRECTORY_DEBUG ${VIZDOOM_OUTPUT_DIR}/python
		LIBRARY_OUTPUT_DIRECTORY_RELEASE ${VIZDOOM_OUTPUT_DIR}/python
		RUNTIME_OUTPUT_DIRECTORY ${VIZDOOM_OUTPUT_DIR}/python
		RUNTIME_OUTPUT_DIRECTORY_DEBUG ${VIZDOOM_OUTPUT_DIR}/python
		RUNTIME_OUTPUT_DIRECTORY_RELEASE ${VIZDOOM_OUTPUT_DIR}/python
        OUTPUT_NAME vizdoom)
    
    if(UNIX)
        ADD_CUSTOM_COMMAND( 
            TARGET vizdoompython
            COMMAND ln -sf ${VIZDOOM_OUTPUT_DIR}/python/vizdoom.so ${PROJECT_SOURCE_DIR}/examples/python/vizdoom.so
            COMMAND ln -sf ${PROJECT_SOURCE_DIR}/examples/config/bots.cfg ${PROJECT_SOURCE_DIR}/examples/python/bots.cfg
            COMMENT "Adding symlinks to vizdoom.so and bots.cfg in examples/python."
        )
    endif()
        
endif()

#Java binding
if(BUILD_JAVA)

    if (NOT DEFINED ENV{JAVA_HOME})
        message(FATAL_ERROR "JAVA_HOME environment variable is not set.")
    endif()
        
    find_package(Java REQUIRED)
    find_package(JNI REQUIRED)
	
    include(UseJava)
    set(VIZDOOM_JAVA_INCLUDE_DIR ${VIZDOOM_INCLUDE_DIR} vizdoom_api_bindings/java)
    
    set(VIZDOOM_JAVA_LIBS 
        ${VIZDOOM_LIBS}  
        ${Java_LIBRARIES}
        ${JNI_LIBRARIES})
		
	if( UNIX )
		set(JNI_INCLUDE_DIR ${JNI_INCLUDE_DIR}
			${_JAVA_HOME}/include	
			${_JAVA_HOME}/include/linux)
	endif()

	if( WIN32 )
		set( JNI_INCLUDE_DIR ${JNI_INCLUDE_DIR}
			${_JAVA_HOME}/include
			${_JAVA_HOME}/include/win32 )
	endif()	
        
    include_directories(${VIZDOOM_JAVA_INCLUDE_DIR} 
                        ${Boost_INCLUDE_DIR}
                        ${Java_INCLUDE_DIRS} 
                        ${JNI_INCLUDE_DIR} )  

    set(VIZDOOM_JAVA_SOURCES vizdoom_api_bindings/java/ViZDoomGameJava.cpp)


    add_library(vizdoomjava SHARED ${VIZDOOM_JAVA_SOURCES})
    target_link_libraries (vizdoomjava ${VIZDOOM_JAVA_LIBS} vizdoomstatic)


    add_jar(vizdoomjavajar
	    ./vizdoom_api_bindings/java/enums/Button.java
	    ./vizdoom_api_bindings/java/enums/GameVariable.java
	    ./vizdoom_api_bindings/java/enums/Mode.java
	    ./vizdoom_api_bindings/java/enums/ScreenFormat.java
	    ./vizdoom_api_bindings/java/enums/ScreenResolution.java
	    ./vizdoom_api_bindings/java/errors/ViZDoomMismatchedVersionException.java
	    ./vizdoom_api_bindings/java/errors/ViZDoomErrorException.java
        ./vizdoom_api_bindings/java/errors/ViZDoomIsNotRunningException.java
	    ./vizdoom_api_bindings/java/errors/ViZDoomUnexpectedExitException.java
	    ./vizdoom_api_bindings/java/errors/MessageQueueException.java
	    ./vizdoom_api_bindings/java/errors/FileDoesNotExistException.java
	    ./vizdoom_api_bindings/java/errors/SharedMemoryException.java
	    ./vizdoom_api_bindings/java/GameState.java
        ./vizdoom_api_bindings/java/DoomGame.java 
  	    OUTPUT_DIR ${VIZDOOM_OUTPUT_DIR}/java 
		OUTPUT_NAME vizdoom)

    set_target_properties(vizdoomjava
        PROPERTIES
        ARCHIVE_OUTPUT_DIRECTORY ${VIZDOOM_OUTPUT_DIR}/java
		ARCHIVE_OUTPUT_DIRECTORY_DEBUG ${VIZDOOM_OUTPUT_DIR}/java
		ARCHIVE_OUTPUT_DIRECTORY_RELEASE ${VIZDOOM_OUTPUT_DIR}/java
		LIBRARY_OUTPUT_DIRECTORY ${VIZDOOM_OUTPUT_DIR}/java
		LIBRARY_OUTPUT_DIRECTORY_DEBUG ${VIZDOOM_OUTPUT_DIR}/java
		LIBRARY_OUTPUT_DIRECTORY_RELEASE ${VIZDOOM_OUTPUT_DIR}/java
		RUNTIME_OUTPUT_DIRECTORY ${VIZDOOM_OUTPUT_DIR}/java
		RUNTIME_OUTPUT_DIRECTORY_DEBUG ${VIZDOOM_OUTPUT_DIR}/java
		RUNTIME_OUTPUT_DIRECTORY_RELEASE ${VIZDOOM_OUTPUT_DIR}/java
        OUTPUT_NAME vizdoom)
        	
endif()

#if( WIN32 )
#	if(NOT EXISTS "${PROJECT_SOURCE_DIR}/scenarios/freedoom.zip")
#		message("-- Downloading Freedoom WADs")
#		message("-- Original Doom WADs can not be provided due to licences")
#
#		file( DOWNLOAD https://github.com/freedoom/freedoom/releases/download/v0.10.1/freedoom-0.10.1.zip ${PROJECT_SOURCE_DIR}/scenarios/freedoom.zip SHOW_PROGRESS )
#	endif()
#endif()

if(UNIX)
    add_custom_target( freedoom2 ALL
        COMMAND ./download_freedoom.sh
    )
endif()
