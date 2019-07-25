cmake_minimum_required(VERSION 3.12)

project(ReadStat)

set(ReadStat_VERSION "\"0.2.0-prerelease\"")

# Find headers in "." as well
set(CMAKE_INCLUDE_CURRENT_DIR ON)

# Make it possible to use cmake files from the cmake subdir
set(CMAKE_MODULE_PATH ${CMAKE_CURRENT_SOURCE_DIR}/cmake ${CMAKE_MODULE_PATH})

string(APPEND CMAKE_C_FLAGS "-pedantic-errors -std=c99")

# Set installation paths
# This takes care of installing into "lib64" on distros that use that, for instance,
# by setting CMAKE_INSTALL_LIBDIR.
include(GNUInstallDirs)

# Enable "make test"
include(CTest)

set(libreadstat_la_SOURCES
    src/CKHashTable.c
    src/readstat_bits.c
    src/readstat_convert.c
    src/readstat_error.c
    src/readstat_io_unistd.c
    src/readstat_malloc.c
    src/readstat_metadata.c
    src/readstat_parser.c
    src/readstat_value.c
    src/readstat_variable.c
    src/readstat_writer.c
    src/sas/ieee.c
    src/sas/readstat_sas.c
    src/sas/readstat_sas7bcat_read.c
    src/sas/readstat_sas7bcat_write.c
    src/sas/readstat_sas7bdat_read.c
    src/sas/readstat_sas7bdat_write.c
    src/sas/readstat_sas_rle.c
    src/sas/readstat_xport.c
    src/sas/readstat_xport_read.c
    src/sas/readstat_xport_write.c
    src/spss/readstat_por.c
    src/spss/readstat_por_parse.c
    src/spss/readstat_por_read.c
    src/spss/readstat_por_write.c
    src/spss/readstat_sav.c
    src/spss/readstat_sav_compress.c
    src/spss/readstat_sav_parse.c
    src/spss/readstat_sav_parse_timestamp.c
    src/spss/readstat_sav_read.c
    src/spss/readstat_sav_write.c
    src/spss/readstat_spss.c
    src/spss/readstat_spss_parse.c
    src/stata/readstat_dta.c
    src/stata/readstat_dta_parse_timestamp.c
    src/stata/readstat_dta_read.c
    src/stata/readstat_dta_write.c
)

find_package(ZLIB)

if(ZLIB_FOUND)
    list(APPEND libreadstat_la_SOURCES
        src/spss/readstat_zsav_compress.c
        src/spss/readstat_zsav_read.c
        src/spss/readstat_zsav_write.c
    )
endif()

find_package(RAGEL)

if(RAGEL_FOUND)
    #TODO .rl.c:
    #TODO $(AM_V_GEN)$(RAGEL) $(RAGELFLAGS) -C $< -o $@
endif()

add_library(libreadstat STATIC ${libreadstat_la_SOURCES})
set_property(TARGET libreadstat PROPERTY OUTPUT_NAME readstat)
target_link_libraries(libreadstat PUBLIC m)
target_compile_options(libreadstat PRIVATE -Werror)

# When using ReadStat as a CMake subproject, find the header files automatically
target_include_directories(libreadstat PUBLIC "$<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/src>")

if(ZLIB_FOUND)
    target_link_libraries(libreadstat PUBLIC z)
    target_compile_definitions(libreadstat PUBLIC HAVE_ZLIB=1)
endif()

install(FILES man/readstat.man DESTINATION ${CMAKE_INSTALL_MANDIR}/man1)
install(FILES src/readstat.h DESTINATION ${CMAKE_INSTALL_INCLUDEDIR})

#### readstat ##################################################3

set(readstat_SOURCES
    src/bin/readstat.c
    src/bin/format.c
    src/bin/module_util.c
    src/bin/modules/double_decimals.c
    src/bin/modules/jsmn.c
    src/bin/modules/json_metadata.c
    src/bin/modules/produce_csv_value_dta.c
    src/bin/modules/produce_csv_value_sav.c
    src/bin/modules/produce_csv_value_csv.c
    src/bin/modules/produce_csv_value.c
    src/bin/modules/produce_value_label.c
    src/bin/modules/produce_value_label_dta.c
    src/bin/modules/produce_value_label_sav.c
    src/bin/modules/produce_missingness.c
    src/bin/modules/produce_missingness_dta.c
    src/bin/modules/produce_missingness_sav.c
    src/bin/modules/produce_csv_column_header.c
    src/bin/modules/mod_csv.c
    src/bin/modules/mod_readstat.c
    src/bin/util/readstat_dta_days.c
    src/bin/util/readstat_sav_date.c
)

