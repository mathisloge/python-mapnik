cmake_minimum_required(VERSION 3.15)
project(python-mapnik)

find_package(PkgConfig REQUIRED) 
find_package(Python COMPONENTS Interpreter Development REQUIRED)
find_package(Boost COMPONENTS log python${Python_VERSION_MAJOR}${Python_VERSION_MINOR} REQUIRED)
find_package(mapnik CONFIG REQUIRED)
find_path(MAPBOX_GEOMETRY_INCLUDE_DIRS "mapbox/geometry.hpp" REQUIRED HINTS "${MAPNIK_INCLUDE_DIR}")

set(PYTHON_MAPNIK_COMPILE_DEFS "")
set(PYTHON_MAPNIK_OPTIONAL_LIBS "")
set(PYTHON_MAPNIK_OPTIONAL_INC "")

pkg_check_modules(PY_CAIRO IMPORTED_TARGET pycairo)
if(PY_CAIRO_FOUND)
    list(APPEND PYTHON_MAPNIK_COMPILE_DEFS HAVE_PYCAIRO)
    list(APPEND PYTHON_MAPNIK_OPTIONAL_LIBS PkgConfig::PY_CAIRO)
    message(STATUS "Using pkg-config pycairo")
else()
    execute_process(COMMAND ${Python_EXECUTABLE} -c "import os; import cairo; print(os.path.dirname(cairo.__file__))" OUTPUT_VARIABLE PYTHON_PY_CAIRO OUTPUT_STRIP_TRAILING_WHITESPACE)
    if(${PYTHON_PY_CAIRO})
        message(STATUS "Using python pycairo from ${PYTHON_PY_CAIRO}")
        file(TO_CMAKE_PATH "${PYTHON_PY_CAIRO}" PYTHON_PY_CAIRO)
        list(APPEND PYTHON_MAPNIK_COMPILE_DEFS HAVE_PYCAIRO)
        list(APPEND PYTHON_MAPNIK_OPTIONAL_INC "${PYTHON_PY_CAIRO}/include")
    endif()
endif()

# currently the cmake files are not installed, when installing proj via apt-get. So search via pkg-config
if(NOT PROJ_FOUND)
    message(STATUS "PROJ not found via FindPROJ. Searching via pkg-config...")
    pkg_check_modules(PROJ REQUIRED IMPORTED_TARGET proj)
endif()

Python_add_library(python-mapnik MODULE 
    src/boost_std_shared_shim.hpp
    src/mapnik_color.cpp
    src/mapnik_coord.cpp
    src/mapnik_datasource_cache.cpp
    src/mapnik_datasource.cpp
    src/mapnik_enumeration_wrapper_converter.hpp
    src/mapnik_enumeration.hpp
    src/mapnik_envelope.cpp
    src/mapnik_expression.cpp
    src/mapnik_feature.cpp
    src/mapnik_featureset.cpp
    src/mapnik_font_engine.cpp
    src/mapnik_fontset.cpp
    src/mapnik_gamma_method.cpp
    src/mapnik_geometry.cpp
    src/mapnik_grid_view.cpp
    src/mapnik_grid.cpp
    src/mapnik_image_view.cpp
    src/mapnik_image.cpp
    src/mapnik_label_collision_detector.cpp
    src/mapnik_layer.cpp
    src/mapnik_logger.cpp
    src/mapnik_map.cpp
    src/mapnik_palette.cpp
    src/mapnik_parameters.cpp
    src/mapnik_proj_transform.cpp
    src/mapnik_projection.cpp
    src/mapnik_python.cpp
    src/mapnik_query.cpp
    src/mapnik_raster_colorizer.cpp
    src/mapnik_rule.cpp
    src/mapnik_scaling_method.cpp
    src/mapnik_style.cpp
    src/mapnik_svg.hpp
    src/mapnik_symbolizer.cpp
    src/mapnik_threads.hpp
    src/mapnik_value_converter.hpp
    src/mapnik_view_transform.cpp
    src/python_grid_utils.cpp
    src/python_grid_utils.hpp
    src/python_optional.hpp
    src/python_to_value.hpp
)


