tested `553bf98dc532a10616191ceadcfcafdc9a45435c` against its exact parent `1bf1dff4be4b2a604fd761b532e1c5aff4cc0bad` with a focused build of the actual epee HTTP path on Linux x86_64.

no correctness blocker found. the accepted-socket probe passed IPv4/IPv6 × RPC/NET/P2P × plaintext/TLS on both revisions. plaintext RPC changes from NODELAY off to on; plaintext NET/P2P stays off. incoming TLS already enables NODELAY during the handshake and stays on in both revisions.

four alternating-order rounds, 100 measured batches per scenario per revision:

| scenario | parent median | PR median |
| --- | ---: | ---: |
| one plaintext response | 0.050 ms | 0.050 ms |
| four pipelined plaintext responses | 44.003 ms | 0.079 ms |
| one 256 KiB plaintext response | 0.404 ms | 0.428 ms |
| four pipelined TLS responses | 0.137 ms | 0.137 ms |

these are whole-batch timings. the normal pipeline result repeated in all four rounds without forcing a delayed-ACK hint. all 12,800 measured responses passed status, length, and payload validation. the 256 KiB case had worse, noisy timings, including p95 0.955 → 1.870 ms, so this is not evidence of a win for every workload.

the harness uses production response construction, which sends headers and body together. the built-in HTTP client is sequential, so the pipeline improvement should not be read as typical wallet latency. no packet capture or full daemon/unit-test-suite run was performed.

also prepared two regression tests for incoming IPv4/IPv6 sockets, each checking explicit RPC=true and NET/P2P=false expectations after presetting the opposite socket option. both pass on this PR and fail only the RPC expectation on the parent. the test-only patch and complete reproduction sources are included below. this contribution was prepared with Codex assistance; the test and benchmark results come from executed checks, with the scope described above.


<details>
<summary>environment and complete timing summary</summary>

Linux 6.18.44 x86_64; GCC 13.3.0; C++17 Release; Boost 1.83.0. The C++ server used system OpenSSL 3.0.13; Python 3.12.14 used OpenSSL 3.5.8 on the client. The client enabled TCP_NODELAY equally for both revisions. Connection setup and TLS handshakes are excluded; timing includes Python response parsing and validation. One server worker, loopback only. No WAN, concurrent-client, packet-count, or macOS testing.

400 measured batches per scenario per revision; 15 warmup batches for each of four rounds. Case order was shuffled with seed 11053; revision order alternated. Each cell is median / p95 in milliseconds per batch.

| scenario | responses/batch | parent ms | PR ms |
| --- | ---: | ---: | ---: |
| plain HTTP sequential | 1 | 0.050 / 0.119 | 0.050 / 0.111 |
| plain HTTP pipeline | 4 | 44.003 / 47.964 | 0.079 / 0.444 |
| plain HTTP sequential, delayed-ACK hint | 1 | 0.049 / 0.129 | 0.050 / 0.170 |
| plain HTTP pipeline, delayed-ACK hint | 4 | 43.997 / 46.484 | 0.079 / 0.197 |
| plain HTTP, 256 KiB | 1 | 0.404 / 0.955 | 0.428 / 1.870 |
| TLS sequential | 1 | 0.063 / 0.220 | 0.061 / 0.120 |
| TLS pipeline | 4 | 0.137 / 0.283 | 0.137 / 0.315 |

Normal plaintext pipeline per-round medians: parent 43.998, 44.008, 44.002, 44.007 ms; PR 0.117, 0.080, 0.067, 0.079 ms. TCP_QUICKACK=0 cases are explicit Linux stress hints, not guaranteed delayed-ACK settings. The interpretation is consistent with Nagle/delayed-ACK interaction but was not confirmed by packet capture.

Incoming TLS's post-handshake NODELAY policy is unchanged. Outbound epee TLS configures the option before connection startup and is also affected by the changed startup predicate; outbound coverage here is static inspection only.

</details>

<details>
<summary>reproduction instructions</summary>

Save each source block below under its indicated filename in a new `review` directory. Save the test patch as `tcp_nodelay_regression.patch` in that directory.

Requirements: Linux with IPv4/IPv6 loopback, Git, C++17 compiler, CMake 3.20+, Ninja, Python 3, OpenSSL development libraries, and Boost chrono/filesystem/thread/regex/system development components.

```bash
git clone --no-checkout https://github.com/monero-project/monero.git monero
git -C monero fetch origin pull/11053/head
git -C monero checkout --detach 553bf98dc532a10616191ceadcfcafdc9a45435c
git -C monero submodule update --init external/gtest external/rapidjson
git -C monero worktree add --detach ../monero-parent 1bf1dff4be4b2a604fd761b532e1c5aff4cc0bad
git -C monero apply --check ../review/tcp_nodelay_regression.patch
git -C monero apply ../review/tcp_nodelay_regression.patch
cp monero/tests/unit_tests/tcp_nodelay.cpp review/tcp_nodelay.cpp
bash review/reproduce.sh "$PWD/monero-parent" "$PWD/monero" "$PWD/review-output"
```

