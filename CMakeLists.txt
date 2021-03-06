cmake_minimum_required(VERSION 2.8)

# This project doesn't require any of the standard compilers, hence the "NONE"
# argument.
project(Tangelo NONE)

mark_as_advanced(CMAKE_INSTALL_PREFIX)

# Extend cmake module path.
list(APPEND CMAKE_MODULE_PATH "${CMAKE_SOURCE_DIR}/cmake")

# Include utility functions.
include(tangelo)

# Cache variables.
#
# Whether to build Tangelo at all, and then if yes, whether to build the Python
# package.
#
# This option is marked as advanced, as it only applies in a particular set of
# circumstances that apply mainly to Tangelo developers.
set(BUILD_TANGELO ON CACHE BOOL "Build Tangelo")
mark_as_advanced(BUILD_TANGELO)
if(${BUILD_TANGELO})
    set(BUILD_TANGELO_PYTHON_PACKAGE ON CACHE BOOL "Generate a Python package for Tangelo")
    mark_as_advanced(CLEAR BUILD_TANGELO_PYTHON_PACKAGE)
    mark_as_advanced(CLEAR JSLint_EXECUTABLE)
    mark_as_advanced(CLEAR UglifyJS_EXECUTABLE)
else()
    mark_as_advanced(FORCE BUILD_TANGELO_PYTHON_PACKAGE)
    mark_as_advanced(FORCE JSLint_EXECUTABLE)
    mark_as_advanced(FORCE UglifyJS_EXECUTABLE)
endif()

# Whether to build Sphinx documentation.
set(BUILD_DOCUMENTATION OFF CACHE BOOL "Generate documentation for project.")
if(${BUILD_DOCUMENTATION})
    mark_as_advanced(CLEAR Sphinx_EXECUTABLE)
else()
    mark_as_advanced(FORCE Sphinx_EXECUTABLE)
endif()

# Show/hide testing options depending on whether testing is enabled.
if(${BUILD_TESTING})
    set(JSLINT_TESTS OFF CACHE BOOL "Generate tests for JS source files using JSLint program.")
    mark_as_advanced(CLEAR JSLINT_TESTS)

    set(SERVICE_TESTS OFF CACHE BOOL "Generate tests for Tangelo test services.")
    mark_as_advanced(CLEAR SERVICE_TESTS)
else()
    mark_as_advanced(FORCE JSLINT_TESTS)
    mark_as_advanced(FORCE SERVICE_TESTS)
endif()

# We will need the Python interpreter to build the package and run the tests.
find_package(PythonInterp 2.7 REQUIRED)

# Create a binary directory to hold various timestamps.
file(MAKE_DIRECTORY ${CMAKE_BINARY_DIR}/stamps)

# These files should be validated with JSLint.
set(JS_LINT_FILES
    ${CMAKE_SOURCE_DIR}/src/js/core/core.js
    ${CMAKE_SOURCE_DIR}/src/js/core/util.js
    ${CMAKE_SOURCE_DIR}/src/js/plugin/controlPanel.js
    ${CMAKE_SOURCE_DIR}/src/js/plugin/GoogleMapSVG.js
    ${CMAKE_SOURCE_DIR}/src/js/plugin/svgColorLegend.js
    ${CMAKE_SOURCE_DIR}/src/js/plugin/dendrogram.js
    ${CMAKE_SOURCE_DIR}/src/js/plugin/geodots.js
    ${CMAKE_SOURCE_DIR}/src/js/plugin/geonodelink.js
    ${CMAKE_SOURCE_DIR}/src/js/plugin/mapdots.js
    ${CMAKE_SOURCE_DIR}/src/js/plugin/nodelink.js
    ${CMAKE_SOURCE_DIR}/src/js/stream/_package.js
    ${CMAKE_SOURCE_DIR}/src/js/stream/stream.js
    ${CMAKE_SOURCE_DIR}/src/js/vtkweb/_package.js
    ${CMAKE_SOURCE_DIR}/src/js/vtkweb/vtkweb.js
    ${CMAKE_SOURCE_DIR}/src/js/widget/widget.js
    ${CMAKE_SOURCE_DIR}/tangelo/web/examples/index.js
    ${CMAKE_SOURCE_DIR}/tangelo/web/examples/landingPage.js
    ${CMAKE_SOURCE_DIR}/tangelo/web/examples/app-template/myapp.js
    ${CMAKE_SOURCE_DIR}/tangelo/web/examples/charitynet/charitynet.js
    ${CMAKE_SOURCE_DIR}/tangelo/web/examples/enron/enron.js
    ${CMAKE_SOURCE_DIR}/tangelo/web/examples/enron-impala/enron-imp.js
    ${CMAKE_SOURCE_DIR}/tangelo/web/examples/enron-titan/enron-titan.js
    ${CMAKE_SOURCE_DIR}/tangelo/web/examples/flickr/flickr.js
    ${CMAKE_SOURCE_DIR}/tangelo/web/examples/ner/ner.js
    ${CMAKE_SOURCE_DIR}/tangelo/web/examples/options/options.js
    ${CMAKE_SOURCE_DIR}/tangelo/web/examples/primes/primes.js
    ${CMAKE_SOURCE_DIR}/tangelo/web/examples/vegabuilder/vegabuilder.js
)

