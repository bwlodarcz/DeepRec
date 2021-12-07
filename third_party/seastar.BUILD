licenses(["notice"])  # Apache 2.0

package(default_visibility = ["//visibility:public"])

cc_library(
    name = "seastar",
    includes = [
        ".", "cached-fmt", "cached-c-ares",
        "cached-build/release/gen",
        "cached-build/release/c-ares"],
    srcs = glob(["**/*.cc"],
                exclude= [
                    "demo/**",
                    "demo-tf/**",
                    "demo-tf-test/**",
                    "cached-c-ares/c-ares/test/**",
                    "cached-fmt/test/**",
                    "cached-dpdk/**",
                    "tests/**",
                    "build/**",
                    "apps/**",
                    "rpc/**",
                    "core/prometheus.cc",
                    "net/proxy.cc",
                    "net/virtio.cc",
                    "net/dpdk.cc",
                    "net/ip.cc",
                    "net/ethernet.cc",
                    "net/arp.cc",
                    "net/native-stack.cc",
                    "net/ip_checksum.cc",
                    "net/udp.cc",
                    "net/tcp.cc",
                    "net/dhcp.cc",
                    "net/tls.cc",
                    "net/dns.cc",
                    "core/dpdk_rte.cc",
                    "cached-build/release/gen/proto/**",
                ]),
    copts = [
        "-std=gnu++1y",
        "-DNO_EXCEPTION_HACK",
        "-DDEFAULT_ALLOCATOR",
        "-DHAVE_NUMA",
    ],
    linkopts = [
        "-ldl",
        "-lm",
        "-lrt",
        "-lstdc++fs",
    ],
    deps = [
        "@boost//:asio",
        "@boost//:filesystem",
        "@boost//:fusion",
        "@boost//:lockfree",
        "@boost//:program_options",
        "@boost//:system",
        "@boost//:thread",
        "@boost//:variant",
        "@systemtap-sdt",
        "@xfs",
        "@lz4",
        "@libaio",
        "@sctp",
    ],
)