The script writes build/test logs, all raw batch samples to `results/raw.csv`, and timing summaries to `results/summary.json`. The parent regression test should exit 1 with exactly two RPC expectation failures; the PR should pass both tests. Both socket probes should pass all 12 cases under their respective expected policies.

The component commands were run for the results above. The wrapper was syntax-checked; a second full run through the wrapper was not performed. The test patch was checked for applicability and whitespace. The full upstream unit-test executable was not built.

</details>

<details>
<summary>tcp_nodelay_regression.patch</summary>

```diff
diff --git a/tests/unit_tests/CMakeLists.txt b/tests/unit_tests/CMakeLists.txt
index eccff9a..21066bd 100644
--- a/tests/unit_tests/CMakeLists.txt
+++ b/tests/unit_tests/CMakeLists.txt
@@ -86,6 +86,7 @@ set(unit_tests_sources
   test_tx_utils.cpp
   test_peerlist.cpp
   test_protocol_pack.cpp
+  tcp_nodelay.cpp
   threadpool.cpp
   tx_proof.cpp
   tx_verification_utils.cpp
diff --git a/tests/unit_tests/tcp_nodelay.cpp b/tests/unit_tests/tcp_nodelay.cpp
new file mode 100644
index 0000000..9761a03
--- /dev/null
+++ b/tests/unit_tests/tcp_nodelay.cpp
@@ -0,0 +1,126 @@
+// Copyright (c) 2026, The Monero Project
+//
+// All rights reserved.
+//
+// Redistribution and use in source and binary forms, with or without modification, are
+// permitted provided that the following conditions are met:
+//
+// 1. Redistributions of source code must retain the above copyright notice, this list of
+//    conditions and the following disclaimer.
+//
+// 2. Redistributions in binary form must reproduce the above copyright notice, this list
+//    of conditions and the following disclaimer in the documentation and/or other
+//    materials provided with the distribution.
+//
+// 3. Neither the name of the copyright holder nor the names of its contributors may be
+//    used to endorse or promote products derived from this software without specific
+//    prior written permission.
+//
+// THIS SOFTWARE IS PROVIDED BY THE COPYRIGHT HOLDERS AND CONTRIBUTORS "AS IS" AND ANY
+// EXPRESS OR IMPLIED WARRANTIES, INCLUDING, BUT NOT LIMITED TO, THE IMPLIED WARRANTIES OF
+// MERCHANTABILITY AND FITNESS FOR A PARTICULAR PURPOSE ARE DISCLAIMED. IN NO EVENT SHALL
+// THE COPYRIGHT HOLDER OR CONTRIBUTORS BE LIABLE FOR ANY DIRECT, INDIRECT, INCIDENTAL,
+// SPECIAL, EXEMPLARY, OR CONSEQUENTIAL DAMAGES (INCLUDING, BUT NOT LIMITED TO,
+// PROCUREMENT OF SUBSTITUTE GOODS OR SERVICES; LOSS OF USE, DATA, OR PROFITS; OR BUSINESS
+// INTERRUPTION) HOWEVER CAUSED AND ON ANY THEORY OF LIABILITY, WHETHER IN CONTRACT,
+// STRICT LIABILITY, OR TORT (INCLUDING NEGLIGENCE OR OTHERWISE) ARISING IN ANY WAY OUT OF
+// THE USE OF THIS SOFTWARE, EVEN IF ADVISED OF THE POSSIBILITY OF SUCH DAMAGE.
+//
+
+#include <chrono>
+#include <memory>
+#include <boost/asio/ip/tcp.hpp>
+#include <boost/make_shared.hpp>
+#include "gtest/gtest.h"
+#include "net/abstract_tcp_server2.h"
+#include "scope_guard.h"
+
+namespace
+{
+  struct protocol_handler
+  {
+    using connection_context = epee::net_utils::connection_context_base;
+    struct config_type {};
+
+    protocol_handler(epee::net_utils::i_service_endpoint*, config_type&, connection_context&) {}
+    void after_init_connection() {}
+    void handle_qued_callback() {}
+    bool release_protocol() { return true; }
+    bool handle_recv(const void*, std::size_t) { return false; }
+  };
+
+  void check_nodelay(const boost::asio::ip::address& address)
+  {
+    namespace net_utils = epee::net_utils;
+    using tcp = boost::asio::ip::tcp;
+    using connection = net_utils::connection<protocol_handler>;
+    const struct
+    {
+      net_utils::t_connection_type type;
+      bool expected;
+      const char* name;
+    } cases[] = {
+      {net_utils::e_connection_type_RPC, true, "RPC"},
+      {net_utils::e_connection_type_NET, false, "NET"},
+      {net_utils::e_connection_type_P2P, false, "P2P"}
+    };
+
+    for (const auto& test : cases)
+    {
+      SCOPED_TRACE(test.name);
+      boost::asio::io_context context;
+      tcp::acceptor acceptor{context};
+      const tcp::endpoint endpoint{address, 0};
+      boost::system::error_code error;
+      acceptor.open(endpoint.protocol(), error);
+      if (!error)
+        acceptor.bind(endpoint, error);
+      if (address.is_v6() &&
+          (error == boost::asio::error::address_family_not_supported ||
+           error == boost::system::errc::make_error_condition(
+             boost::system::errc::address_not_available)))
+        GTEST_SKIP() << "IPv6 loopback unavailable: " << error.message();
+      ASSERT_FALSE(error) << error.message();
+      acceptor.listen();
+
+      tcp::socket client{context};
+      client.connect(acceptor.local_endpoint());
+      tcp::socket accepted{context};
+      acceptor.accept(accepted);
+      // Require start() to establish the policy, irrespective of socket defaults.
+      accepted.set_option(tcp::no_delay{!test.expected});
+      auto server = boost::make_shared<connection>(context, std::move(accepted),
+        std::make_shared<connection::shared_state>(), test.type,
+        net_utils::ssl_support_t::e_ssl_support_disabled);
+      // Ensure exceptions cannot leave a live connection on a running context.
+      const epee::scope_guard stop_context{[&context] { context.stop(); }};
+      const bool started = server->start(true, false);
+      EXPECT_TRUE(started);
+      if (started)
+      {
+        tcp::no_delay actual;
+        server->socket().get_option(actual, error);
+        EXPECT_FALSE(error) << error.message();
+        if (!error)
+          EXPECT_EQ(test.expected, actual.value());
+      }
+
+      client.close();
+      server->cancel();
+      context.run_for(std::chrono::seconds{5});
+      EXPECT_TRUE(context.stopped()) << "connection cleanup timed out";
+      if (started)
+        EXPECT_EQ(connection::WASTED, server->get_status());
+    }
+  }
+}
+
+TEST(tcp_nodelay, incoming_ipv4)
+{
+  check_nodelay(boost::asio::ip::address_v4::loopback());
+}
+
+TEST(tcp_nodelay, incoming_ipv6)
+{
+  check_nodelay(boost::asio::ip::address_v6::loopback());
+}
```

