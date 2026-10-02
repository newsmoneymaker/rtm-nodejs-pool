# rthash needs the Raptoreum Core source tree that was built (headers plus the static libraries of its build) and its depends prefix:
#   RTM=/path/to/raptoreum make        (default /root/rtm-src, HOST default x86_64-pc-linux-gnu)
# STATIC=-static makes a binary that runs on any Linux (build it where the node is built).
RTM ?= /root/rtm-src
HOST ?= x86_64-pc-linux-gnu
DEPS = $(RTM)/depends/$(HOST)
CXX ?= g++
STATIC ?=
SRC = $(RTM)/src
LIBS = $(SRC)/libraptoreum_server.a $(SRC)/libraptoreum_common.a $(SRC)/libraptoreum_util.a $(SRC)/libraptoreum_consensus.a \
  $(SRC)/crypto/libraptoreum_crypto.a $(SRC)/crypto/libraptoreum_crypto_sse41.a $(SRC)/crypto/libraptoreum_crypto_avx2.a $(SRC)/crypto/libraptoreum_crypto_x86_shani.a \
  $(SRC)/leveldb/libleveldb.a $(SRC)/leveldb/libmemenv.a $(SRC)/secp256k1/.libs/libsecp256k1.a $(SRC)/.libs/libunivalue.a \
  $(SRC)/dashbls/.libs/libdashbls.a
rthash: rthash.cpp
	$(CXX) -O2 -std=c++17 $(STATIC) -I$(SRC) -I$(SRC)/config -I$(DEPS)/include -o $@ $< -Wl,--start-group $(LIBS) -Wl,--end-group \
	  -L$(DEPS)/lib -lboost_system-mt-s-x64 -lboost_filesystem-mt-s-x64 -lboost_thread-mt-s-x64 -lssl -lcrypto -levent_pthreads -levent -lgmp -ldb_cxx -lzmq -lbacktrace -lpthread -ldl
clean:
	rm -f rthash