find_library(CSVREADER csv)
if(LIBCSVREADER)
    list(APPEND reader_SOURCES src/bin/modules/mod_csv_reader.c)
endif()

add_executable(readstat ${readstat_SOURCES})
target_link_libraries(readstat PRIVATE libreadstat)
target_compile_definitions(readstat PRIVATE READSTAT_VERSION=${ReadStat_VERSION})

if(HAVE_CSVREADER)
    target_link_libraries(readstat PRIVATE ${LIBCSVREADER})
    target_compile_definitions(readstat PRIVATE HAVE_CSVREADER=1)
endif()

install(TARGETS readstat DESTINATION ${CMAKE_INSTALL_BINDIR})

#### extract_metadata ##################################################3

set(extract_metadata_SOURCES
    src/bin/format.c
    src/bin/extract_label_set.c
    src/bin/write_missing_values.c
    src/bin/write_value_labels.c
    src/bin/extract_metadata.c
    src/bin/util/readstat_dta_days.c
    src/bin/util/readstat_sav_date.c
)

find_library(LIBXLSXWRITER xlsxwriter)
if(LIBXLSXWRITER)
    list(APPEND extract_metadata_SOURCES src/bin/modules/mod_xlsx.c)
endif()

add_executable(extract_metadata ${extract_metadata_SOURCES})
target_link_libraries(extract_metadata PRIVATE libreadstat)
install(TARGETS extract_metadata DESTINATION ${CMAKE_INSTALL_BINDIR})

if(LIBXLSXWRITER)
    target_link_libraries(extract_metadata PRIVATE ${LIBXLSXWRITER})
    target_compile_definitions(extract_metadata PRIVATE HAVE_XLSXWRITER=1)
endif()

#### tests ##################################################3

set(test_readstat_SOURCES
    src/test/test_buffer.c
    src/test/test_buffer_io.c
    src/test/test_dta.c
    src/test/test_error.c
    src/test/test_read.c
    src/test/test_readstat.c
    src/test/test_sas.c
    src/test/test_sav.c
    src/test/test_write.c
)

add_executable(test_readstat ${test_readstat_SOURCES})
target_link_libraries(test_readstat libreadstat)
target_compile_options(test_readstat PRIVATE -Wall -Werror -DDEBUG=1)
add_test(test_readstat test_readstat)

set(test_dta_days_SOURCES
    src/bin/util/readstat_dta_days.c
    src/test/test_dta_days.c
)
add_executable(test_dta_days ${test_dta_days_SOURCES})
target_link_libraries(test_dta_days libreadstat)
target_compile_options(test_dta_days PRIVATE -Wall -Werror)
add_test(test_dta_days test_dta_days)

set(test_sav_date_SOURCES
    src/bin/util/readstat_sav_date.c
    src/test/test_sav_date.c
)
add_executable(test_sav_date ${test_sav_date_SOURCES})
target_link_libraries(test_sav_date PRIVATE libreadstat)
target_compile_options(test_sav_date PRIVATE -Wall -Werror)
add_test(test_sav_date test_sav_date)

set(test_double_decimals_SOURCES
    src/bin/modules/double_decimals.c
    src/test/test_double_decimals.c
)
add_executable(test_double_decimals ${test_double_decimals_SOURCES})
target_compile_options(test_double_decimals PRIVATE -Wall -Werror)
add_test(test_double_decimals test_double_decimals)

set(generate_corpus_SOURCES
    src/fuzz/generate_corpus.c
    src/test/test_buffer.c
    src/test/test_write.c
    src/test/test_read.c
    src/test/test_buffer_io.c
    src/test/test_error.c
    src/test/test_dta.c
    src/test/test_sas.c
    src/test/test_sav.c
)
add_executable(generate_corpus ${generate_corpus_SOURCES})
target_link_libraries(generate_corpus PRIVATE libreadstat)