</details>

<details>
<summary>CMakeLists.txt</summary>

```cmake
cmake_minimum_required(VERSION 3.20)
project(monero_rpc_transport_review LANGUAGES C CXX)
set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)
if(NOT MONERO_SOURCE)
  message(FATAL_ERROR "Set MONERO_SOURCE to the pinned Monero checkout")
endif()
find_package(Threads REQUIRED)
find_package(OpenSSL REQUIRED)
find_package(Boost 1.69 REQUIRED CONFIG COMPONENTS chrono filesystem thread regex system)
add_compile_definitions(AUTO_INITIALIZE_EASYLOGGINGPP BOOST_ASIO_ENABLE_SEQUENTIAL_STRAND_ALLOCATION BOOST_NO_AUTO_PTR BOOST_UUID_DISABLE_ALIGNMENT HAVE_EXPLICIT_BZERO HAVE_STRPTIME)
include_directories(
  "${MONERO_SOURCE}/contrib/epee/include"
  "${MONERO_SOURCE}/src"
  "${MONERO_SOURCE}/external"
  "${MONERO_SOURCE}/external/easylogging++"
  "${RAPIDJSON_INCLUDE}")
set(epee_sources byte_slice.cpp byte_stream.cpp hex.cpp abstract_http_client.cpp
  http_auth.cpp http_protocol_handler.cpp mlog.cpp net_helper.cpp net_utils_base.cpp
  string_tools.cpp parserse_base_utils.cpp wipeable_string.cpp levin_base.cpp memwipe.c
  connection_basic.cpp network_throttle.cpp network_throttle-detail.cpp mlocker.cpp
  buffer.cpp net_ssl.cpp int-util.cpp portable_storage.cpp misc_language.cpp file_io_utils.cpp
  net_parse_helpers.cpp)
list(TRANSFORM epee_sources PREPEND "${MONERO_SOURCE}/contrib/epee/src/")
add_library(review_epee STATIC ${epee_sources}
  "${MONERO_SOURCE}/external/easylogging++/easylogging++.cc")
target_link_libraries(review_epee PUBLIC Boost::chrono Boost::filesystem Boost::thread
  Boost::regex Boost::system OpenSSL::SSL OpenSSL::Crypto Threads::Threads)
add_executable(http_probe_server http_probe_server.cpp)
target_link_libraries(http_probe_server PRIVATE review_epee)
if(EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/socket_options.cpp")
  add_executable(socket_options socket_options.cpp)
  target_link_libraries(socket_options PRIVATE review_epee)
endif()
if(REGRESSION_SOURCE AND GTEST_SOURCE)
  set(BUILD_GMOCK OFF CACHE BOOL "" FORCE)
  set(INSTALL_GTEST OFF CACHE BOOL "" FORCE)
  add_subdirectory("${GTEST_SOURCE}" "${CMAKE_BINARY_DIR}/gtest")
  add_executable(tcp_nodelay_regression "${REGRESSION_SOURCE}")
  target_link_libraries(tcp_nodelay_regression PRIVATE review_epee gtest_main)
endif()
```

