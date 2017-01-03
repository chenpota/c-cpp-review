GitHub: https://github.com/chenpota/c-cpp/tree/master/cpp-plugin

# File structure

    cpp-plugin/
    ├── app/
    │   ├── CMakeLists.txt
    │   └── src/
    │       └── main.cpp
    ├── CMakeLists.txt
    ├── libmyplugin/
    │   ├── CMakeLists.txt
    │   ├── include/
    │   │   └── MyPlugin.hpp
    │   └── src/
    │       └── MyPlugin.cpp
    └── libplugin/
        ├── CMakeLists.txt
        ├── include/
        │   └── Plugin.hpp
        └── src/
            └── Plugin.cpp

## cpp-plugin/CMakeLists.txt
```cmake
cmake_minimum_required(VERSION 2.8)
project(cpp-plugin)

add_subdirectory(app)

add_subdirectory(libplugin)

add_subdirectory(libmyplugin)
```

## cpp-plugin/app/CMakeLists.txt

Remember to link *libdl*

```cmake
cmake_minimum_required(VERSION 2.8)
project(app)

add_definitions(
    -DMYPLUGIN_PATH="${CMAKE_BINARY_DIR}/libmyplugin/libmyplugin.so")

set(SRC
    ${CMAKE_CURRENT_SOURCE_DIR}/src/main.cpp)

include_directories(
    ${CMAKE_CURRENT_SOURCE_DIR}/../libplugin/include)

set(LIB
    plugin
    dl)

add_executable(
    app
    ${SRC}
)

target_link_libraries(
    app
    ${LIB})
```

## cpp-plugin/app/src/main.cpp

### dlopen()

Load shared object dynamically.

### dlsym()

Take a handle of a dynamic loaded shared object returned by dlopen().

### dlerror()

Returns NULL if no errors have occurred.

Return a human-readable, null-terminated string describing the most recent error that occurred from a call to one of the functions in the dlopen API since the last call to dlerror().

### dlclose()

Decrease the reference count on the dynamically loaded shared object referred to by handle.

If the reference count drops to zero, then the object is unloaded.

```cpp
#include <dlfcn.h>
#include <stdlib.h>

#include <iostream>

#include <Plugin.hpp>

using libplugin::Plugin;

using std::cerr;
using std::endl;

typedef
    Plugin * (*create_plugin_t)();

int main()
{
    void *dlHandle = dlopen(MYPLUGIN_PATH, RTLD_LAZY);

    if(dlHandle == NULL)
    {
        cerr << "dlopen error: " << dlerror() << endl;
        exit(1);
    }

    create_plugin_t create_plugin = (create_plugin_t)dlsym(dlHandle, "create");

    if(create_plugin == NULL)
    {
        cerr << "dlsym error: " << dlerror() << endl;
        exit(1);
    }

    Plugin *plugin = create_plugin();

    plugin->execute();

    const int result = dlclose(dlHandle);

    if(result != 0)
    {
        cerr << "dlclose error: " << dlerror() << endl;
        exit(1);
    }

    return 0;
}
```

## cpp-plugin/libplugin/CMakeLists.txt
```cmake
cmake_minimum_required(VERSION 2.8)
project(libplugin)

set(SRC
    ${CMAKE_CURRENT_SOURCE_DIR}/src/Plugin.cpp)

include_directories(
    ${CMAKE_CURRENT_SOURCE_DIR}/include)

add_library(
    plugin
    SHARED
    ${SRC})
```

```cpp
// cpp-plugin/libplugin/include/Plugin.hpp

#ifndef _PLUGIN_HPP_
#define _PLUGIN_HPP_

namespace libplugin
{
    class Plugin
    {
    public:
        Plugin();

        virtual ~Plugin();

        void
            execute();

    private:
        virtual void
            do_execute() = 0;
    };
}
#endif
```