find_library(LIBFUZZER Fuzzer)
if (LIBFUZZER_FOUND)

# noinst_PROGRAMS +=
# 	fuzz_compression_sas_rle
# 	fuzz_compression_sav
# 	fuzz_format_dta
# 	fuzz_format_por
# 	fuzz_format_sav
# 	fuzz_format_sas7bcat
# 	fuzz_format_sas7bdat
# 	fuzz_format_xport

    set(fuzz_format_dta_SOURCES
        src/fuzz/fuzz_format.c
        src/fuzz/fuzz_format_dta.c
        src/test/test_buffer_io.c
    )
    add_executable(fuzz_format_dta ${fuzz_format_dta_SOURCES})
    target_link_libraries(fuzz_format_dta PRIVATE libreadstat stdc++ ${LIBFUZZER})
    target_compile_options(fuzz_format_dta PRIVATE -Wall -Werror)

    set(fuzz_format_por_SOURCES
        src/fuzz/fuzz_format.c
        src/fuzz/fuzz_format_por.c
        src/test/test_buffer_io.c
    )
    add_executable(fuzz_format_por ${fuzz_format_por_SOURCES})
    target_link_libraries(fuzz_format_por PRIVATE libreadstat stdc++ ${LIBFUZZER})
    target_compile_options(fuzz_format_por PRIVATE -Wall -Werror)

    set(fuzz_format_sas7bcat_SOURCES
        src/fuzz/fuzz_format.c
        src/fuzz/fuzz_format_sas7bcat.c
        src/test/test_buffer_io.c
    )
    add_executable(fuzz_format_sas7bcat ${fuzz_format_sas7bcat_SOURCES})
    target_link_libraries(fuzz_format_sas7bcat PRIVATE libreadstat stdc++ ${LIBFUZZER})
    target_compile_options(fuzz_format_sas7bcat PRIVATE -Wall -Werror)

    set(fuzz_format_sas7bdat_SOURCES
        src/fuzz/fuzz_format.c
        src/fuzz/fuzz_format_sas7bdat.c
        src/test/test_buffer_io.c
    )
    add_executable(fuzz_format_sas7bdat ${fuzz_format_sas7bdat_SOURCES})
    target_link_libraries(fuzz_format_sas7bdat PRIVATE libreadstat stdc++ ${LIBFUZZER})
    target_compile_options(fuzz_format_sas7bdat PRIVATE -Wall -Werror)

    set(fuzz_format_sav_SOURCES
        src/fuzz/fuzz_format.c
        src/fuzz/fuzz_format_sav.c
        src/test/test_buffer_io.c
    )
    add_executable(fuzz_format_sav ${fuzz_format_sav_SOURCES})
    target_link_libraries(fuzz_format_sav PRIVATE libreadstat stdc++ ${LIBFUZZER})
    target_compile_options(fuzz_format_sav PRIVATE -Wall -Werror)

    set(fuzz_format_xport_SOURCES
        src/fuzz/fuzz_format.c
        src/fuzz/fuzz_format_xport.c
        src/test/test_buffer_io.c
    )
    add_executable(fuzz_format_export ${fuzz_format_export_SOURCES})
    target_link_libraries(fuzz_format_export PRIVATE libreadstat stdc++ ${LIBFUZZER})
    target_compile_options(fuzz_format_export PRIVATE -Wall -Werror)

    set(fuzz_compression_sas_rle_SOURCES
        src/fuzz/fuzz_compression_sas_rle.c
    )
    add_executable(fuzz_compression_sas_rle ${fuzz_compression_sas_rle_SOURCES})
    target_link_libraries(fuzz_compression_sas_rle PRIVATE libreadstat stdc++ ${LIBFUZZER})
    target_compile_options(fuzz_compression_sas_rle PRIVATE -Wall -Werror)

    set(fuzz_compression_sav_SOURCES
        src/fuzz/fuzz_compression_sav.c
    )
    add_executable(fuzz_compression_sav ${fuzz_compression_sav_SOURCES})
    target_link_libraries(fuzz_compression_sav PRIVATE libreadstat stdc++ ${LIBFUZZER})
    target_compile_options(fuzz_compression_sav PRIVATE -Wall -Werror)
endif()