</details>

<details>
<summary>http_probe_server.cpp</summary>

```cpp
// SPDX-License-Identifier: BSD-3-Clause
// A dummy application over Monero's unmodified production epee HTTP transport.
// No blockchain, wallet, signing, or daemon RPC handlers are involved.
#include <iostream>
#include <string>
#include "net/http_server_impl_base.h"

class probe_server : public epee::http_server_impl_base<probe_server>
{
public:
  bool handle_http_request(const epee::net_utils::http::http_request_info& request,
    epee::net_utils::http::http_response_info& response,
    epee::net_utils::connection_context_base&) override
  {
    response.m_response_code = 200;
    response.m_response_comment = "OK";
    response.m_mime_tipe = "application/octet-stream";
    response.m_body.assign(request.m_URI == "/large" ? 262144 : 32, 'x');
    return true;
  }
};

int main(int argc, char** argv)
{
  try
  {
    mlog_configure("", false);
    mlog_set_log_level(-1);
    probe_server server;
    const auto ssl = argc > 1 && std::string(argv[1]) == "tls"
      ? epee::net_utils::ssl_support_t::e_ssl_support_enabled
      : epee::net_utils::ssl_support_t::e_ssl_support_disabled;
    if (!server.init(nullptr, "0", "127.0.0.1", "::", false, true, {}, boost::none, ssl))
      return 2;
    server.run(1, false);
    std::cout << "READY " << server.get_binded_port() << std::endl;
    std::string command;
    std::getline(std::cin, command);
    server.send_stop_signal();
    const bool stopped = server.timed_wait_server_stop(5000);
    server.deinit();
    return stopped ? 0 : 3;
  }
  catch (const std::exception& error)
  {
    std::cerr << error.what() << std::endl;
    return 1;
  }
}
```

</details>

<details>
<summary>benchmark.py</summary>

