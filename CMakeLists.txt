cmake_minimum_required( VERSION 3.13 )

# project pyth-client
project( pyth-client )

# default build convention
if( NOT CMAKE_BUILD_TYPE )
  set( CMAKE_BUILD_TYPE Release )
endif()

# use root directory as include
set( CMAKE_INCLUDE_CURRENT_DIR ON )

# find oracle header files
include_directories( program/c/src/ )

# gcc compiler/linker flags
add_compile_options( -ggdb -Wall -Wextra -Wsign-conversion -Werror -Wno-deprecated-declarations -m64 )
set( CMAKE_CXX_STANDARD 14)
set( CMAKE_CXX_FLAGS -std=c++14 )
set( CMAKE_C_FLAGS -std=c99 )
set( CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -pthread")

#
# pyth client API library
#
set( PC_SRC
  pc/attr_id.cpp;
  pc/capture.cpp;
  pc/key_pair.cpp;
  pc/key_store.cpp;
  pc/jtree.cpp;
  pc/log.cpp;
  pc/manager.cpp;
  pc/mem_map.cpp;
  pc/misc.cpp;
  pc/net_socket.cpp;
  pc/pub_stats.cpp;
  pc/replay.cpp;
  pc/request.cpp;
  pc/rpc_client.cpp;
  pc/user.cpp;
  program/c/src/oracle/model/price_model.c
  )

set( PC_HDR
  pc/attr_id.hpp;
  pc/capture.hpp;
  pc/dbl_list.hpp;
  pc/error.hpp;
  pc/jtree.hpp;
  pc/key_pair.hpp;
  pc/key_store.hpp;
  pc/hash_map.hpp;
  pc/log.hpp;
  pc/manager.hpp;
  pc/mem_map.hpp;
  pc/misc.hpp;
  pc/net_socket.hpp;
  pc/replay.hpp;
  pc/request.hpp;
  pc/rpc_client.hpp
  pc/user.hpp )

add_library( pc STATIC ${PC_SRC} )

# dependencies
set( PC_DEP pc ssl crypto z zstd )

#
# applications
#

add_executable( pythd pcapps/pythd.cpp )
target_link_libraries( pythd ${PC_DEP} )
add_executable( pyth pcapps/pyth.cpp )
target_link_libraries( pyth ${PC_DEP} )
add_executable( pyth_admin pcapps/admin_rpc_client.cpp pcapps/admin_request.cpp pcapps/pyth_admin.cpp )
target_link_libraries( pyth_admin ${PC_DEP} )
add_executable( pyth_csv pcapps/pyth_csv.cpp )
target_link_libraries( pyth_csv ${PC_DEP} )
add_executable( pyth_tx pcapps/tx_rpc_client.cpp pcapps/tx_svr.cpp pcapps/pyth_tx.cpp )
target_link_libraries( pyth_tx ${PC_DEP} )

#
# install
#

install( TARGETS pc DESTINATION lib )
install( TARGETS pyth pyth_admin pythd pyth_csv pyth_tx DESTINATION bin )
install( FILES ${PC_HDR} DESTINATION include/pc )
install( FILES program/c/src/oracle/oracle.h DESTINATION include/oracle )

#
# test programs
#

enable_testing()
add_executable( test_unit pctest/test_unit.cpp )
target_link_libraries( test_unit ${PC_DEP} )
add_executable( test_net pctest/test_net.cpp )
target_link_libraries( test_net ${PC_DEP} )

# This doesn't build on the bullseye base image, due to a packaging bug
# in the newer version of libqt5websockets5-dev for Debian. The below build instructions
# are left in for completeness, and work with libqt5websockets5-dev=5.11.3-5 (buster).
# find_package(Qt5 COMPONENTS Core Network WebSockets)
# add_executable( test_publish_websocket pctest/test_publish_websocket.cpp )
# target_link_libraries( test_publish_websocket Qt5::Core Qt5::Network Qt5::WebSockets /usr/local/lib/libjcon.so)

add_executable( test_qset pctest/test_qset.cpp )
target_link_libraries( test_qset ${PC_DEP} )
add_executable( test_twap pctest/test_twap.cpp )
target_link_libraries( test_twap ${PC_DEP} )
add_executable( leader_stats pctest/leader_stats.cpp )
target_link_libraries( leader_stats ${PC_DEP} )

add_test( test_unit test_unit )
add_test( test_net test_net )


#
# fuzz testing application
#

add_executable( fuzz pctest/fuzz.cpp )
target_link_libraries( fuzz ${PC_DEP} )


#
# bpf static analysis (CLion)
#

function( add_bpf_lib targ )
  if( SOLANA AND NOT BPF )
    set( BPF ${SOLANA}/sdk/bpf )
  endif()
  if( BPF )
    add_library( ${targ} STATIC ${ARGN} )
    if( ${CMAKE_VERSION} VERSION_GREATER_EQUAL "3.21" )
      # bpf.mk uses --std=c17
      set_property( TARGET ${targ} PROPERTY C_STANDARD 17 )
    else()
      set_property( TARGET ${targ} PROPERTY C_STANDARD 11 )
    endif()
    set_property( TARGET ${targ} PROPERTY C_STANDARD_REQUIRED ON )
    set_property( TARGET ${targ} PROPERTY C_EXTENSIONS OFF )
    target_compile_definitions( ${targ} PRIVATE __bpf__=1 )
    target_include_directories( ${targ} SYSTEM PRIVATE
      ${BPF}/c/inc
      ${BPF}/dependencies/criterion/include
    )
  endif()
endfunction()

# test_oracle.c includes oracle.c
add_bpf_lib( test-oracle-bpf program/c/src/oracle/test_oracle.c )