target_link_libraries(python-mapnik PRIVATE
    Boost::boost 
    Boost::log 
    Boost::python${Python_VERSION_MAJOR}${Python_VERSION_MINOR}
    mapnik::core    
    mapnik::json
    mapnik::wkt
    mapnik::mapnik
    ${PYTHON_MAPNIK_OPTIONAL_LIBS}
)
target_include_directories(python-mapnik PRIVATE 
    ${MAPBOX_GEOMETRY_INCLUDE_DIRS}
    ${PYTHON_MAPNIK_OPTIONAL_INC}
)

set_target_properties(python-mapnik PROPERTIES 
    OUTPUT_NAME "_mapnik"
    PREFIX ""
)



target_compile_definitions(python-mapnik PRIVATE ${PYTHON_MAPNIK_COMPILE_DEFS})


# copy plugins, fonts
file(TO_CMAKE_PATH "${CMAKE_LIBRARY_OUTPUT_DIRECTORY}" MODULE_PATH)
set(MODULE_LIB_DIR "${MODULE_PATH}/lib/mapnik")
set(MODULE_FONTS_DIR "${MODULE_LIB_DIR}/fonts")
set(MODULE_PLUGINS_DIR "${MODULE_LIB_DIR}/input")
file(COPY "${MAPNIK_FONTS_DIR}/" DESTINATION "${MODULE_FONTS_DIR}/")

mapnik_find_plugin_dir(MAPNIK_PLUGINS_DIR)
file(COPY "${MAPNIK_PLUGINS_DIR}/" DESTINATION "${MODULE_PLUGINS_DIR}/" PATTERN "*.input")

# copy dlls on windows
if(WIN32)
    file(GENERATE OUTPUT ${CMAKE_CURRENT_BINARY_DIR}/$<CONFIG>_install_deps.cmake CONTENT "
    file(GET_RUNTIME_DEPENDENCIES
        RESOLVED_DEPENDENCIES_VAR RESOLVED_DEPS
        UNRESOLVED_DEPENDENCIES_VAR UNRESOLVED_DEPS
        LIBRARIES 
            \"${MODULE_PLUGINS_DIR}/csv.input\"
            \"${MODULE_PLUGINS_DIR}/gdal.input\"
            \"${MODULE_PLUGINS_DIR}/geobuf.input\"
            \"${MODULE_PLUGINS_DIR}/geojson.input\"
            \"${MODULE_PLUGINS_DIR}/ogr.input\"
            \"${MODULE_PLUGINS_DIR}/pgraster.input\"
            \"${MODULE_PLUGINS_DIR}/postgis.input\"
            \"${MODULE_PLUGINS_DIR}/raster.input\"
            \"${MODULE_PLUGINS_DIR}/shape.input\"
            \"${MODULE_PLUGINS_DIR}/sqlite.input\"
            \"${MODULE_PLUGINS_DIR}/topojson.input\"
            ${MODULE_FILE_PATH}
        DIRECTORIES \"${VCPKG_INSTALLED_DIR}/${VCPKG_TARGET_TRIPLET}/bin/\"
        PRE_EXCLUDE_REGEXES \"api-ms-*\" \"ext-ms-*\"
        POST_EXCLUDE_REGEXES \".*system32/.*\\\\.dll\"
    )
    message(STATUS \"X \${RESOLVED_DEPS}\")
    #foreach(dep \${RESOLVED_DEPS})
    #    file(COPY \${dep} DESTINATION \"${MODULE_PATH}\")
    #endforeach()
    foreach(dep \${UNRESOLVED_DEPS})
        message(WARNING \"Runtime dependency \${dep} could not be resolved.\")
    endforeach()
    ")
    add_custom_command(TARGET python-mapnik POST_BUILD COMMAND ${CMAKE_COMMAND} -P ${CMAKE_CURRENT_BINARY_DIR}/$<CONFIG>_install_deps.cmake)
endif()

# write configure file
configure_file("${CMAKE_CURRENT_SOURCE_DIR}/src/paths.py.in" "${MODULE_PATH}/paths.py")