```python
#!/usr/bin/env python3
"""Paired parent/PR benchmarks of Monero's actual epee HTTP response path.

Only transport is exercised. The fixed dummy endpoint performs no real RPC work.
Linux TCP_QUICKACK=0 is an explicit stress condition, not normal client behavior.
"""
import argparse
import csv
import json
import math
import os
import platform
import random
import selectors
import socket
import ssl
import statistics
import subprocess
import time
from pathlib import Path

CASES = [
    ("http_serial", False, 1, False, 32),
    ("http_pipeline4", False, 4, False, 32),
    ("http_serial_delayed_ack_hint", False, 1, True, 32),
    ("http_pipeline4_delayed_ack_hint", False, 4, True, 32),
    ("http_large_256k", False, 1, False, 262144),
    ("tls_serial", True, 1, False, 32),
    ("tls_pipeline4", True, 4, False, 32),
]


class Server:
    def __init__(self, executable, tls, logfile):
        self.log = open(logfile, "w")
        self.process = subprocess.Popen([str(executable), "tls" if tls else "plain"],
            stdin=subprocess.PIPE, stdout=subprocess.PIPE, stderr=self.log, text=True)
        selector = selectors.DefaultSelector()
        selector.register(self.process.stdout, selectors.EVENT_READ)
        deadline = time.monotonic() + 30
        try:
            while time.monotonic() < deadline:
                if not selector.select(max(0, deadline - time.monotonic())):
                    break
                line = self.process.stdout.readline()
                if line.startswith("READY "):
                    self.port = int(line.split()[1])
                    return
                self.log.write(line)
                if not line:
                    break
            raise RuntimeError("server failed to become ready; see " + str(logfile))
        except BaseException:
            self.close()
            raise
        finally:
            selector.close()

    def close(self):
        if self.process.poll() is None:
            try:
                self.process.communicate("stop\n", timeout=10)
            except subprocess.TimeoutExpired:
                self.process.kill()
                self.process.communicate()
                raise RuntimeError("server did not stop cleanly")
        self.log.close()
        if self.process.returncode:
            raise RuntimeError("server exit code " + str(self.process.returncode))


class Reader:
    def __init__(self, sock):
        self.sock = sock
        self.buffer = b""

    def response(self, expected):
        while b"\r\n\r\n" not in self.buffer:
            self.receive()
        headers, self.buffer = self.buffer.split(b"\r\n\r\n", 1)
        lines = headers.split(b"\r\n")
        if lines[0].split()[1] != b"200":
            raise AssertionError("non-200 response: " + repr(lines[0]))
        fields = dict(line.split(b":", 1) for line in lines[1:] if b":" in line)
        fields = {k.lower(): v.strip() for k, v in fields.items()}
        length = int(fields[b"content-length"])
        if length != expected:
            raise AssertionError(f"expected {expected} bytes, received length {length}")
        while len(self.buffer) < length:
            self.receive()
        body, self.buffer = self.buffer[:length], self.buffer[length:]
        if body != b"x" * expected:
            raise AssertionError("corrupt payload")

    def receive(self):
        chunk = self.sock.recv(1048576)
        if not chunk:
            raise EOFError("connection closed before complete response")
        self.buffer += chunk


def measure(server, case, warmup, count):
    name, tls, batch, delayed, size = case
    raw = socket.create_connection(("127.0.0.1", server.port), timeout=5)
    raw.setsockopt(socket.IPPROTO_TCP, socket.TCP_NODELAY, 1)
    sock = raw
    if tls:
        context = ssl.SSLContext(ssl.PROTOCOL_TLS_CLIENT)
        context.check_hostname = False
        context.verify_mode = ssl.CERT_NONE  # local test-only generated certificate
        sock = context.wrap_socket(raw, server_hostname="localhost")
    request = (f"GET {'/large' if size > 32 else '/small'} HTTP/1.1\r\n"
        "Host: localhost\r\nConnection: keep-alive\r\nContent-Length: 0\r\n\r\n").encode() * batch
    reader = Reader(sock)
    rows = []
    try:
        for index in range(warmup + count):
            if delayed:
                sock.setsockopt(socket.IPPROTO_TCP, socket.TCP_QUICKACK, 0)
            start = time.perf_counter_ns()
            sock.sendall(request)  # one client write for the whole batch
            for _ in range(batch):
                reader.response(size)
            elapsed = time.perf_counter_ns() - start
            if reader.buffer:
                raise AssertionError("unexpected response bytes after batch")
            if index >= warmup:
                rows.append({"case": name, "sample": index - warmup,
                    "batch_size": batch, "body_bytes": size, "latency_ns": elapsed})
    finally:
        sock.close()
    return rows


def percentile(values, p):
    values = sorted(values)
    return values[max(0, math.ceil(p * len(values)) - 1)]


def main():
    parser = argparse.ArgumentParser(description=__doc__)
    parser.add_argument("--parent", type=Path, required=True)
    parser.add_argument("--pr", type=Path, required=True)
    parser.add_argument("--out", type=Path, required=True)
    parser.add_argument("--rounds", type=int, default=4)
    parser.add_argument("--samples", type=int, default=100)
    parser.add_argument("--warmup", type=int, default=15)
    args = parser.parse_args()
    if min(args.rounds, args.samples) <= 0 or args.warmup < 0:
        parser.error("rounds/samples must be positive and warmup nonnegative")
    args.out.mkdir(parents=True, exist_ok=True)
    samples = []
    schedule = []
    rng = random.Random(11053)
    started = time.time()
    for round_index in range(args.rounds):
        variants = ["parent", "pr"] if round_index % 2 == 0 else ["pr", "parent"]
        cases = list(CASES)
        rng.shuffle(cases)
        for case in cases:
            for variant in variants:
                schedule.append({"round": round_index, "case": case[0], "variant": variant})
                server = Server(getattr(args, variant).resolve(), case[1],
                    args.out / f"server-{round_index}-{case[0]}-{variant}.log")
                try:
                    rows = measure(server, case, args.warmup, args.samples)
                    for row in rows:
                        row.update(round=round_index, variant=variant)
                    samples.extend(rows)
                finally:
                    server.close()
                median = statistics.median(row["latency_ns"] for row in rows) / 1e6
                print(f"round={round_index} {case[0]} {variant}: median {median:.3f} ms", flush=True)
    with (args.out / "raw.csv").open("w", newline="") as f:
        writer = csv.DictWriter(f, fieldnames=list(samples[0]))
        writer.writeheader()
        writer.writerows(samples)
    summaries = []
    for case in CASES:
        for variant in ("parent", "pr"):
            rows = [r for r in samples if r["case"] == case[0] and r["variant"] == variant]
            values = [r["latency_ns"] / 1e6 for r in rows]
            round_medians = [statistics.median(r["latency_ns"] / 1e6 for r in rows
                if r["round"] == n) for n in range(args.rounds)]
            summaries.append({"case": case[0], "variant": variant, "batch_size": case[2],
                "samples": len(values), "median_batch_ms": statistics.median(values),
                "p95_batch_ms": percentile(values, .95), "min_batch_ms": min(values),
                "max_batch_ms": max(values), "round_medians_ms": round_medians})
    report = {"host": platform.platform(), "python": platform.python_version(),
        "openssl": ssl.OPENSSL_VERSION, "cpu_count": os.cpu_count(),
        "started_utc": time.strftime("%Y-%m-%dT%H:%M:%SZ", time.gmtime(started)),
        "elapsed_seconds": time.time() - started,
        "arguments": {k: str(v) if isinstance(v, Path) else v for k, v in vars(args).items()},
        "validated_responses": sum(r["batch_size"] for r in samples),
        "failures": 0, "schedule": schedule, "summary": summaries,
        "limits": ["Loopback transport-only test, not daemon/wallet RPC performance",
            "Connection and TLS handshake times excluded", "One server worker",
            "TCP_QUICKACK=0 cases are explicit Linux delayed-ACK stress hints",
            "No packet capture; packet-count/ACK causality not directly measured"]}
    (args.out / "summary.json").write_text(json.dumps(report, indent=2) + "\n")
    print(json.dumps(summaries, indent=2))


if __name__ == "__main__":
    main()
```

</details>