## cpp-plugin/libplugin/src/Plugin.cpp
```cpp
#include <Plugin.hpp>

namespace libplugin
{
    Plugin::Plugin()
    {
    }

    Plugin::~Plugin()
    {
    }

    void
        Plugin::execute()
    {
        this->do_execute();
    }
}
```

## cpp-plugin/libmyplugin/CMakeLists.txt
```cmake
cmake_minimum_required(VERSION 2.8)
project(libplugin)

set(SRC
    ${CMAKE_CURRENT_SOURCE_DIR}/src/MyPlugin.cpp)

include_directories(
    ${CMAKE_CURRENT_SOURCE_DIR}/../libplugin/include
    ${CMAKE_CURRENT_SOURCE_DIR}/include)

set(LIB
    plugin)

add_library(
    myplugin
    SHARED
    ${SRC})

target_link_libraries(
    myplugin
    ${LIB})
```

## cpp-plugin/libmyplugin/include/MyPlugin.hpp
```cpp
#ifndef _MY_PLUGIN_HPP_
#define _MY_PLUGIN_HPP_

#include <Plugin.hpp>

namespace myplugin
{
    class MyPlugin: public libplugin::Plugin
    {
    public:
        MyPlugin();

        virtual ~MyPlugin();

        virtual void
            do_execute();
    };
}

extern "C" {
    libplugin::Plugin *
        create();
}

#endif
```

## cpp-plugin/libmyplugin/src/MyPlugin.cpp
```cpp
#include <iostream>

#include <MyPlugin.hpp>

namespace myplugin
{
    using std::cout;
    using std::endl;

    MyPlugin::MyPlugin()
        : Plugin()
    {
    }

    MyPlugin::~MyPlugin()
    {
    }

    void
        MyPlugin::do_execute()
    {
        cout << "MyPlugin::do_execute()" << endl;
    }
}

extern "C" {
    libplugin::Plugin *
        create()
    {
        return new myplugin::MyPlugin();
    }
}
```

# Execute

    root@3f8bf6ba4ec6:~/cpp-plugin# mkdir build; cd build
    root@3f8bf6ba4ec6:~/cpp-plugin/build# cmake ..
    -- The C compiler identification is GNU 5.4.0
    -- The CXX compiler identification is GNU 5.4.0
    -- Check for working C compiler: /usr/bin/cc
    -- Check for working C compiler: /usr/bin/cc -- works
    -- Detecting C compiler ABI info
    -- Detecting C compiler ABI info - done
    -- Detecting C compile features
    -- Detecting C compile features - done
    -- Check for working CXX compiler: /usr/bin/c++
    -- Check for working CXX compiler: /usr/bin/c++ -- works
    -- Detecting CXX compiler ABI info
    -- Detecting CXX compiler ABI info - done
    -- Detecting CXX compile features
    -- Detecting CXX compile features - done
    -- Configuring done
    -- Generating done
    -- Build files have been written to: /root/cpp-plugin/build
    root@3f8bf6ba4ec6:~/cpp-plugin/build# make
    Scanning dependencies of target plugin
    [ 16%] Building CXX object libplugin/CMakeFiles/plugin.dir/src/Plugin.cpp.o
    [ 33%] Linking CXX shared library libplugin.so
    [ 33%] Built target plugin
    Scanning dependencies of target app
    [ 50%] Building CXX object app/CMakeFiles/app.dir/src/main.cpp.o
    [ 66%] Linking CXX executable app
    [ 66%] Built target app
    Scanning dependencies of target myplugin
    [ 83%] Building CXX object libmyplugin/CMakeFiles/myplugin.dir/src/MyPlugin.cpp.o
    [100%] Linking CXX shared library libmyplugin.so
    [100%] Built target myplugin
    root@3f8bf6ba4ec6:~/cpp-plugin/build# ./app/app 
    MyPlugin::do_execute()
    root@3f8bf6ba4ec6:~/cpp-plugin/build# 

# Reference
[1] http://www.tldp.org/HOWTO/html_single/C++-dlopen/
