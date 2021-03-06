include(vtkGhostscript)

# Only run GL2PS tests if GhostScript is available
if(VTK_GHOSTSCRIPT_EXECUTABLE)
  set(GL2PSTests
    TestGL2PSBillboardTextActor3D.cxx
    TestGL2PSScalarBar.cxx
    TestGL2PSTextActor.cxx
    TestGL2PSTextActor3D.cxx
    TestGL2PSTextMapper.cxx
    TestGL2PSLabeledDataMapper.cxx
    TestStackedPlotGL2PS.cxx
    )

  set(GL2PSTestsPDFBothOpenGL
    TestGL2PSExporterRaster.cxx
    TestGL2PSExporterMultipleRenderers.cxx
    TestGL2PSExporterVolumeRaster.cxx
    )
  # TODO: Check why this test fails on OpenGL2
  set(GL2PSTestsPDFOpenGL1
    TestLinePlotGL2PS.cxx
  )
  set(GL2PSTestsPDF
    ${GL2PSTestsPDFBothOpenGL} ${GL2PSTestsPDFOpenGL1}
  )

  # These only produce PDFs. They're separated since the above tests produce
  # and verify both PDF and PS.
  if (VTK_RENDERING_BACKEND STREQUAL "OpenGL")
    # Mark this as NO_VALID on OpenGL backend. It looks like the textured
    # brush painting is broken. See issue #17009. We still run the test to
    # validate the generated PDF, but won't validate the OpenGL rendering.
    set(PDFTests TestPDFContextExport.cxx,NO_DATA,NO_VALID)
  else()
    set(PDFTests TestPDFContextExport.cxx,NO_DATA)
  endif()
endif()

vtk_add_test_cxx(${vtk-module}CxxTests tests
  X3DTest.cxx,NO_DATA,NO_VALID
  TestOBJExporter.cxx,NO_DATA,NO_VALID
  ${GL2PSTests} ${GL2PSTestsPDF} ${PDFTests}
  TestRIBExporter.cxx,NO_VALID
  UnitTestRIB.cxx,NO_DATA,NO_VALID
  )
vtk_test_cxx_executable(${vtk-module}CxxTests tests
  RENDERING_FACTORY
  )

# See if Pixar's Renderman is available
find_path(RMANTREE bin/prman HINTS
          /Applications/Pixar/RenderManProServer-19.0
          /opt/pixar/RenderManProServer-19.0
)
mark_as_advanced(RMANTREE)

# Tests to run if Renderman is present
if(RMANTREE)
  # write a Renderman ini file to specify the shaderpath
  write_file(${${vtk-module}_BINARY_DIR}/Testing/Cxx/rendermn.ini
          "/shaderpath &:${${vtk-module}_SOURCE_DIR}/Testing/Cxx")
  add_test(NAME ${vtk-module}-RIBExporterShaderCompile
          COMMAND ${RMANTREE}/bin/shader
          ${${vtk-module}_SOURCE_DIR}/Testing/Data/Input/bozo.sl
          ${${vtk-module}_SOURCE_DIR}/Testing/Data/Input/dented.sl
          WORKING_DIRECTORY ${${vtk-module}_BINARY_DIR}/Testing/Cxx
  )
  add_test(NAME ${vtk-module}-RIBExporterPrman
          COMMAND ${RMANTREE}/bin/prman
            ${VTK_TEST_OUTPUT_DIR}/TestRIBExporter.rib
          WORKING_DIRECTORY ${${vtk-module}_BINARY_DIR}/Testing/Cxx
  )
  # Image diff rman generated tif with baseline
  set(baseline_dir ${${vtk-module}_SOURCE_DIR}/Testing/Data/Baseline)
  ExternalData_add_test(VTKData
        NAME ${vtk-module}Cxx-ComparePrmanImage
        COMMAND ${RMANTREE}/bin/cmptiff -d 100
        DATA{${baseline_dir}/TestRIBExporter.tif}
        ${VTK_TEST_OUTPUT_DIR}/TestRIBExporter.tif
  )
  set_tests_properties(${vtk-module}-RIBExporterPrman
        PROPERTIES
        DEPENDS
        "${vtk-module}-TestRIBExporter"
        DEPENDS
        "${vtk-module}-RIBExporterShaderCompile"
        ENVIRONMENT
        "RMANTREE=${RMANTREE}"
  )
  set_tests_properties("${vtk-module}Cxx-ComparePrmanImage"
        PROPERTIES
        DEPENDS "${vtk-module}Cxx-TestRIBExporter"
        REQUIRED_FILES
          "${VTK_TEST_OUTPUT_DIR}/TestRIBExporter.tif"
  )
endif()

set(TestGL2PSExporterVolumeRasterError 50)

include(vtkTestingGL2PS)
vtk_add_pdf_test_cxx(${PDFTests})
if (VTK_RENDERING_BACKEND STREQUAL "OpenGL2")
  # text allignment and rotation with TextAsPath off is not
  # yet implemented for the OpenGL2 backend.
  vtk_add_gl2ps_test_cxx(${GL2PSTests} ${GL2PSTestsPDFOpenGL1})
  # test postscript and pdf
  vtk_add_gl2ps_test_cxx(pdf ${GL2PSTestsPDFBothOpenGL})
else()
  # tests postscript
  vtk_add_gl2ps_test_cxx(${GL2PSTests})
  # test postscript and pdf
  vtk_add_gl2ps_test_cxx(pdf ${GL2PSTestsPDF})
endif()