<details>
<summary>socket_options.cpp</summary>

```cpp
// Standalone regression probe for Monero PR #11053.
// Exercises production epee::connection with real accepted loopback sockets.
// Usage: socket_options baseline|patched
// "baseline" expects the PR parent's plaintext policy; "patched" expects RPC
// TCP_NODELAY enabled. TLS expectations are identical for both revisions.

#include <chrono>
#include <functional>
#include <iostream>
#include <memory>
#include <stdexcept>
#include <string>

#include <boost/asio/ip/tcp.hpp>
#include <boost/asio/ip/v6_only.hpp>
#include <boost/asio/ssl.hpp>
#include <boost/asio/steady_timer.hpp>
#include <boost/asio/write.hpp>
#include <boost/shared_ptr.hpp>

#include "net/abstract_tcp_server2.h"
#include "net/net_ssl.h"

namespace
{
  namespace asio = boost::asio;
  namespace epee_net = epee::net_utils;
  using tcp = asio::ip::tcp;
  using error_code = boost::system::error_code;

  struct probe_config
  {
    std::function<void()> received;
  };

  struct probe_protocol
  {
    using connection_context = epee_net::connection_context_base;
    using config_type = probe_config;

    probe_config& config;

    probe_protocol(epee_net::i_service_endpoint*, config_type& config,
                   connection_context&) : config(config) {}

    void after_init_connection() {}
    void handle_qued_callback() {}
    bool release_protocol() { return true; }
    bool handle_recv(const void*, std::size_t)
    {
      if (config.received)
        config.received();
      return true;
    }
  };

  using connection = epee_net::connection<probe_protocol>;
  using connection_ptr = boost::shared_ptr<connection>;

  void require(bool condition, const std::string& message)
  {
    if (!condition)
      throw std::runtime_error(message);
  }

  bool get_nodelay(tcp::socket& socket)
  {
    tcp::no_delay actual;
    socket.get_option(actual);
    return actual.value();
  }

  void bind_loopback(tcp::acceptor& acceptor, bool ipv6)
  {
    const auto address = ipv6
      ? asio::ip::address(asio::ip::address_v6::loopback())
      : asio::ip::address(asio::ip::address_v4::loopback());
    const tcp::endpoint endpoint(address, 0);
    acceptor.open(endpoint.protocol());
    if (ipv6)
      acceptor.set_option(asio::ip::v6_only(true));
    acceptor.bind(endpoint);
    acceptor.listen();
  }

  // Cancellation must run through the production connection's event handlers.
  void stop_connection(asio::io_context& context, const connection_ptr& server)
  {
    server->cancel();
    if (context.stopped())
      context.restart();
    context.run_for(std::chrono::seconds(2));
    require(context.stopped(), "connection cleanup exceeded 2 seconds");
  }

  void plaintext_case(epee_net::t_connection_type type, bool ipv6, bool expected)
  {
    asio::io_context context;
    tcp::acceptor acceptor(context);
    bind_loopback(acceptor, ipv6);
    tcp::socket client(context);
    client.connect(acceptor.local_endpoint());
    tcp::socket accepted(context);
    acceptor.accept(accepted);

    // Verify start() actively establishes the policy, rather than accidentally
    // accepting whatever default this operating system gives a new socket.
    accepted.set_option(tcp::no_delay(!expected));
    auto state = std::make_shared<connection::shared_state>();
    connection_ptr server(new connection(context, std::move(accepted), state,
      type, epee_net::ssl_support_t::e_ssl_support_disabled));
    const bool started = server->start(true, false);
    const bool observed = started && get_nodelay(server->socket());
    epee_net::connection_context_base details;
    server->get_context(details);
    error_code ignored;
    client.close(ignored);
    stop_connection(context, server);
    require(started, "accepted connection failed to start");
    require(details.m_is_income, "connection was not recorded as incoming");
    require(observed == expected, "TCP_NODELAY expected " +
      std::to_string(expected) + ", observed " + std::to_string(observed));
  }

  void tls_case(epee_net::t_connection_type type, bool ipv6)
  {
    asio::io_context context;
    tcp::acceptor acceptor(context);
    bind_loopback(acceptor, ipv6);

    // Test-only self-signed certificate; verification is disabled exclusively
    // for this in-process loopback pair. No external endpoint is contacted.
    auto state = std::make_shared<connection::shared_state>();
    epee_net::ssl_options_t ssl_options(epee_net::ssl_support_t::e_ssl_support_enabled);
    ssl_options.verification = epee_net::ssl_verification_t::none;
    state->configure_ssl(ssl_options);
    asio::ssl::context client_context(asio::ssl::context::tlsv12_client);
    asio::ssl::stream<tcp::socket> client(context, client_context);
    client.set_verify_mode(asio::ssl::verify_none);
    client.next_layer().connect(acceptor.local_endpoint());
    tcp::socket accepted(context);
    acceptor.accept(accepted);
    accepted.set_option(tcp::no_delay(false));

    connection_ptr server(new connection(context, std::move(accepted), state,
      type, epee_net::ssl_support_t::e_ssl_support_enabled));
    asio::steady_timer deadline(context, std::chrono::seconds(5));
    bool received = false;
    bool observed = false;
    std::string failure;
    const auto finish = [&] {
      error_code ignored;
      deadline.cancel(ignored);
      client.next_layer().close(ignored);
      server->cancel();
    };
    state->received = [&] {
      received = true;
      observed = get_nodelay(server->socket());
      finish();
    };
    deadline.async_wait([&](error_code error) {
      if (!error) {
        failure = "TLS handshake/data deadline exceeded";
        finish();
      }
    });

    require(server->start(true, false), "TLS accepted connection failed to start");
    const std::string ping("x");
    client.async_handshake(asio::ssl::stream_base::client, [&](error_code error) {
      if (error) {
        failure = "TLS client handshake: " + error.message();
        finish();
        return;
      }
      asio::async_write(client, asio::buffer(ping),
        [&](error_code error, std::size_t) {
          if (error && !received) {
            failure = "TLS client write: " + error.message();
            finish();
          }
        });
    });
    context.run_for(std::chrono::seconds(7));
    const bool completed = context.stopped();
    if (!completed) {
      finish();
      context.stop();
    }
    // Drop callbacks before their captured locals leave scope.
    state->received = {};
    require(completed, "TLS event loop exceeded 7 seconds");
    require(failure.empty(), failure);
    require(received, "server did not receive decrypted TLS application data");
    require(observed, "TLS server TCP_NODELAY was disabled after handshake");
  }
}

int main(int argc, char** argv)
{
  if (argc != 2 || (std::string(argv[1]) != "baseline" &&
                    std::string(argv[1]) != "patched")) {
    std::cerr << "usage: " << argv[0] << " baseline|patched\n";
    return 2;
  }
  const bool patched = std::string(argv[1]) == "patched";
  struct case_type { epee_net::t_connection_type type; const char* name; };
  const case_type types[] = {
    {epee_net::e_connection_type_RPC, "RPC"},
    {epee_net::e_connection_type_NET, "NET"},
    {epee_net::e_connection_type_P2P, "P2P"}
  };
  unsigned passed = 0;
  unsigned failed = 0;
  for (const bool ipv6 : {false, true}) {
    for (const auto entry : types) {
      for (const bool tls : {false, true}) {
        const std::string name = std::string(ipv6 ? "IPv6 " : "IPv4 ") +
          entry.name + (tls ? " TLS-post-handshake" : " plaintext");
        try {
          if (tls)
            tls_case(entry.type, ipv6);
          else
            plaintext_case(entry.type, ipv6,
              patched && entry.type == epee_net::e_connection_type_RPC);
          ++passed;
          std::cout << "PASS " << name << '\n';
        }
        catch (const std::exception& error) {
          ++failed;
          std::cerr << "FAIL " << name << ": " << error.what() << '\n';
        }
      }
    }
  }
  std::cout << "passed=" << passed << " failed=" << failed
            << " expected_policy=" << argv[1] << '\n';
  return failed ? 1 : 0;
}
```