# These files should be uglified and concatenated.
set(JS_UGLIFY_FILES
    ${CMAKE_SOURCE_DIR}/src/js/core/core.js
    ${CMAKE_SOURCE_DIR}/src/js/core/util.js
    ${CMAKE_SOURCE_DIR}/src/js/widget/widget.js
    ${CMAKE_SOURCE_DIR}/src/js/vega/_package.js
    ${CMAKE_SOURCE_DIR}/src/js/vega/geovis.js
    ${CMAKE_SOURCE_DIR}/src/js/vtkweb/_package.js
    ${CMAKE_SOURCE_DIR}/src/js/vtkweb/vtkweb.js
    ${CMAKE_SOURCE_DIR}/src/js/plugin/controlPanel.js
    ${CMAKE_SOURCE_DIR}/src/js/plugin/GoogleMapSVG.js
    ${CMAKE_SOURCE_DIR}/src/js/plugin/svgColorLegend.js
    ${CMAKE_SOURCE_DIR}/src/js/plugin/dendrogram.js
    ${CMAKE_SOURCE_DIR}/src/js/plugin/geodots.js
    ${CMAKE_SOURCE_DIR}/src/js/plugin/geonodelink.js
    ${CMAKE_SOURCE_DIR}/src/js/plugin/mapdots.js
    ${CMAKE_SOURCE_DIR}/src/js/plugin/nodelink.js
    ${CMAKE_SOURCE_DIR}/src/js/stream/_package.js
    ${CMAKE_SOURCE_DIR}/src/js/stream/stream.js
    ${CMAKE_SOURCE_DIR}/src/js/data/_package.js
    ${CMAKE_SOURCE_DIR}/src/js/data/tree.js
)

# These files should be copied to the build directory.
glob_recurse_ungit(TANGELO_SOURCES tangelo/*)

if(${BUILD_TANGELO})
    # Copy package files to build directory.
    foreach(m ${TANGELO_SOURCES})
        set(outfile ${CMAKE_BINARY_DIR}/${m})
        copy_file(${m} ${outfile})
        list(APPEND TANGELO_BUILD_FILES ${outfile})
    endforeach()

    # Also copy the README file.
    copy_file(README.rst ${CMAKE_BINARY_DIR}/tangelo/README)

    # And the server testing framework script.
    copy_file(
        testing/tangelo-server-test.sh
        ${CMAKE_BINARY_DIR}/testing/tangelo-server-test.sh
    )

    # Uglify and concatenate the library js files.
    #
    # Set up options to create an unminified file.
    set(concatenator cat)
    set(output_operator ">")

    # Use UglifyJS to create a minified file, falling back on an unminified file if
    # it cannot be found.
    find_package(UglifyJS)
    if(UglifyJS_EXECUTABLE)
        set(min_concatenator ${UglifyJS_EXECUTABLE})
        set(min_output_operator "-o")
        set(min_mangle_flag "-m")
    else()
        message(WARNING "Could not find UglifyJS - generating UNMINIFIED javascript file for tangelo.min.js")

        set(min_concatenator ${concatenator})
        set(min_output_operator ${output_operator})
        unset(min_mangle_flag)
    endif()

    set(MIN_JS_FILE ${CMAKE_BINARY_DIR}/tangelo/web/js/tangelo.min.js)
    set(JS_FILE ${CMAKE_BINARY_DIR}/tangelo/web/js/tangelo.js)

    add_custom_command(
        OUTPUT ${MIN_JS_FILE}
        COMMAND ${CMAKE_COMMAND} -E make_directory ${CMAKE_BINARY_DIR}/tangelo/web/js
        COMMAND ${min_concatenator} ${JS_UGLIFY_FILES} ${min_mangle_flag} ${min_output_operator} ${MIN_JS_FILE}
        DEPENDS ${JS_UGLIFY_FILES}
        COMMENT "Creating minified javascript library"
    )

    add_custom_command(
        OUTPUT ${JS_FILE}
        COMMAND ${concatenator} ${JS_UGLIFY_FILES} ${mangle_flag} ${output_operator} ${JS_FILE}
        DEPENDS ${JS_UGLIFY_FILES}
        COMMENT "Creating unminified javascript library"
    )
endif()

# Generate documentation.
if(${BUILD_DOCUMENTATION})
    # Find Sphinx
    find_package(Sphinx REQUIRED)

    # A signal file to indicate the last time the documentation was built.
    set(DOCUMENTATION_STAMP ${CMAKE_BINARY_DIR}/stamps/documentation)

    set(DOCUMENTATION_SOURCES
        ${CMAKE_SOURCE_DIR}/docs/advanced-usage.rst
        ${CMAKE_SOURCE_DIR}/docs/architecture.rst
        ${CMAKE_SOURCE_DIR}/docs/basic-usage.rst
        ${CMAKE_SOURCE_DIR}/docs/building-an-app.rst
        ${CMAKE_SOURCE_DIR}/docs/coding-style-guide.rst
        ${CMAKE_SOURCE_DIR}/docs/developing-visualizations.rst
        ${CMAKE_SOURCE_DIR}/docs/index.rst
        ${CMAKE_SOURCE_DIR}/docs/installation.rst
        ${CMAKE_SOURCE_DIR}/docs/python-services.rst
        ${CMAKE_SOURCE_DIR}/docs/tangelo-js.rst
        ${CMAKE_SOURCE_DIR}/docs/tangelo-manpage.rst
        ${CMAKE_SOURCE_DIR}/docs/tangelo-passwd-manpage.rst
        ${CMAKE_SOURCE_DIR}/docs/tangelo-py.rst
        ${CMAKE_SOURCE_DIR}/docs/setup.rst
        ${CMAKE_SOURCE_DIR}/docs/static/tangelo-sphinx.css
        ${CMAKE_SOURCE_DIR}/docs/templates/layout.html
    )

    set(DOCUMENTATION_CONTROL_FILE
        ${CMAKE_SOURCE_DIR}/docs/conf.py
    )

    # A command to generate the documentation.
    add_custom_command(
        OUTPUT ${DOCUMENTATION_STAMP}
        COMMAND ${Sphinx_EXECUTABLE} -b html ${CMAKE_SOURCE_DIR}/docs ${CMAKE_BINARY_DIR}/tangelo/web/docs && ${CMAKE_COMMAND} -E touch ${DOCUMENTATION_STAMP}
        DEPENDS ${DOCUMENTATION_SOURCES} ${DOCUMENTATION_CONTROL_FILE}
        COMMENT "Generating documentation"
    )
endif()

if(${BUILD_TANGELO} AND ${BUILD_TANGELO_PYTHON_PACKAGE})
    # A target to build the package.
    set(DIST_STAMP ${CMAKE_BINARY_DIR}/stamps/sdist)
    add_custom_command(
        OUTPUT ${DIST_STAMP}
        COMMAND ${PYTHON_EXECUTABLE} setup.py sdist --dist-dir ../sdist >../sdist.log && ${CMAKE_COMMAND} -E touch ${DIST_STAMP}
        DEPENDS ${TANGELO_BUILD_FILES}
        WORKING_DIRECTORY ${CMAKE_BINARY_DIR}/tangelo
        COMMENT "Generating Python package"
        )
endif()

# The top-level target.
get_property(ALL_COPIED_FILES GLOBAL PROPERTY copied_files)
add_custom_target(
    tangelo-all
    ALL
    DEPENDS ${ALL_COPIED_FILES} ${EXAMPLES_SYMLINK} ${MIN_JS_FILE} ${JS_FILE} ${DOCUMENTATION_STAMP} ${DIST_STAMP}
)

# Testing
include(CTest)
enable_testing()

set(TESTING_SOURCES
    testing/vtkweb/vtkwebtest.py
    testing/vtkweb/baseline-cone.png
    testing/vtkweb/baseline-phylotree.png
    testing/vtkweb/tangelo-vtkweb-test.py
)

foreach(f ${TESTING_SOURCES})
    configure_file(${f} ${f} COPYONLY)
endforeach()

# VTKWeb test.
add_test(
    NAME vtkwebcone
    WORKING_DIRECTORY ${CMAKE_BINARY_DIR}/testing/vtkweb
    COMMAND /home/roni/work/VTK/build-osmesa/bin/vtkpython vtkweb-cone-test.py
)

# JSLint tests.
if(${JSLINT_TESTS})
    # Find JSLint.
    find_package(JSLint REQUIRED)

    # Generate the tests.
    foreach(f ${JS_LINT_FILES})
        add_test(jslint-${f} ${JSLint_EXECUTABLE} ${f})
    endforeach()
endif()

if(NOT "${WIN32}")
    set(tester "${CMAKE_BINARY_DIR}/testing/tangelo-server-test.sh")
    set(site "${SERVER_HOSTNAME}:${SERVER_PORT}")

    # curl uses \r at the ends of lines for some reason when reporting response
    # headers, so we include one at the end of our "expected string" lines.
    add_test(static-content ${tester}
        "curl --silent -I ${site} | head -n 1"
        "HTTP/1.1 200 OK\r"
    )

    add_test(geodots ${tester}
        "python ${CMAKE_SOURCE_DIR}/src/imagetest.py --name=geodots --url=http://${site}/app/testing/vis/geodots.html --baseline-url=http://${site}/app/testing/vis/geodots.png"
        "success"
    )

    add_test(resolve ${tester}
        "python ${CMAKE_SOURCE_DIR}/tangelo/web/examples/testing/resolve/test.py"
        "success"
    )

    if(SERVICE_TESTS)
        add_test(rest-get ${tester}
            "curl --silent -X GET ${site}/service/test/restful/one/two/three?foo=bar"
            "GET: one two three{'foo': u'bar'}"
        )

        add_test(rest-post ${tester}
            "curl --silent -X POST -d foo=bar ${site}/service/test/restful/one/two/three"
            "POST: one two three{'foo': u'bar'}"
        )

        add_test(404-not-found ${tester}
            "curl --silent -I ${site}/service/test/404 | head -n 1"
            "HTTP/1.1 404 Not Found\r"
        )

        add_test(service-ner ${tester}
            "curl --silent \"${site}/app/ner/service/ner?text=Toto,+I've+got+a+feeling+we're+not+in+Kansas+anymore.\""
            "{\"result\": [[\"GPE\", \"Toto\"], [\"GPE\", \"Kansas\"]], \"error\": null}"
        )

        add_test(service-echo-blank ${tester}
            "curl --silent ${site}/service/test/echo"
            "(No arguments passed)"
        )

        add_test(service-echo ${tester}
            "curl --silent \"${site}/service/test/echo/jan/feb/mar?color=red&answer=42\""
            "[jan, feb, mar]\ncolor -> red\nanswer -> 42"
        )
    endif()
endif()