</details>

<details>
<summary>reproduce.sh</summary>

```bash
#!/usr/bin/env bash
set -euo pipefail

if (( $# < 3 )); then
  echo "usage: $0 PARENT_CHECKOUT PR_CHECKOUT OUTPUT_DIR [extra CMake options...]" >&2
  exit 2
fi
bundle_dir=$(cd -- "$(dirname -- "${BASH_SOURCE[0]}")" && pwd)
parent_dir=$(cd -- "$1" && pwd)
pr_dir=$(cd -- "$2" && pwd)
mkdir -p -- "$3"
output_dir=$(cd -- "$3" && pwd)
shift 3
cmake_bin=${CMAKE_BIN:-cmake}
build_jobs=${BUILD_JOBS:-3}

[[ $(git -C "$parent_dir" rev-parse HEAD) == 1bf1dff4be4b2a604fd761b532e1c5aff4cc0bad ]]
[[ $(git -C "$pr_dir" rev-parse HEAD) == 553bf98dc532a10616191ceadcfcafdc9a45435c ]]
git -C "$parent_dir" diff --quiet -- contrib/epee external/easylogging++
git -C "$pr_dir" diff --quiet -- contrib/epee external/easylogging++
test -f "$pr_dir/external/gtest/CMakeLists.txt"
test -f "$pr_dir/external/rapidjson/include/rapidjson/rapidjson.h"

for variant in parent pr; do
  source_dir=$parent_dir
  [[ $variant != pr ]] || source_dir=$pr_dir
  "$cmake_bin" -S "$bundle_dir" -B "$output_dir/build-$variant" -G Ninja \
    -DCMAKE_BUILD_TYPE=Release \
    -DMONERO_SOURCE="$source_dir" \
    -DRAPIDJSON_INCLUDE="$pr_dir/external/rapidjson/include" \
    -DREGRESSION_SOURCE="$bundle_dir/tcp_nodelay.cpp" \
    -DGTEST_SOURCE="$pr_dir/external/gtest" "$@" \
    > "$output_dir/configure-$variant.log" 2>&1
  "$cmake_bin" --build "$output_dir/build-$variant" -j "$build_jobs" \
    > "$output_dir/build-$variant.log" 2>&1
done

"$output_dir/build-parent/socket_options" baseline > "$output_dir/socket-parent.log" 2>&1
"$output_dir/build-pr/socket_options" patched > "$output_dir/socket-pr.log" 2>&1
"$output_dir/build-pr/tcp_nodelay_regression" --gtest_color=no \
  > "$output_dir/regression-pr.log" 2>&1
parent_status=0
"$output_dir/build-parent/tcp_nodelay_regression" --gtest_color=no \
  > "$output_dir/regression-parent.log" 2>&1 || parent_status=$?
if [[ $parent_status != 1 ]]; then
  echo "Expected parent regression test exit code 1, got $parent_status" >&2
  exit 1
fi
echo "PR tests passed; parent negative control failed as expected. Inspect regression-parent.log for RPC-only failures."

python3 "$bundle_dir/benchmark.py" \
  --parent "$output_dir/build-parent/http_probe_server" \
  --pr "$output_dir/build-pr/http_probe_server" \
  --out "$output_dir/results" --rounds 4 --samples 100 --warmup 15 \
  > "$output_dir/benchmark-run.log" 2>&1
echo "Completed. Results: $output_dir/results/summary.json"
```

</details>

<details>
<summary>regression-pr.log</summary>

```text
Running main() from monero/external/gtest/googletest/src/gtest_main.cc
[==========] Running 2 tests from 1 test suite.
[----------] Global test environment set-up.
[----------] 2 tests from tcp_nodelay
[ RUN      ] tcp_nodelay.incoming_ipv4
[       OK ] tcp_nodelay.incoming_ipv4 (2 ms)
[ RUN      ] tcp_nodelay.incoming_ipv6
[       OK ] tcp_nodelay.incoming_ipv6 (0 ms)
[----------] 2 tests from tcp_nodelay (3 ms total)

[----------] Global test environment tear-down
[==========] 2 tests from 1 test suite ran. (3 ms total)
[  PASSED  ] 2 tests.
```

</details>

<details>
<summary>regression-parent.log</summary>

```text
Running main() from monero/external/gtest/googletest/src/gtest_main.cc
[==========] Running 2 tests from 1 test suite.
[----------] Global test environment set-up.
[----------] 2 tests from tcp_nodelay
[ RUN      ] tcp_nodelay.incoming_ipv4
monero/tests/unit_tests/tcp_nodelay.cpp:106: Failure
Expected equality of these values:
  test.expected
    Which is: true
  actual.value()
    Which is: false
Google Test trace:
monero/tests/unit_tests/tcp_nodelay.cpp:71: RPC

[  FAILED  ] tcp_nodelay.incoming_ipv4 (5 ms)
[ RUN      ] tcp_nodelay.incoming_ipv6
monero/tests/unit_tests/tcp_nodelay.cpp:106: Failure
Expected equality of these values:
  test.expected
    Which is: true
  actual.value()
    Which is: false
Google Test trace:
monero/tests/unit_tests/tcp_nodelay.cpp:71: RPC

[  FAILED  ] tcp_nodelay.incoming_ipv6 (2 ms)
[----------] 2 tests from tcp_nodelay (7 ms total)

[----------] Global test environment tear-down
[==========] 2 tests from 1 test suite ran. (7 ms total)
[  PASSED  ] 0 tests.
[  FAILED  ] 2 tests, listed below:
[  FAILED  ] tcp_nodelay.incoming_ipv4
[  FAILED  ] tcp_nodelay.incoming_ipv6

 2 FAILED TESTS
```

</details>

<details>
<summary>socket-pr.log</summary>

```text
PASS IPv4 RPC plaintext
PASS IPv4 RPC TLS-post-handshake
PASS IPv4 NET plaintext
PASS IPv4 NET TLS-post-handshake
PASS IPv4 P2P plaintext
PASS IPv4 P2P TLS-post-handshake
PASS IPv6 RPC plaintext
PASS IPv6 RPC TLS-post-handshake
PASS IPv6 NET plaintext
PASS IPv6 NET TLS-post-handshake
PASS IPv6 P2P plaintext
PASS IPv6 P2P TLS-post-handshake
passed=12 failed=0 expected_policy=patched
```

</details>

<details>
<summary>socket-parent.log</summary>

```text
PASS IPv4 RPC plaintext
PASS IPv4 RPC TLS-post-handshake
PASS IPv4 NET plaintext
PASS IPv4 NET TLS-post-handshake
PASS IPv4 P2P plaintext
PASS IPv4 P2P TLS-post-handshake
PASS IPv6 RPC plaintext
PASS IPv6 RPC TLS-post-handshake
PASS IPv6 NET plaintext
PASS IPv6 NET TLS-post-handshake
PASS IPv6 P2P plaintext
PASS IPv6 P2P TLS-post-handshake
passed=12 failed=0 expected_policy=baseline
```

</details>
